# 抓包分析(Charles / mitmproxy / DevTools / tcpdump / Wireshark)

## 选工具:按场景决策

| 场景 | 首选 | 备注 |
|------|------|------|
| 浏览器页面的请求 | Chrome DevTools Network | 不需要装任何东西,先用这个 |
| iOS/Android App 的 HTTP(S) | Charles 或 mitmproxy(代理抓包) | 手机设代理 + 装证书 |
| 命令行/可脚本化抓包 | mitmproxy / mitmdump | 可写 Python 脚本改写请求 |
| 非 HTTP 流量、连接层问题 | tcpdump + Wireshark | TCP 握手失败、DNS、TLS 握手排查 |
| App 用了 SSL pinning | 见文末专节 | 代理抓包会失败,需绕过 |

## Charles(GUI,移动端抓包最常用)

1. **电脑端**:Proxy → Proxy Settings 确认端口(默认 8888);Help → SSL Proxying → Install Charles Root Certificate(电脑信任证书)
2. **手机设代理**:手机和电脑同一 Wi-Fi → Wi-Fi 设置 → 手动代理 → 填电脑 IP + 8888;Charles 弹窗 Allow
3. **手机装证书**:手机浏览器访问 `chls.pro/ssl` 下载证书
   - iOS:设置 → 通用 → VPN与设备管理 → 安装描述文件,然后**必须再去** 通用 → 关于本机 → 证书信任设置 → 打开完全信任(最容易漏的一步)
   - Android:设置里安装 CA 证书;**Android 7+ 的 App 默认不信任用户证书**,只能抓 debug 包(需要 App 的 network_security_config 允许用户证书),release 包抓不到是正常现象,不是配置错了
4. **开启 SSL Proxying**:Proxy → SSL Proxying Settings → Add,Host 填 `*`(或指定域名),Port 443。没做这步只能看到域名看不到内容
5. 常用功能:Breakpoints(拦截改包)、Rewrite(自动改写规则)、Map Local(用本地文件替换响应)、Throttle(弱网模拟,Proxy → Throttle Settings)

## mitmproxy(命令行 + 可编程)

```bash
brew install mitmproxy    # 或 pip install mitmproxy

mitmproxy                 # 交互式 TUI,默认监听 8080
mitmweb                   # 浏览器 UI(更接近 Charles 体验)
mitmdump -w flows.dump    # 无界面录制到文件
mitmdump -r flows.dump    # 回放查看
```

手机配置同 Charles:设代理到电脑 IP:8080,浏览器访问 `mitm.it` 装证书(iOS 记得开完全信任)。

脚本改写(mitmproxy 的独特优势,适合构造异常响应测试客户端容错):

```python
# fault_inject.py — 把指定接口的响应改成 500,测 App 的错误处理
# 运行: mitmdump -s fault_inject.py
from mitmproxy import http

def response(flow: http.HTTPFlow):
    if "/api/orders" in flow.request.pretty_url:
        flow.response = http.Response.make(
            500, b'{"code":50000,"message":"internal error"}',
            {"Content-Type": "application/json"})
```

类似手法可做:响应延迟(`time.sleep` 放 request 钩子)、字段篡改(测客户端对脏数据的容错)、请求重放。

## tcpdump + Wireshark(连接层问题)

HTTP 层看不出问题(连接根本没建立、TLS 握手失败、DNS 解析错)时用:

```bash
# 抓指定主机流量存文件(服务器上常用)
sudo tcpdump -i any host api.example.com -w capture.pcap
sudo tcpdump -i any port 443 and host 1.2.3.4 -w capture.pcap
# 拿回本地用 Wireshark 打开分析
```

Wireshark 常用过滤:`tcp.flags.reset == 1`(谁发的 RST)、`tls.handshake.type == 1`(ClientHello,看 SNI)、`dns`、`tcp.analysis.retransmission`(重传,网络质量差)。

iOS 真机还可以用 rvictl 免代理抓包(不需要装证书,但抓到的是加密流量,适合看连接层):

```bash
rvictl -s <设备UDID>          # 创建虚拟接口 rvi0
sudo tcpdump -i rvi0 -w ios.pcap
rvictl -x <设备UDID>
```

## SSL Pinning 与抓不到包的排查

**现象 → 原因对照:**

- 设了代理后 App 直接白屏/报网络错误,浏览器正常 → App 大概率做了 SSL pinning(校验固定证书),代理证书被拒
- Charles 里域名是乱码/只有 CONNECT 记录 → 没开 SSL Proxying,或证书没装/没信任
- Android 上浏览器能抓、App 抓不到 → Android 7+ 用户证书限制,换 debug 包
- 完全没有任何请求经过代理 → App 可能绕过系统代理(自实现 socket、gRPC 直连),用 tcpdump 确认流量走向

**Pinning 的处理(仅限测试自家 App)**:让开发在 debug 构建关掉 pinning 开关是最正路的方案;iOS 越狱/Frida(`objection android sslpinning disable` 类工具)属于逆向手段,只能用于自己有授权的 App。

## 抓包分析的方法论

拿到抓包记录后,按序核对:

1. 请求发出去了吗?(没有 → 客户端逻辑问题/被拦截)
2. URL、method、header(尤其 Authorization、Content-Type)、body 参数符合接口文档吗?
3. 响应状态码和 body 是什么?业务 code 是什么?
4. 耗时分布:DNS/连接/TTFB/下载,哪段慢?
5. 和"正常情况"的同一请求做 diff(一份好的抓包对比胜过盲目猜测)

结论落到:**客户端发错了 / 服务端答错了 / 网络层断了** 三者之一,并附上对应的请求记录作为证据。
