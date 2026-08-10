# 日志分析(移动端 / 前端 / 后端 / 通用技巧)

## 通用方法论

日志排查的目标是**用时间线还原事发经过**。不管什么平台,套路一致:

1. **锚定时间点**:先确定问题发生的准确时间(用户操作时间、报错弹窗时间、监控告警时间),把日志范围缩到前后几分钟
2. **找第一个异常**:错误经常连锁,日志里最后的报错往往是结果不是原因,向上找**第一条**异常记录
3. **跟踪一次请求/操作的完整链路**:用 request id / trace id / 用户 id 串起跨端日志;客户端时间和服务端时间可能有偏差,对表
4. **区分"错误"和"噪音"**:很多 App/系统日志常年打 error 但无害,拿不准就对比正常时段是否也有同样的报错

## Android:adb logcat

```bash
adb logcat -c                          # 清空旧日志(复现前先清,减少噪音)
adb logcat                             # 全量实时输出
adb logcat *:E                         # 只看 Error 及以上(V<D<I<W<E<F)
adb logcat -s MyAppTag                 # 只看指定 tag
adb logcat --pid=$(adb shell pidof -s com.example.myapp)   # 只看本 App 进程
adb logcat --buffer=crash              # 崩溃专用缓冲区
adb logcat -v time > app.log           # 带时间戳存文件
adb logcat -d -t "08-10 14:30:00.000"  # 只导出某时间点之后的
adb bugreport bugreport.zip            # 完整诊断包(ANR/电量/系统状态都在里面)
```

崩溃看什么:`FATAL EXCEPTION` 开头的段落,第一行是异常类型和消息,`Caused by:` 链的最后一个才是根因,栈里找自己包名的第一行。ANR 看 `data/anr/traces.txt`(在 bugreport 里),主线程 `main` 的栈显示它卡在哪。

## iOS:模拟器与真机日志

```bash
# 模拟器:实时流(predicate 过滤,否则噪音极大)
xcrun simctl spawn booted log stream --level=debug \
  --predicate 'process == "MyApp"'
xcrun simctl spawn booted log stream \
  --predicate 'subsystem == "com.example.myapp"'    # 用 os_log 的 subsystem 过滤

# 查历史日志
xcrun simctl spawn booted log show --last 10m --predicate 'process == "MyApp"'

# 真机
# 方式1:Xcode → Window → Devices and Simulators → Open Console
# 方式2:Console.app 左侧选设备,搜索框过滤 process
# 方式3:命令行(需 libimobiledevice)
brew install libimobiledevice
idevicesyslog | grep MyApp
```

崩溃报告(.ips 文件):模拟器在 `~/Library/Logs/DiagnosticReports/`,真机在 Xcode Devices 窗口 View Device Logs。看崩溃报告重点:

- `Exception Type`:`EXC_BAD_ACCESS` 野指针/过释放、`EXC_CRASH (SIGABRT)` 常见于未捕获异常/断言、`EXC_BREAKPOINT` Swift 强解包 nil 或数组越界
- Termination Reason 里的 watchdog 超时(`0x8badf00d`)是主线程卡死被系统杀
- release 包的栈是地址,需要 dSYM 符号化:`atos -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp -l <load address> <crash address>`

Flutter App:`flutter logs` 直接聚合设备日志;Dart 层异常有完整栈,原生层崩溃仍按上面 iOS/Android 方式查。

## 前端(浏览器)日志

- Console 报错点击右侧文件链接跳到源码;压缩代码需要 sourcemap,没有的话用 `{}` pretty-print 后靠列号定位
- 线上问题:让用户/客服提供 Console 截图 + Network 截图(含失败请求的红色行);更系统的做法是接 Sentry 类前端监控,拿到带 sourcemap 还原的错误栈
- `window.onerror` 抓不到的:Promise 未处理拒绝(要监听 `unhandledrejection`)、跨域脚本错误(只显示 Script error,需要脚本加 crossorigin + CORS 头)
- 复现用户环境:UA、屏幕尺寸、登录态、LocalStorage 内容都可能影响行为,DevTools → Application 面板核对

## 后端日志(NestJS / 通用)

NestJS 默认 Logger 输出到 stdout;生产建议 JSON 结构化日志(pino/winston),每条带 request id。排查接口问题:

```bash
# 文本日志的经典组合拳
grep "ERROR" app.log | head -50
grep -n "req-abc123" app.log                  # 按 request id 串链路
grep "ERROR" app.log | awk '{print $5}' | sort | uniq -c | sort -rn   # 错误种类统计
awk '$1 >= "2026-08-10T14:00" && $1 <= "2026-08-10T14:10"' app.log    # 时间段切片(ISO 时间戳可字符串比较)
tail -f app.log | grep --line-buffered "orders"                       # 实时观察复现
zgrep "OutOfMemory" app.log.*.gz              # 查已轮转的压缩日志
```

Docker/K8s 环境:`docker logs -f --since 10m <container>`、`kubectl logs -f deploy/api --since=10m`、`kubectl logs <pod> --previous`(看重启前的日志,查 OOMKilled 必用)。

## 把日志证据写进报告

引用日志时:保留时间戳和上下文(报错行前后各 3-5 行),标注日志来源(哪个设备/服务/文件),敏感信息(token、手机号)打码。结论句式:"14:32:05 客户端发起 /api/pay 请求(见客户端日志 L120),14:32:08 服务端抛出 TimeoutError(见服务日志 L88),说明……"——每个论断都指向一条具体日志。
