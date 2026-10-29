# dsh 服务端部署方案(NAS 虚拟机)

> 部署完成于 2026-10-28。版本快照:dsh `0.2.1-rc.2`(npm `latest` tag)、Node `v24.21.0`、Debian 13 (trixie) x86_64。
> 文中域名、token 均为占位符,替换为自己的实际值即可复用。
>
> **状态:全链路已端到端验证通过**,包含 WebSocket 事件流(nginx 访问日志实证 `GET /api/remote.mux upgrade=[websocket] -> 101`)与插件市场安装。
>
> 相关文档:[网络诊断记录](./network-diagnosis.md)——插件安装失败的排查实录与方法论。

## 1. 架构总览

```
浏览器(手机/平板/PC)
   │  HTTPS
   ▼
NAS 平台反向代理(子域名转发,平台账号认证,TLS 终止)
   │  HTTP/1.1(平台内;实测行为:把 Host 改写成上游地址,保留 Upgrade 头)
   ▼
Debian 虚拟机(loopback 上三跳)
   ├─ nginx(127.0.0.1:3082)── Host 改写回外部域名 + WS 透传
   └─ dsh --profile web(127.0.0.1:3083)
```

关键点:平台代理把 Host 改写成 `127.0.0.1:3082`,而 dsh 的信任栅栏要求 **Host 与浏览器 Origin 指向同一 authority**。因此 nginx 必须夹在中间把 Host 改写回外部域名——这一层不是可选项,漏掉会让所有带 Origin 的 API/WS 请求 403。

分层安全模型,由外到内:

| 层 | 机制 | 说明 |
|---|---|---|
| 1 | 平台账号认证 | NAS 平台的反向代理要求登录平台账号后才可访问子域名,**务必保持平台的"账号保护"开关开启** |
| 2 | 虚拟机边界 | dsh 跑在独立 Debian VM 里,独立内核,爆炸半径 = VM 文件系统 + 显式挂载进 VM 的存储;升级前打 VM 快照可整体回滚 |
| 3 | dsh token/cookie | 启动 token 兑换签名会话 cookie(30 天),每个 API 与 WebSocket 流都认证;Host/Origin 栅栏只放行 loopback、绑定地址与 `--trusted-host` 点名的 authority |
| 4 | preset 能力分级 | NAS 上的日常会话可默认 `minimal` preset(无文件读写编辑工具),写操作会话显式升级 |

**为什么是这个形态**(选型结论,详见仓库早期讨论):

- **VM 而非容器**:dsh 官方 SAFETY.md 的排序就是"虚拟机优先"——对一个会执行模型生成命令的 agent,独立内核是最硬的隔离;VM 快照同时兼任升级回滚机制
- **VM 内直装 npm 而非 Docker**:在通用 Debian VM 里,Node/进程守护/生命周期管理都是标准 Linux 问题;容器的隔离价值在已有 VM 边界时是冗余的
- **绑 loopback 而非虚拟机网卡 IP**:与 dsh 官方公网部署文档的推荐形态一致——服务只听 loopback,外部链路(域名、TLS、认证)完全归前置代理所有

### 1.1 关于"绕过 NAS NAT"的实测结论

VM 处于平台 NAT 网段(`100.64.0.0/24`),平台还额外提供一条覆盖网(实例内独立网卡上的全局 IPv6,由平台网络守护进程分配,**跨重启地址稳定**),它与宿主机/客户端组成 L3 可达网。这条覆盖网**确实能直连 dsh 且完整支持 WebSocket**(实测握手 101、ping 帧正常)。

但实测发现:该覆盖网的 OS TUN 通道在客户端侧会周期性抖动(观测到连续 5–74 秒不等的静默断连,约 15% 的连接超时),同一时刻平台自身的本地代理通道(实例内 `127.0.0.1:61090`)却 100% 正常。因此**不建议**把覆盖网直连当作日常访问入口;平台反向代理这条 HTTP 路径(本文方案)更稳定,且额外获得平台账号认证与 TLS。

需要临时直连调试时,覆盖网 IPv6 地址仍是有效手段——但只用于诊断,不用于日常使用。


## 2. 环境要求

- Debian 13(x86_64 实测,ARM 同理),内存 4GB 起(实测占用约 450MB)
- **Node.js ≥ 24.9.0**(dsh 的 `engines` 硬性要求,Debian 仓库版本不够新,见下节)
- 虚拟机可出网:npm 源 + 模型 API(`api.deepseek.com`)
- 一个专用非特权用户(本文以 `agent` 为例),有 sudo 权限

## 3. 安装

### 3.1 Node 24

```sh
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v   # 确认 >= 24.9.0
```

### 3.2 dsh(锁版本)

```sh
# 放行原生模块的安装脚本(dsh 的 shell 工具依赖 node-pty/koffi,
# npm 11 起默认拦截 postinstall,不放行会静默得到损坏的安装)
sudo npm config set allow-scripts=@deepseek-ai/dsh-subprocess-local,koffi,node-pty,@google/genai,protobufjs --location=user

# 锁定精确版本安装(npm latest tag)
sudo npm install -g @deepseek-ai/dsh@0.2.1-rc.2
dsh --version
```

国内网络注意:npmmirror 镜像对 rc 版本的元数据同步有延迟,遇到 `ETARGET` 时用 `--prefer-online` 或改用官方源。

### 3.3 预装旧版的遮蔽处理

若环境镜像里已预装 dsh(如沙盒/开发镜像),新版可能装到不同前缀被旧版 PATH 遮蔽。诊断与修复:

```sh
which -a dsh                  # 出现多个路径即存在遮蔽
sudo npm prefix -g            # npm 全局前缀(新版所在)
# 将旧路径的符号链接指向新安装:
sudo ln -sfn <npm-prefix>/lib/node_modules/@deepseek-ai/dsh/lib/bin.js <旧路径>/dsh
```

## 4. systemd 服务

`/etc/systemd/system/dsh.service`:

```ini
[Unit]
Description=DeepSeek Harness (dsh web)
After=network-online.target
Wants=network-online.target

[Service]
User=agent
Group=agent
WorkingDirectory=/home/agent/workspace/dsh
Environment=DSH_HOME=/home/agent/.dsh
Environment=TZ=Asia/Shanghai
EnvironmentFile=/etc/dsh.env
ExecStart=/usr/local/bin/dsh-start
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`/usr/local/bin/dsh-start`(0755,root 属主)——把启动逻辑从 unit 里拿出来,便于在不改 unit 的情况下换绑定地址/端口,同时等待网络地址就绪:

```sh
#!/bin/sh
set -eu

DSH_BIN=${DSH_BIN:-/usr/bin/dsh}
PORT=${PORT:-3083}
BIND=${DSH_BIND:-127.0.0.1}

i=0
while [ "$i" -lt 90 ]; do
    if ip -o addr show 2>/dev/null | grep -q " $BIND/"; then break; fi
    sleep 1
    i=$((i + 1))
done

if ! ip -o addr show 2>/dev/null | grep -q " $BIND/"; then
    echo "dsh-start: $BIND is not configured on this instance after 90s" >&2
    exit 1
fi

echo "dsh-start: binding $BIND:$PORT"
exec "$DSH_BIN" --profile web --no-open \
    --host "$BIND" --port "$PORT" \
    --trusted-host dsh.<你的NAS域名> \
    --public-url https://dsh.<你的NAS域名>/
```

`/etc/dsh.env`(`chmod 600`,root 属主):

```sh
DEEPSEEK_API_KEY=sk-xxx
```

启动:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now dsh
journalctl -u dsh -b | grep "dsh web:" | tail -1
# 输出形如:dsh web: https://dsh.<你的NAS域名>/?token=<token>
```

关键参数说明:

| 参数 | 作用 |
|---|---|
| `--host 127.0.0.1` | 只监听 loopback。平台代理把 Host 改写为上游地址,dsh 栅栏天然接受 loopback;同时不与虚拟机的 NAT/覆盖网接口共享暴露面 |
| `--port 3083` | dsh 自己听 3083;3082 让给 nginx(平台代理指向 3082) |
| `--trusted-host dsh.<域名>` | 栅栏白名单。浏览器实际可见的 authority 必须被点名(不带端口的条目匹配任意端口;可重复多次)。**即使 Host 被 nginx 改写成域名,这一条也必须存在**,否则栅栏拒绝 |
| `--public-url https://…` | 让启动日志打印的 URL 直接是外部可点击形式(token 已拼好),免去手动拼接 |
| `DSH_HOME` | 会话历史、配置、profile 补丁的持久化目录 |

> `--host` 不接受通配地址。实测 `--host ::` 直接报错退出:`binding every interface would expose remote code execution to the network`。这是 dsh 的有意设计,不要试图绕过。

## 5. nginx 前置层(Host 改写,必须)

平台转发器把 Host 头改写为上游地址(`127.0.0.1:3082`)。而 dsh 的请求信任栅栏要求:**Origin(浏览器可见 authority)必须与收到的 Host 指向同一 authority**——被改写后二者不一致,浏览器所有带 Origin 的 API/WS 请求全部 403(curl 不带 Origin 测不出来,务必用浏览器验证)。因此在 dsh 前加一层 nginx,把 Host 改写回外部域名。

端口分配:`nginx` 听 `3082`(平台代理指向它),`dsh` 听 `3083`。

`/etc/nginx/conf.d/dsh.conf`(安装:`apt-get install -y nginx`,并删除 `sites-enabled/default`):

```nginx
log_format wsdebug '$remote_addr "$request" upgrade=[$http_upgrade] done=$status';

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 127.0.0.1:3082;
    access_log /var/log/nginx/dsh-ws.log wsdebug;

    location / {
        proxy_pass http://127.0.0.1:3083;
        proxy_set_header Host dsh.<你的NAS域名>;   # 关键:改写回外部域名,与 Origin 对齐
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
    }
}
```

`log_format` 里的 `$http_upgrade` 是这套方案最好用的诊断手段:一条 `GET /api/remote.mux upgrade=[websocket] done=101` 就同时证明了 Upgrade 头存活、nginx 转发成功、WS 已建立。若显示 `upgrade=[-]`,说明 WS 压根没到这一层。

## 6. 平台反向代理接入要点(NAS 侧)

- **上游地址**:`127.0.0.1`,端口 `3082`(nginx;dsh 实际在 3083)
- **协议**:HTTP——TLS 由平台代理终止,后端明文是预期形态
- **账号保护**:保持开启(未登录平台账号不可访问)
- **WebSocket**:本平台的 HTTP 服务转发模式**保留** Upgrade 头,WS 可原生透传(实测 `/api/remote.mux` 握手 101)。不需要四层转发,也不需要额外开关
- **查询参数**:token 走 URL query,代理需透传(常规行为)

> 若上游填错(例如填了没在监听的地址),平台会返回自己的 502 页面,并直接在页面上写出它实际连接的上游,形如 `The configured upstream http://127.0.0.1:3082 is not accepting connections inside instance sandbox@…`。这是定位端口不匹配最快的线索。

## 7. 验证清单

部署完成后的逐项验证(`BASE=https://dsh.<你的NAS域名>`):

```sh
# 1. 未认证访问:401(栅栏放行、认证拦截)
curl -s -o /dev/null -w "%{http_code}\n" $BASE/

# 2. token 兑换:303 重定向 + Set-Cookie(HttpOnly, SameSite=Strict)
curl -s -D - -o /dev/null -c /tmp/c.txt "$BASE/?token=<token>" | grep -iE "HTTP|location|set-cookie"

# 3. 带 cookie 加载页面:200,标题为 DeepSeek Harness
curl -s -o /dev/null -w "%{http_code}\n" -b /tmp/c.txt $BASE/

# 4. 跨站 Origin 必须被拦:403(--http1.1 避免 HTTP/2 混淆)
curl -s -o /dev/null -w "%{http_code}\n" -b /tmp/c.txt \
     -H "Origin: https://evil.example" $BASE/api/rpc

# 5. WebSocket 握手:101(这是整套方案最容易失败的一项)
curl -s --http1.1 -i -N -b /tmp/c.txt \
     -H "Connection: Upgrade" -H "Upgrade: websocket" \
     -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
     -H "Origin: $BASE" $BASE/api/remote.mux | head -1
```

第 5 项必须用 `--http1.1`:HTTP/2 不支持经典的 `Upgrade` 机制,用默认协商会得到 403 而误判为故障。

服务端侧的一锤定音证据(不依赖客户端行为):

```sh
sudo grep 'upgrade=\[websocket\]' /var/log/nginx/dsh-ws.log | tail -3
# 期望:GET /api/remote.mux HTTP/1.1" upgrade=[websocket] done=101
```

功能验证:打开带 token 的 URL → 界面正常(标题 DeepSeek Harness)→ 配置模型 API key 后发一条消息 → **流式回复**。

## 8. 日常运维

```sh
# 会话模型(实测):打印的 token 每次进程重启轮换,但 cookie 签名密钥持久化在
# $DSH_HOME/.credentials.yaml,已兑换的 cookie 跨 dsh 重启/NAS 重启依然有效(30 天)。
# 因此 token URL 是"每个浏览器一次"的首次登录钥匙,不是每次重启都要取;
# 干净地址(无 token)的书签在持有 cookie 的浏览器上长期可用。
journalctl -u dsh -b | grep "dsh web:" | tail -1   # 仅在新浏览器首次登录时需要

# 服务状态/重启
systemctl status dsh && sudo systemctl restart dsh

# 升级(建议先打 VM 快照;dev preview 版本间可能有破坏性变更)
sudo npm install -g @deepseek-ai/dsh@<新版本>
sudo systemctl restart dsh

# 会话数据备份:DSH_HOME 目录(如 /home/agent/.dsh)纳入 VM 快照/备份策略
```

API key 变更有两种方式:网页 **设置 → 模型** 直接改(不重启、token 不变),或编辑 `/etc/dsh.env` 后重启(会轮换 token)。

## 9. 插件市场(实测踩坑,重要)

> 完整的排查过程、误判分析与可复用的诊断方法见 [network-diagnosis.md](./network-diagnosis.md)。

dsh 内置的插件市场(dshmarket)有两套独立的网络路径,理解这个区别能省掉大量排错时间:

| 路径 | 触发方式 | 网络行为 |
|---|---|---|
| CLI 安装 | `dsh plugin --profile web add <pkg>` | 走 pnpm/npm,复用 npm registry,自带强重试 |
| Web UI 市场 | 侧边栏 **Plugin Market** | 走 dshmarket 的 catalog 抓取,单独一套源与重试策略 |

所以**「CLI 能装、网页报错」是完全可能且正常的**——它们不共享网络路径,不能用其中一个的成功反证另一个没问题。

### 9.1 症状:插件市场一直 `fetch failed`

典型报错:

```
route error: fetch failed (42s, 4 attempts)
· earlier: npm:https://<镜像>/npm/dsh-plugin-catalog — fetch failed
· url:https://awesome-dsh-plugin.com/plugins.json — fetch failed
```

或更具体的:

```
no new data for 15s while downloading the catalog (received 6797765 bytes) (72s, 4 attempts)
```

后者说明目录已经开始下载(6.7MB),但中途 15 秒无新数据,dshmarket 的停滞检测主动放弃。

### 9.2 根因:默认 region 指向的国内镜像源不可用

市场配置在 `$DSH_HOME/profiles/web/.dsh-market/state.json`,默认 `"region": "china"`。而 `china` 的 catalog 顺序是**内网镜像优先**:

```js
china: {
  catalog: [
    { kind: 'npm', registry: NPM_CHINA, pkg: CATALOG_PACKAGE },  // 排第一
    { kind: 'url', url: CATALOG_OFFICIAL },                      // 兜底
  ],
}
global: {
  catalog: [{ kind: 'url', url: CATALOG_OFFICIAL }],              // 只有官方源
}
```

实测该镜像源(一个仅 114KB 的 JSON)**同一文件三次返回三种大小**——297KB / 297KB / 229KB,耗时从 3.8s 恶化到 60s 超时。内容被截断且速度不可控。每次抓取都先耗 45 秒在这个坏源上,把 4 次重试预算全部烧光,于是必然失败。

对照:官方源同样的 9.5MB 目录,正常时只需 3–9 秒。

### 9.3 修复

```sh
# 1. 备份并切换到 global region(跳过不可用的国内镜像)
cp ~/.dsh/profiles/web/.dsh-market/state.json /tmp/state.json.bak
python3 - <<'PY'
import json, os
p = os.path.expanduser("~/.dsh/profiles/web/.dsh-market/state.json")
d = json.load(open(p))
d["region"] = "global"
d["regionAuto"] = False          # 关掉自动探测,防止它又切回坏源
json.dump(d, open(p, "w"), ensure_ascii=False)
PY

# 2. 重启使配置生效(会轮换 token,已有 cookie 不受影响)
sudo systemctl restart dsh
```

生效验证:打开 **Plugin Market**,应能看到完整插件列表(实测 7.4k 个),并显示版本兼容性过滤信息,例如
`Hid 222 plugins incompatible with DSH 0.2.1-rc.2`。

### 9.4 判断是否已修复

```sh
# 市场日志:出现 install / update 的 info 记录即正常,不再有 catalog fetch failed
tail -n 10 $DSH_HOME/profiles/web/.dsh-market/log.ndjson

# 官方源直测(应该几秒内拿到完整大小,而不是 0 字节)
curl -s -o /dev/null -w "%{size_download}B %{time_total}s\n" \
     https://awesome-dsh-plugin.com/plugins.json
```

装完插件后 UI 会出现 `1 change(s) pending restart`,重启 dsh 生效。

## 10. 模型 API 连接稳定性(长回复中断)

### 10.1 症状

运行任务时偶发:

```
本轮运行失败 DeepSeek: connection lost mid-reply
(read tcp 100.64.0.10:58898->198.18.0.13:443: read: connection reset by peer)
```

`198.18.0.13` 是该域名在 fake-ip 段里的映射地址(见第 10 章踩坑 14),实际目标是模型 API 的 443 端口。请求链路是:

```
dsh → 本地模型网关(127.0.0.1:3425) → api.deepseek.com
```

### 10.2 实测:不是地址问题,是链路间歇故障

用真实 API 做长流(约 200KB / 单次 5 秒)交替对比两条地址路径,各 8 轮:

| 路径 | 成功 | 失败形态 |
|---|---|---|
| fake-ip(`198.18.0.x`) | 6/8 | 1 次建连超时、1 次传输中途停滞 |
| 真实 IP(`--resolve` 指定) | 7/8 | 1 次传输中途停滞 |

**两条路径都失败**,说明问题不在 DNS 或 fake-ip,而在上游链路的时段性中断。失败有两种形态:

1. **建连超时**——SYN 丢失,请求根本没发出去
2. **传输中途停滞**——已收到部分字节(实测 5KB / 117KB 不等),之后长时间无数据,直到客户端超时

第二种正是用户看到的"connection lost mid-reply":回复进行到一半,连接被静默掐断。

### 10.3 缓解:提高重试韧性

链路问题无法在 VM 内根治(故障在上游代理),因此把重点放在"单次中断不至于让整轮任务失败"。

dsh 的 provider 配置支持 `retryPolicy`,默认 `maxRetries: 5`。实测单次成功率约 75–87%,按 10 次重试计算累积成功率可达 99.9999%,足以覆盖当前故障率。

编辑 `$DSH_HOME/profiles/web/cordis.patch.yml`,在该 provider 段内加:

```yaml
        retryPolicy:
          mode: normal
          maxRetries: 10
          backoff:
            initialDelayMs: 1000
            maxDelayMs: 10000
            jitterRatio: 0.2
```

`retryableCodes` 用默认值即可(`EMPTY_RESPONSE` / `RATE_LIMIT` / `SERVER` / `TIMEOUT` / `TRANSPORT`),其中 `TRANSPORT` 与 `TIMEOUT` 正好覆盖连接重置和停滞。

配套调快 TCP 僵死连接的探测(默认 2 小时才探测,等于不起作用):

```sh
sudo tee /etc/sysctl.d/90-dsh-keepalive.conf >/dev/null <<'EOF'
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_keepalive_probes = 4
EOF
sudo sysctl -p /etc/sysctl.d/90-dsh-keepalive.conf
```

改完重启:

```sh
sudo systemctl restart dsh
```

### 10.4 验证配置真的生效

不要只看服务是不是 active——配置写错 schema 会被拒绝但服务仍可能起来。用合成的运行时配置确认:

```sh
dsh --profile web --dump-config | grep -A6 "maxRetries: 10"
# 期望看到 retryPolicy / maxRetries: 10 / backoff 三项
```

端到端验证:在 UI 发一条消息,回复能正常流式输出并显示完成。实测一次 11 秒完成、90 tok/s。

### 10.5 残余风险

如果某段时间上游链路连续故障,10 次重试仍可能全部落在坏窗口内,任务会失败——这时**直接重试一次**通常就恢复了。若发现失败率显著上升,再考虑在代理侧为实例流量做规则隔离或更换出口。

## 11. 踩坑记录(实测)

1. **npm 11 拦截原生模块安装脚本**:`node-pty`/`koffi` 的 postinstall 被默认拦截,安装"成功"但 shell 工具损坏,必须先配 `allow-scripts`(见 3.2)
2. **npmmirror 对 rc 版本同步延迟**:报 `ETARGET` 时加 `--prefer-online` 或换官方源
3. **`sudo -S` 与 heredoc 抢 stdin**:`echo 密码 | sudo -S tee file <<EOF` 这类写法里,heredoc 会覆盖管道,sudo 把文件内容当密码。正确做法:先以普通用户写到 `/tmp`,再 `sudo cp`
4. **代理 Host 改写 ≠ 无害**:curl(不带 Origin)测试全通过会给人"已通"的错觉;栅栏实际要求 Origin 与 Host 同一 authority,浏览器请求因此全 403。解决:nginx 前置层把 Host 改写回外部域名(见第 5 节)
5. **预装二进制遮蔽**:见 3.3,升级后务必 `dsh --version` 确认
6. **token 打印值重启即换,但 cookie 不受影响**:dsh 的 cookie 签名密钥持久化(`$DSH_HOME/.credentials.yaml`),实测重启服务后旧 cookie 依然通过认证。token URL 的真实作用是"每个浏览器一次的首次登录钥匙"(cookie 有效期 30 天);不要误判为"每次重启都要重新取 token"
7. **端口不匹配的 502 自解释**:平台代理连不上上游时返回自己的 502 页面,且直接在页面上打印它实际连接的上游地址。改过 dsh 绑定地址/端口后,先看这个页面比翻日志快得多
8. **配置文件落在安装前的目录会静默失败**:`cp` 到尚不存在的目录且 stderr 被 `2>/dev/null` 吞掉,服务会跑默认配置。教训:部署配置后必须 `nginx -t` + 实际端口探测验证,不能只看服务 active
9. **测 WS 必须显式 `--http1.1`**:HTTP/2 不支持 `Upgrade` 机制,用 curl 默认协商去测 WS 会稳定得到 403,极易误判成"被 fence 拦了"或"被平台剥了 Upgrade 头"。真实浏览器在 `wss://` 上走 HTTP/1.1 Upgrade 或 h2 extended CONNECT,不受此影响
10. **覆盖网 TUN 在客户端侧会周期性抖动**:实例内的全局 IPv6(平台覆盖网)在服务端始终健康、WS 也完整可用,但客户端侧 OS TUN 观察到 5–74 秒的静默断连,同一时刻平台本地代理通道 100% 正常。结论:覆盖网适合临时诊断,不适合当日常入口(详见 1.1)
11. **NAT 地址在切网络模式后会变**:如果曾把实例切到 Macvlan 等模式,实例原有的 NAT 地址会消失,依赖它的转发会直接 502。回切 NAT 后地址恢复。dsh 的绑定地址用 `DSH_BIND` 环境变量控制,改一处即可
12. **插件市场的 region 默认值会让国内用户踩坑**:`region: china` 把内网镜像排在 catalog 第一个,而该镜像实测返回截断内容且速度不可控,导致重试预算被耗尽。修复见第 9 节
13. **CLI 安装与 Web UI 安装走不同网络路径**:`dsh plugin add` 走 pnpm/npm,Web UI 走 dshmarket catalog。前者成功不能证明后者没问题,反之亦然。排错时必须分开验证
14. **DNS 返回 `198.18.0.x` 是 DNS 层的 fake-ip 注入**:这是上游 Clash TUN 的 fake-ip 段(平台 Clash 声明 `198.18.0.1/30`,VM 却拿到 .11/.25/.39,已越界)。实测它与真实 IP 的 TCP 可达性无差别(各 8/8),**不是插件安装失败的主因**;真正的瓶颈是长连接传输的时段性停滞。看到这个地址不必惊慌,但值得用 `--resolve` 强制真实 IP 做对照
15. **`ping` 不可用作这台机器的连通性判据**:平台对 ICMP 有限速/过滤,实测网关丢包率可达 40%、公网 100%,但同一时刻 TCP 443 全部可达。用 TCP 而非 ICMP 判断网络健康
16. **长回复中断是链路间歇故障,不是 DNS 或 fake-ip 引起**:用真实 IP 与 fake-ip 各跑 8 轮长流,两者都有失败(7/8 与 6/8)。修复方向是提高重试而非换地址,详见第 10 章
17. **改完配置要看合成后的运行时配置**:`systemctl is-active` 只说明进程活着。用 `dsh --profile web --dump-config` 确认改动真的进了配置树,否则可能 schema 报错被静默忽略

## 12. 后续规划

- [x] **WebSocket 透传**——已解决。平台 HTTP 服务转发保留 Upgrade 头,nginx 前置层负责 Host 对齐与 WS 升级,实测握手 101
- [x] **插件市场抓取失败**——已解决。根因是默认 region 指向的国内镜像源不可用,切到 global 后正常(见第 9 节)
- [x] **插件可安装**——已验证 `dsh-better-sidebar`、`dsh-whale-widget`、`dsh-context`、`dsh-remote` 均可用,Web UI 与 CLI 两条路径都能装
- [x] **长回复中断**——已缓解。链路间歇故障无法在 VM 内根治,改为 provider 级 `maxRetries: 10` + TCP keepalive 提速(见第 10 章),端到端验证通过
- [ ] 模型 API key 配置与消费上限设置(当前 `/etc/dsh.env` 仍是占位符,也可在网页 **设置 → 模型** 里配)
- [ ] preset 能力分级落地(NAS 默认 `minimal`)
- [ ] 可选加固:VM 内 nftables 阻断到 RFC1918 内网的出栈(防提示注入后的横向移动;注意与 NAS 存储挂载通道的兼容性——virtiofs/9p 走 hypervisor 通道不受影响,SMB/NFS 走网络需为 NAS IP 加白)。**注意**:在 NAT 模式下 dsh 绑 loopback、外部入口只有平台代理,出栈加固的边际收益有限,优先级低于前两项
- [ ] 个人插件开发:本地 `dev:web` 迭代 → 自建 https 插件索引分发;首个选题:给 schedule 服务补推送投递(上游的到期提醒只落成会话内 follow-up 消息,无常驻推送)
- [ ] 插件对锁定版本的 CI(dsh 处于 dev preview,`engines.dsh` 兼容检查只防声明层)
- [ ] 清理:此前排查阶段试过的 LPK 反代应用(应用商店里的 `cloud.wuxi.dshrelay`)与本方案无关且已失效,可从应用列表卸载
- [ ] 待观察:该 VM 的出站长连接存在时段性停滞(实测大文件下载 60–90% 成功率,受上游代理链路影响)。目前靠 dshmarket 的 4 次重试覆盖;若将来更频繁,考虑在 Clash 侧为实例流量做规则隔离
