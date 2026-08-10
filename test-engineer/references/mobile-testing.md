# 移动端测试(iOS / Android / Flutter / React Native / UniApp)

## iOS 模拟器操作(xcrun simctl)

simctl 是 iOS 模拟器的命令行入口,常用操作:

```bash
# 列出所有模拟器(找 UDID 和状态)
xcrun simctl list devices
xcrun simctl list devices available   # 只看可用的

# 启动 / 关闭
xcrun simctl boot "iPhone 16 Pro"
open -a Simulator                      # 打开模拟器窗口
xcrun simctl shutdown booted           # booted 代表当前启动的设备

# 安装 / 卸载 / 启动 App
xcrun simctl install booted ./MyApp.app
xcrun simctl uninstall booted com.example.myapp
xcrun simctl launch booted com.example.myapp
xcrun simctl terminate booted com.example.myapp

# 截图 / 录屏(取证据必备)
xcrun simctl io booted screenshot screen.png
xcrun simctl io booted recordVideo demo.mp4    # Ctrl+C 结束录制

# 常用模拟场景
xcrun simctl openurl booted "myapp://deeplink/path"   # 测 deep link
xcrun simctl push booted com.example.myapp payload.json  # 测推送(payload 为 APNs JSON)
xcrun simctl status_bar booted override --time "9:41" --batteryLevel 100  # 截图前美化状态栏
xcrun simctl privacy booted grant photos com.example.myapp  # 授权/拒绝权限(grant/revoke/reset)
xcrun simctl location booted set 31.2304,121.4737     # 模拟定位(上海)

# 重置模拟器(清除所有数据,复现"首次安装"场景)
xcrun simctl erase booted
```

模拟器 App 沙盒路径(直接看 App 本地数据):

```bash
xcrun simctl get_app_container booted com.example.myapp data
```

### 模拟器测不了的东西(必须真机)

推送的真实到达、相机、蓝牙、NFC、真实性能(模拟器用的是 Mac 的 CPU)、后台挂起行为、Metal 渲染差异。涉及这些时提醒用户用真机验证。

## Android 模拟器与 adb

```bash
# 设备管理
adb devices                            # 列出连接的设备/模拟器
emulator -list-avds                    # 列出 AVD
emulator -avd Pixel_8_API_35 &         # 启动模拟器
adb -s emulator-5554 <cmd>             # 多设备时用 -s 指定

# 安装 / 卸载
adb install -r app-debug.apk           # -r 覆盖安装保留数据
adb uninstall com.example.myapp
adb shell pm clear com.example.myapp   # 清除 App 数据(等于重装)

# 启动 / 停止
adb shell am start -n com.example.myapp/.MainActivity
adb shell am force-stop com.example.myapp
adb shell am start -a android.intent.action.VIEW -d "myapp://deeplink"  # deep link

# 截图 / 录屏
adb exec-out screencap -p > screen.png
adb shell screenrecord /sdcard/demo.mp4   # Ctrl+C 结束
adb pull /sdcard/demo.mp4

# 输入模拟
adb shell input tap 500 800
adb shell input swipe 500 1500 500 500 300   # 从(500,1500)滑到(500,500),300ms
adb shell input text "hello"
adb shell input keyevent KEYCODE_BACK

# 网络模拟(弱网测试)
adb shell svc wifi disable / enable
adb shell svc data disable / enable
# 模拟器弱网:emulator 启动参数 -netdelay gprs -netspeed edge,或用抓包工具限速

# 性能快速查看
adb shell dumpsys meminfo com.example.myapp   # 内存
adb shell top -n 1 | grep myapp               # CPU
adb shell dumpsys gfxinfo com.example.myapp   # 帧率/卡顿
```

## Flutter 测试

三层金字塔:unit(纯 Dart 逻辑)→ widget(组件渲染与交互)→ integration(整机 E2E)。

```bash
flutter test                           # 跑 test/ 下所有单测和 widget 测试
flutter test test/login_test.dart      # 跑单个文件
flutter test --coverage                # 生成 coverage/lcov.info
flutter test integration_test          # 集成测试(需要连模拟器/真机)
flutter drive --driver=test_driver/integration_test.dart --target=integration_test/app_test.dart
```

Widget 测试要点:

```dart
testWidgets('登录按钮在表单为空时禁用', (tester) async {
  await tester.pumpWidget(const MaterialApp(home: LoginPage()));
  final button = find.byKey(const Key('login_button'));   // 组件加 Key 便于定位
  expect(tester.widget<ElevatedButton>(button).onPressed, isNull);

  await tester.enterText(find.byKey(const Key('username')), 'user1');
  await tester.enterText(find.byKey(const Key('password')), 'pass1');
  await tester.pump();                  // 触发重建;有动画用 pumpAndSettle()
  expect(tester.widget<ElevatedButton>(button).onPressed, isNotNull);
});
```

- 异步 UI 更新后必须 `pump()` / `pumpAndSettle()`,否则断言的是旧帧
- 网络依赖用 mockito/mocktail mock 掉,widget 测试不发真实请求
- 调试渲染问题:`flutter run` 后按 `p` 显示布局边界;DevTools 的 Widget Inspector

## React Native 测试

```bash
# 单测/组件测试:Jest + React Native Testing Library
npm test
npx jest --watch

# E2E 首选 Detox(或 Maestro,配置更简单)
npx detox test -c ios.sim.debug
```

组件测试用 `@testing-library/react-native`,查询优先 `getByTestId` / `getByText`;原生模块在 `jest.setup.js` 里 mock。RN 调试:摇一摇/Cmd+D 打开 Dev Menu,用 Flipper 或 React Native DevTools 看网络与日志。

## UniApp / 小程序

- H5 端:当普通 Vue 项目测,Chrome DevTools + Playwright 都可用
- 微信小程序端:微信开发者工具内置调试器(Console/Network/Storage/AppData 面板);自动化用 `miniprogram-automator`(需开发者工具开启服务端口)
- 多端一致性是 UniApp 测试重点:同一功能至少在 H5 + 一个小程序端 + 一个 App 端各验证一遍,条件编译(`#ifdef`)的分支要分别测

## App 闪退排查流程

1. **收集信息**:什么操作触发、必现还是偶现、设备/系统版本、App 版本
2. **看崩溃日志**:
   - iOS 模拟器:`xcrun simctl spawn booted log stream --predicate 'process == "MyApp"'`,崩溃报告在 `~/Library/Logs/DiagnosticReports/`
   - iOS 真机:Xcode → Window → Devices and Simulators → View Device Logs
   - Android:`adb logcat --buffer=crash`,或 `adb logcat *:E | grep -i "FATAL\|AndroidRuntime"`
   - Flutter:崩溃前通常有 Dart 异常栈,`flutter logs` 直接看
3. **符号化**(release 包的地址栈需要还原成代码行):iOS 用 dSYM + `symbolicatecrash`/`atos`;Android 用 `ndk-stack`(native)或 mapping.txt + R8 retrace(Java/Kotlin)
4. **定位常见原因**:空值解包(iOS force unwrap / Kotlin NPE)、主线程阻塞被 watchdog 杀(0x8badf00d)、内存不足(Jetsam)、原生插件版本不兼容(Flutter/RN 常见)
5. 产出:复现步骤 + 关键日志片段 + 定位结论
