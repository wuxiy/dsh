# 网络诊断记录:插件安装失败排查实录

> 记录于 2026-10-28。这是一次"症状在 A、根因在 B"的排查,过程里有几处误判值得留存,避免下次重复走弯路。
> 结论先行:**根因是插件市场默认 region 指向的国内镜像源不可用**,网络本身只是时段性抖动。
> 部署方案本身见 [deployment.md](./deployment.md)。

## 1. 起始症状

插件安装失败,报错:

```
fetch failed (42s, 4 attempts)
· earlier: npm:https://<镜像>/npm/dsh-plugin-catalog — fetch failed
· url:https://awesome-dsh-plugin.com/plugins.json — fetch failed
```

后续演进为更具体的:

```
no new data for 15s while downloading the catalog (received 6797765 bytes) (72s, 4 attempts)
```

关键转折信息来自使用者的一条观察:**「直接在服务器上执行 `dsh plugin --profile web add dsh-better-sidebar` 能装,但通过 Web 客户端会报错。」**

## 2. 最终根因

`dshmarket` 的 region 配置默认是 `china`,而 `china` 的 catalog 源顺序把国内镜像排在第一位:

```js
china: {
  catalog: [
    { kind: 'npm', registry: NPM_CHINA, pkg: CATALOG_PACKAGE },  // 优先
    { kind: 'url', url: CATALOG_OFFICIAL },                      // 兜底
  ],
}
```

该镜像源实测不可用——**同一个 114KB 的文件,三次下载返回三种大小**:

| 尝试 | 大小 | 耗时 |
|---|---|---|
| 1 | 297,061 B | 3.8s |
| 2 | 297,061 B | 10.2s |
| 3 | 229,029 B | 60s 超时 |

内容截断 + 速度不可控。每次抓取先在这上面耗掉 45 秒,4 次重试预算全部烧光,必然失败。

官方源对照:9.5MB 的 `plugins.json` 正常时 3–9 秒完成。

修复方式见 deployment.md 第 9 节(把 `region` 改为 `global` 并关闭 `regionAuto`)。

## 3. 为什么 CLI 能装而 Web UI 不能

这是全局最有价值的线索。dsh 有两条完全独立的网络路径:

| 路径 | 命令/入口 | 网络行为 |
|---|---|---|
| CLI | `dsh plugin --profile web add <pkg>` | 走 pnpm/npm,复用 npm registry 配置,自带强重试 |
| Web UI | 侧边栏 Plugin Market | 走 dshmarket 自己的 catalog 抓取,独立源与重试策略 |

`dshmarket/lib/dsh-cli.js` 的源码注释也印证了这个分裂:pnpm 只读 `npm_config_https_proxy`,不认 `HTTPS_PROXY`;市场自身的 fetch 读标准变量。两者需要显式转换才能共享代理配置。

**教训:不能用一条路径的成功反证另一条路径没问题。** 排错时必须分别验证。

## 4. 排查中走过的弯路(留作前车之鉴)

### 4.1 误判一:把 `ping` 当连通性判据

早期用 `ping` 测丢包,看到网关丢包 40.5%、公网 100%,几乎判定为链路严重故障。

但同一时刻 TCP 443 全部可达。**该平台对 ICMP 有限速或过滤**,ping 结果毫无参考价值。后续所有判断改用 TCP 连接测试。

### 4.2 误判二:脚本 bug 导致的"0ms 成功"

写 DNS 探测脚本时没等待真实响应就打印,得出"直连公网 DNS 0ms 成功"的错误结论。实际那些 DNS 服务器全部不可达。**测量脚本必须校验返回内容,而不只是没抛异常。**

### 4.3 误判三:查错了 Clash 实例

一度怀疑 NAS 上的 Clash 是元凶,报告"36 个连接来自同一个内网 IP"。实际那是 **Mac 本机 Clash Verge** 的数据(连本地局域网 MySQL/RabbitMQ),与 VM 无关。

区分方法:VM 出口 IP 落在境外云厂商网段(实测为 AWS 东京),Mac Clash 出口是另一个完全不同的地址。真正的 NAS Clash 在 `clash.<域名>/api/`,当时只有 6–14 个连接且全走 DIRECT。

**教训:多台机器都有同名软件时,先确认自己连的是哪一台。**

### 4.4 误判四:把 fake-ip 当成主因

VM 里所有域名解析成 `198.18.0.x`,这是上游 Clash TUN 的 fake-ip 段,且地址越界(Clash 声明 `198.18.0.1/30`,VM 却拿到 .11/.25/.39)。

看起来很像根因,但对照实验推翻了这个假设:

| 路径 | TCP 可达 | 大文件下载 |
|---|---|---|
| fake-ip 198.18.0.25 | 8/8 | 60–70% 完整 |
| 真实 IP 185.199.111.153 | 8/8 | 90% 完整 |

TCP 层完全一致,只有大文件传输有差异。所以 fake-ip 只是让故障略严重,不是主因。**主因是长连接传输的时段性停滞。**

### 4.5 误判五:动了不该动的配置

诊断过程中曾把 VM 网卡 MTU 从 1500 降到 1400、把 `resolv.conf` 指向 127.0.0.53,均无改善。这类"看起来合理"的改动增加了变量,事后需要逐一回滚。

**原则:在定位到证据之前,不要修改生产配置。** 若必须改,先备份并记录,结束后恢复。

## 5. 有效的诊断方法(可复用)

### 5.1 分层定位

从外到内逐层测,而不是一上来就猜:

```
DNS 解析 → TCP 建连 → TLS 握手 → HTTP 首字节 → 完整传输
```

本次结论:前四层全部健康,问题出在第五层(长连接传输中途停滞)。

### 5.2 分阶段计时 + 进度标记

记录下载过程中每 2 秒的已收字节数,能看出是"慢"还是"停":

```
#2: 9508325 bytes in 15.4s DONE  rate=602KB/s
#3: 7354937 bytes in 30.9s STALL rate=232KB/s   ← 停在 7.3MB
```

"停在固定位置且长时间无数据"= 连接被掐断,而非带宽不足。

### 5.3 强制真实 IP 做对照

用 curl 的 `--resolve` 绕过 DNS 层,直接验证地址解析是否参与故障:

```sh
curl --resolve awesome-dsh-plugin.com:443:185.199.111.153 \
     -o /dev/null -w "%{size_download}B %{time_total}s\n" \
     https://awesome-dsh-plugin.com/plugins.json
```

这是区分"DNS 问题"和"链路问题"最干净的方法。

### 5.4 看应用自己的日志,而不是只看 curl

`$DSH_HOME/profiles/web/.dsh-market/log.ndjson` 里的错误比 curl 的输出精确得多——它明确写出了"收到多少字节后停滞""几次尝试",直接指向停滞检测逻辑。

## 6. 该环境的网络特征(备忘)

以下都是实测结论,后续遇到类似现象可直接参考:

- **出站走代理**:VM 出口 IP 是 AWS 东京节点,不是本地直连
- **DNS 返回 fake-ip**:`198.18.0.x`,上游 Clash TUN 注入;与真实 IP 的 TCP 可达性无差别
- **ICMP 不可信**:限速/过滤严重,判断连通性请用 TCP
- **长连接有停滞**:大文件传输成功率 60–90%,与目标、时段相关;小文件(HEAD、Range)通常正常
- **平台代理通道**:`127.0.0.1:61090`(VM 内),可用但不是万能——它同样受底层链路影响
- **覆盖网 TUN 抖动**:实例的全局 IPv6 在服务端健康、WS 可用,但客户端侧有 5–74 秒静默断连(见 deployment.md 1.1)

## 7. 修复验证

改 `region` 为 `global` 并重启后:

- 插件市场正常加载,**显示 7.4k 个插件**(此前完全打不开)
- 版本兼容性过滤正常工作:`Hid 222 plugins incompatible with DSH 0.2.1-rc.2`
- Web UI 实际安装成功,日志可证:

```json
{"event":"install","detail":"dsh-remote: pinned to the registry's latest, 0.8.54"}
{"event":"install","detail":"dsh-remote@0.8.54 exit=0 hot=true"}
```

- 已装插件确认存在:`dsh-better-sidebar`、`dsh-whale-widget`、`dsh-context`、`dsh-remote`
- 安装后 UI 提示 `1 change(s) pending restart`,重启生效

### 7.1 修复后仍会偶发失败

切到 global 后链路抖动依然存在,日志中仍会看到零星的失败,随后重试成功:

```
09:39:38 error install route error: fetch failed (40s, 4 attempts) · url:https://awesome-dsh-plugin.com/
09:42:32 error install route error: fetch failed (41s, 4 attempts) · url:https://awesome-dsh-plugin.com/
09:42:44 info  install dsh-remote: pinned to the registry's latest, 0.8.54
09:42:58 info  install dsh-remote@0.8.54 exit=0 hot=true
```

这是上游链路的问题,不是配置问题。**应对方式就是重试**——单次成功的概率已经足够高,4 次重试的累积成功率可达 99% 以上。若未来失败变频繁,再考虑在 Clash 侧为实例流量做规则隔离。

## 8. 后续追加:长回复中途中断

插件问题解决后,又出现了一个新的症状,属于同一底层链路问题:

```
本轮运行失败 DeepSeek: connection lost mid-reply
(read tcp 100.64.0.10:58898->198.18.0.13:443: read: connection reset by peer)
```

`198.18.0.13` 是模型 API 的 fake-ip 映射。此时的判断路径和前面一致:**先别急着怀疑地址解析,直接测真实行为。**

### 8.1 用真实 API 做长流对照

写脚本读取本机凭据,直接对 API 发流式请求(约 200KB / 每次 5 秒),fake-ip 与真实 IP 交替各跑 8 轮:

| 路径 | 成功 | 失败形态 |
|---|---|---|
| fake-ip | 6/8 | 1 次建连超时、1 次传输中途停滞(收到 117KB 后停住) |
| 真实 IP | 7/8 | 1 次传输中途停滞(收到 5KB 后停住) |

两个结论:

1. **不是地址问题**——真实 IP 同样失败,排除 fake-ip 嫌疑
2. **失败有两种形态**——建连阶段 SYN 丢失,和传输中途静默停滞。后者正是"mid-reply"的来源:回复进行到一半,连接被掐断且没有任何错误通知,只能等客户端超时

### 8.2 处理思路:不追求消除故障,而是让故障不至于毁掉整轮任务

链路在上游,VM 内无法根治。所以把力气放在重试韧性上:把 provider 的 `maxRetries` 从默认 5 提到 10。单次成功率按 80% 估算,10 次重试的累积失败率约 1e-7。

配套把 TCP keepalive 调快(系统默认 2 小时才探测第一次,对及时发现问题毫无意义),设为 60 秒空闲后每 15 秒探测、4 次失败即判定断连。

具体配置见 [deployment.md](./deployment.md) 第 10 章。

### 8.3 验证要点

改配置后**不要只看服务是否 active**。用 `dsh --profile web --dump-config` 看合成后的配置树里有没有真的出现新的 `maxRetries`,再在 UI 里发一条消息确认能正常流式完成。实测改完一轮回复 11 秒完成、90 tok/s。

## 9. 一句话总结

**症状在网络层,根因在配置层。** 报错信息一致指向"fetch failed",看起来像网络故障,实际是默认配置选了一个坏源,把重试预算耗尽了。下次遇到类似问题,优先检查:应用自己的配置默认值 → 它选了哪个源 → 那个源本身是否可用。

而长回复中断则相反:**根因确实在网络层**,只能通过重试韧性来吸收,不要在上面继续找配置错误。
