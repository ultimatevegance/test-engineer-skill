# Web 前端测试(Chrome DevTools / Playwright / Cypress / Vue / 性能)

## Chrome DevTools 实战技巧

DevTools 是前端排查的第一现场,按面板分:

### Console

```js
// 常用但容易被忽略的
console.table(arrayOfObjects)        // 表格化输出数组对象
console.trace()                      // 打印调用栈,查"这函数谁调的"
monitor(someFunction)                // 函数被调用时自动打印(Console 专用 API)
monitorEvents(document.body, 'click') // 监听并打印事件
copy(someObject)                     // 复制变量到剪贴板
$0                                   // Elements 面板当前选中的元素
$$('.item')                          // querySelectorAll 简写,返回真数组
```

- 报错先看栈顶是不是自己的代码,压缩代码点开后用 `{}` 按钮 pretty-print,有 sourcemap 直接映射回源码
- `Uncaught (in promise)` 说明有 Promise 没有 catch,常见于请求失败没处理

### Network

- **Preserve log**:排查跳转/重定向过程必开,否则页面跳转后记录被清掉
- **Disable cache**:排查"改了没生效"问题必开
- 单个请求重点看:Status、Timing(TTFB 高是后端慢,Content Download 高是响应体大)、Request/Response Headers、Payload、Response
- 右键请求 → **Copy as cURL**:把浏览器请求原样复制出来命令行重放,是"前端问题还是后端问题"的分界法宝——cURL 重放结果不对就是后端问题
- 右键请求 → **Override content**(Chrome 117+):本地改写响应,模拟后端返回异常数据
- 网络条件模拟:Network 面板顶部 throttling 选 Slow 4G / 自定义,配合 Offline 测断网兜底
- 过滤技巧:`status-code:500`、`method:POST`、`larger-than:100k`、`-domain:*.google.com`(排除)

### 其他面板速用

- **Elements**:强制元素状态(:hover/:focus)调样式;DOM 断点(右键 → Break on → subtree modifications)抓"谁改了我的 DOM"
- **Sources**:条件断点(右键行号)、XHR/fetch 断点(抓"谁发的这个请求")、Local Overrides 本地改线上 JS 调试
- **Application**:LocalStorage/SessionStorage/Cookie/IndexedDB 直接查改;测登录态问题先看这里
- **Performance**:录制后看 Long Task(主线程 >50ms 的红块)定位卡顿
- **设备模拟**:Cmd+Shift+M 切移动端视口;Sensors 面板模拟定位和触摸

## Playwright(自动化首选)

```bash
npm init playwright@latest             # 初始化(含浏览器下载)
npx playwright test                    # 全部运行(默认无头)
npx playwright test --headed --project=chromium
npx playwright test -g "登录"          # 按标题过滤
npx playwright codegen https://example.com   # 录制生成脚本(快速起步)
npx playwright show-report             # 打开 HTML 报告
npx playwright test --trace on         # 失败排查神器:记录完整 trace
```

脚本模式(TypeScript):

```ts
import { test, expect } from '@playwright/test';

test.describe('登录流程', () => {
  test('正确账号密码登录成功', async ({ page }) => {
    await page.goto('/login');
    // 定位器优先级:getByTestId > getByRole/getByLabel > CSS
    await page.getByLabel('用户名').fill('user1');
    await page.getByLabel('密码').fill('correct-password');
    await page.getByRole('button', { name: '登录' }).click();
    // web-first 断言自带自动等待,不需要手动 sleep
    await expect(page).toHaveURL(/dashboard/);
    await expect(page.getByTestId('user-name')).toHaveText('user1');
  });

  test('密码错误提示且不跳转', async ({ page }) => {
    await page.goto('/login');
    await page.getByLabel('用户名').fill('user1');
    await page.getByLabel('密码').fill('wrong');
    await page.getByRole('button', { name: '登录' }).click();
    await expect(page.getByRole('alert')).toContainText('用户名或密码错误');
    await expect(page).toHaveURL(/login/);
  });
});
```

实用能力:

```ts
// Mock 接口(不依赖后端环境)
await page.route('**/api/user', route =>
  route.fulfill({ status: 200, body: JSON.stringify({ name: 'mock-user' }) }));

// 等待并断言某个请求
const respPromise = page.waitForResponse(r => r.url().includes('/api/login'));
await page.getByRole('button', { name: '登录' }).click();
const resp = await respPromise;
expect(resp.status()).toBe(200);

// 复用登录态(避免每个用例都走一遍登录 UI)
// 全局 setup 里登录一次:await page.context().storageState({ path: 'auth.json' });
// playwright.config.ts: use: { storageState: 'auth.json' }
```

反脆弱原则:不要 `page.waitForTimeout(3000)` 这类固定等待;断言用 `expect(locator)` 系列(自动重试);测试数据用 API 造而不是 UI 点出来。

## Cypress(备选)

项目已有 Cypress 时沿用即可:`npx cypress open`(交互)/ `npx cypress run`(CI)。与 Playwright 的主要差异:跑在浏览器内、命令自动重试、跨 tab/多域场景较弱。选择器同样优先 `data-cy` 属性。

## Vue 组件测试(Vitest + Vue Test Utils)

```bash
npm i -D vitest @vue/test-utils happy-dom
npx vitest                             # watch 模式
npx vitest run --coverage
```

```ts
import { mount } from '@vue/test-utils';
import { describe, it, expect, vi } from 'vitest';
import LoginForm from '@/components/LoginForm.vue';

describe('LoginForm', () => {
  it('提交时触发 submit 事件并携带表单值', async () => {
    const wrapper = mount(LoginForm);
    await wrapper.find('[data-testid="username"]').setValue('user1');
    await wrapper.find('[data-testid="password"]').setValue('pass1');
    await wrapper.find('form').trigger('submit.prevent');
    expect(wrapper.emitted('submit')![0]).toEqual([{ username: 'user1', password: 'pass1' }]);
  });
});
```

- Pinia 用 `@pinia/testing` 的 `createTestingPinia()`,action 默认已 stub
- 路由依赖:简单场景 mock `useRouter`,复杂场景建真实 router 传入 `global.plugins`
- 异步更新后 `await wrapper.vm.$nextTick()` 或 `await flushPromises()`(来自 `@vue/test-utils`)
- 组件测试测行为(用户输入 → 输出/事件),不测实现细节(内部 data 值、私有方法)

## 性能测试(Lighthouse)

```bash
npm i -g lighthouse
lighthouse https://example.com --output html --output-path report.html --preset=desktop
lighthouse https://example.com --only-categories=performance --form-factor=mobile
```

- 关注 Core Web Vitals:LCP(<2.5s)、INP(<200ms)、CLS(<0.1)
- 本地跑分受机器影响,横向对比要固定环境;线上真实数据看 CrUX/RUM
- DevTools 的 Lighthouse 面板可以直接跑,但命令行便于 CI 集成和批量对比
