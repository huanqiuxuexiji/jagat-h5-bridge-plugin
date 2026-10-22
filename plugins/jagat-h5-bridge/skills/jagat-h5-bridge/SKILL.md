---
name: jagat-h5-bridge
description: 在 H5 网页中编写、修改或排查 jagat 全局桥接库（window.jagat）调用代码时使用，涵盖环境判断、登录态与用户信息获取、App 分享、右上角常驻分享开关、网页标题设置、埋点上报、退出、App 内/浏览器内页面跳转与 VIP 订阅页唤起。当代码中出现 jagat.xxx() 调用、需要判断页面是否运行在 Jagat App 内、需要复用 App 的 昵称头像、需要从外部浏览器唤醒 App，或出现 jagat 未定义、拿到的返回值是 Promise 对象、分享打不开 等问题时，触发本技能。不适用于与本库无关的普通 H5 开发、其他 App 的 JS Bridge 封装对接，以及纯 CSS 样式调整与 UI 还原任务。
---

# jagat H5 桥接库

`jagat` 是 H5 网页与 Jagat App 原生能力通信的全局桥接库。通过 `<script src="https://www.jagat.io/registry/jagat_sdk.js"></script>` 引入后，它在全局（`window`）挂载 `jagat` 对象，页面代码直接调用 `jagat.xxx()` 即可使用 App 的原生能力，共 **17 个方法**。

本技能是该库的完整权威参考：包含全部方法签名、参数、返回值、正确用法，以及生成代码时必须遵守的约束。**在编写或修改任何涉及 `jagat` 的代码前，先读完「一、生成代码前的 6 条硬性约束」。**

---

## 一、生成代码前的 6 条硬性约束

这 6 条是接入该库时最容易出错的地方，生成代码时必须逐条满足。

### 1. 先判断环境，再调用 App 专属能力

`inApp()` 是所有 App 专属能力的前置判断依据，**必须**作为最外层守卫。`share`、`enableShare`、`setTitle`、`track`、`exit`、`openVipOrderPageInApp`、`openWebPageInApp` 都应在 `if (jagat.inApp())` 分支内调用，并为非 App 环境准备降级方案（通常是引导 `openJagatApp()` 或提示去应用商店）。

```javascript
if (jagat.inApp()) {
  jagat.openVipOrderPageInApp();
} else {
  jagat.openJagatApp(); // 降级：引导打开 App
}
```

### 2. 严格区分同步方法与异步方法

以下 7 个方法返回 `Promise`，**必须**用 `await` 或 `.then()` 取值：

`getEnv()`、`isChinaApp()`、`getLocale()`、`getUser()`、`share()`、`enableShare()`

其余 10 个方法是同步的，直接读写返回值。**给异步方法加上 `await` 是最常被漏掉的一步**——漏掉后拿到的是 `Pending` 状态的 Promise 对象，而不是数据，且不会报错，极难排查。异步调用统一包裹 `try/catch`：

```javascript
try {
  const user = await jagat.getUser();
} catch (e) {
  console.error('获取用户信息失败', e); // 桥接通信异常时降级
}
```

### 3. 唤醒类操作必须由用户手势触发

`openJagatApp()`、`openWebPageInApp()`、`share()` 会唤起 App 或弹出系统面板，**只能**放在用户点击回调中执行。绝对不要在页面加载、定时器、请求回调里自动调用，否则会被浏览器或系统安全策略拦截，表现为"点了没反应"。

### 4. 一律做空值兜底

`getEnv()`、`getUser()` 都可能返回 `null`。使用前必须判空，否则会抛 `Cannot read properties of null`。

```javascript
const user = await jagat.getUser();
if (!user) {
  /* 未登录态处理 */ return;
}
```

### 5. 埋点属性保持扁平

`track()` 的 `eventAttr` 只传字符串、数字、布尔等基础类型，**禁止**嵌套对象或数组，否则 App 端可能解析失败。同时 `track()` 是同步方法、无返回值，**不要**用它的返回值做流程判断。

---

## 二、API 总览

| 分类       | 方法                        | 调用方式 | 功能摘要                        |
| :--------- | :-------------------------- | :------- | :------------------------------ |
| 环境判断   | `isAndroid()`               | 同步     | 是否是安卓                      |
| 环境判断   | `isIos()`                   | 同步     | 是否是 iOS                      |
| 环境判断   | `inApp()`                   | 同步     | 是否处于 App 内                 |
| 环境判断   | `getEnv()`                  | **异步** | 获取当前环境值                  |
| 环境判断   | `isChinaApp()`              | **异步** | 是否处于国内 App                |
| 用户与身份 | `getLocale()`               | **异步** | 获取当前 App 的语言，如 `zh-CN` |
| 用户与身份 | `getUser()`                 | **异步** | 获取当前用户信息（昵称、头像）  |
| App 交互   | `share(params)`             | **异步** | 调用 App 分享                   |
| App 交互   | `enableShare(params)`       | **异步** | 开启 App 右上角头部分享         |
| App 交互   | `setTitle(title)`           | 同步     | 设置 App 内网页的标题           |
| App 交互   | `track(params)`             | 同步     | 调用 App 埋点方法               |
| App 交互   | `exit()`                    | 同步     | 退出                            |
| 页面跳转   | `openWebPageInApp(url)`     | 同步     | 唤醒 App，并在 App 内打开网页   |
| 页面跳转   | `openJagatApp()`            | 同步     | 打开 AJAGAT APP                 |
| 页面跳转   | `openWebPageInBrowser(url)` | 同步     | 使用手机浏览器打开网页          |
| 页面跳转   | `openVipOrderPageInApp()`   | 同步     | 打开 VIP 订阅页                 |

---

## 三、环境判断

### `isAndroid(): boolean`

判断当前运行环境是否是安卓（同步，直接返回结果）。

```javascript
if (jagat.isAndroid()) {
  console.log('当前是安卓设备');
}
```

### `isIos(): boolean`

判断当前运行环境是否是 iOS（同步）。

```javascript
if (jagat.isIos()) {
  console.log('当前是 iOS 设备');
}
```

> `isAndroid()` 与 `isIos()` 通常配合使用做分端兜底判断：**两者都为 `false` 时，说明多半运行在普通浏览器等非移动端环境**。

### `inApp(): boolean`

判断当前页面是否运行在 App 内（即被 App 内嵌的 WebView 打开）。这是所有 App 专属能力调用的前置校验。同步。

```javascript
if (jagat.inApp()) {
  document.getElementById('app-only-entry').style.display = 'block';
} else {
  document.getElementById('open-app-guide').style.display = 'block';
}
```

### `getEnv(): Promise<string | null>`

获取当前环境值，成功返回环境枚举字符串，取不到返回 `null`。**异步。**

**环境枚举值（固定三个大写字符串，不得写小写或其他变体）：**

| 枚举值   | 说明             |
| :------- | :--------------- |
| `'TEST'` | 测试环境         |
| `'BETA'` | 灰度（预发）环境 |
| `'PROD'` | 生产环境         |

```javascript
async function checkEnv() {
  const env = await jagat.getEnv();
  if (env === null) {
    console.warn('未能获取到环境值，默认使用生产环境');
    return;
  }
  switch (env) {
    case 'TEST':
      console.log('当前为测试环境');
      break;
    case 'BETA':
      console.log('当前为灰度环境');
      break;
    case 'PROD':
      console.log('当前为生产环境');
      break;
    default:
      console.log('未知环境：', env);
  }
}
checkEnv();
```

### `isChinaApp(): Promise<boolean>`

判断当前是否处于国内 App。国内版返回 `true`，海外版返回 `false`。**异步。**

```javascript
async function initByRegion() {
  const isChina = await jagat.isChinaApp();
  if (isChina) {
    // 国内版本逻辑
  } else {
    // 海外版本逻辑
  }
}
initByRegion();
```

---

## 四、用户与身份

### `getLocale(): Promise<string>`

获取当前 App 的语言标识，形如 `zh-CN`。**异步。**

```javascript
async function applyLocale() {
  const locale = await jagat.getLocale();
  if (locale === 'zh-CN') {
    document.title = '首页';
  } else {
    document.title = 'Home';
  }
}
applyLocale();
```

### `getUser(): Promise<{ nickname: string; avatar: string } | null>`

获取当前用户信息。未登录或取不到时返回 `null`。**异步。**

返回字段：

| 字段       | 类型     | 说明         |
| :--------- | :------- | :----------- |
| `nickname` | `string` | 用户昵称     |
| `avatar`   | `string` | 用户头像地址 |

```javascript
async function renderUser() {
  const user = await jagat.getUser();
  if (!user) {
    document.getElementById('nickname').textContent = '点击登录';
    document.getElementById('avatar').src = './default-avatar.png';
    return;
  }
  document.getElementById('nickname').textContent = user.nickname;
  document.getElementById('avatar').src = user.avatar;
}
renderUser();
```

---

## 五、App 交互

### `share(params): Promise<boolean>`

调用 App 分享，唤起原生分享面板。分享成功返回 `true`；用户取消或失败返回 `false`。**异步。**

```typescript
share: (params: {
  businessId: number; // 用户id
  shareType: string; // 分享类型
  text: string; // 分享文本
  picData: string; // 分享图
}) => Promise<boolean>;
```

| 参数         | 类型     | 必填 | 说明                            |
| :----------- | :------- | :--- | :------------------------------ |
| `businessId` | `number` | 是   | 用户 id                         |
| `shareType`  | `string` | 是   | 分享类型，具体枚举由 App 端约定 |
| `text`       | `string` | 是   | 分享文本                        |
| `picData`    | `string` | 是   | 分享图                          |

```javascript
async function onShareClick() {
  const ok = await jagat.share({
    businessId: 10086,
    shareType: 'link',
    text: '快来看看这个内容',
    picData: 'https://example.com/share.png',
  });

  if (ok) {
    console.log('分享成功');
  } else {
    console.log('分享未完成');
  }
}
```

> 必须由用户点击手势触发，**禁止**在页面加载时自动调用。

### `enableShare(params): Promise<void>`

开启 / 关闭 App 右上角**常驻**的分享入口，无返回数据。**异步。**

```typescript
enableShare: (params: {
  enable: boolean; // 是否开启
  data: any; // 分享数据
  text: string; // 分享文本
  picData: string; // 分享图
  businessType: number;
  businessId: number;
}) => Promise<void>;
```

| 参数           | 类型      | 必填 | 说明                        |
| :------------- | :-------- | :--- | :-------------------------- |
| `enable`       | `boolean` | 是   | `true` 开启，`false` 关闭   |
| `data`         | `any`     | 是   | 分享数据，结构由 App 端约定 |
| `text`         | `string`  | 是   | 分享文本                    |
| `picData`      | `string`  | 是   | 分享图                      |
| `businessType` | `number`  | 是   | 业务类型，枚举由 App 端约定 |
| `businessId`   | `number`  | 是   | 用户 id                     |

```javascript
await jagat.enableShare({
  enable: true,
  data: { url: location.href },
  text: '分享给你',
  picData: 'https://example.com/share.png',
  businessType: 1,
  businessId: 10086,
});

// 离开页面时关闭，避免影响其他不需要分享的页面
await jagat.enableShare({
  enable: false,
  data: {},
  text: '',
  picData: '',
  businessType: 1,
  businessId: 10086,
});
```

> **与 `share()` 的区别**：`share()` 是代码主动触发一次分享；`enableShare()` 是开启/关闭右上角常驻入口，由用户自己点击触发。

### `setTitle(title): void`

设置 App 内网页的标题（同步）。**只在 App 内生效**，普通浏览器中标题由页面自身 `<title>` 决定。

| 参数    | 类型     | 必填 | 说明             |
| :------ | :------- | :--- | :--------------- |
| `title` | `string` | 是   | 要设置的网页标题 |

```javascript
jagat.setTitle('活动详情');
```

### `track(params): void`

调用 App 埋点方法上报自定义事件（同步，无返回值）。

```typescript
track: (params: { eventName: string; eventAttr: Record<string, any> }) => void
```

| 参数        | 类型                  | 必填 | 说明                                 |
| :---------- | :-------------------- | :--- | :----------------------------------- |
| `eventName` | `string`              | 是   | 事件名称                             |
| `eventAttr` | `Record<string, any>` | 是   | 事件属性，键值对形式，值只用基础类型 |

```javascript
jagat.track({
  eventName: 'click_buy_button',
  eventAttr: {
    page: 'detail',
    productId: 123,
    position: 'bottom_bar',
  },
});
```

> 同步方法、无返回值，不要依赖其返回值做流程判断。

### `exit(): void`

退出（同步）。不可逆操作，调用前建议加二次确认。

```javascript
document.getElementById('exit-btn').addEventListener('click', () => {
  jagat.exit();
});
```

---

## 六、页面跳转

### `openWebPageInApp(url): void`

唤醒 App，并在 App 内打开网页（同步）。

| 参数  | 类型     | 必填 | 说明                      |
| :---- | :------- | :--- | :------------------------ |
| `url` | `string` | 是   | 要在 App 内打开的网页地址 |

```javascript
jagat.openWebPageInApp('https://example.com/detail?id=123');
```

> 依赖 App 唤醒能力，需由用户点击触发。

### `openJagatApp(): void`

打开 AJAGAT APP（同步，无参数）。

```javascript
if (!jagat.inApp()) {
  document.getElementById('open-app').addEventListener('click', () => {
    jagat.openJagatApp();
  });
}
```

> 唤醒 App 通常要求用户手势触发，部分浏览器可能拦截，属正常现象，需做好降级提示（如引导前往应用商店）。

### `openWebPageInBrowser(url): void`

使用手机浏览器打开网页（同步）。

| 参数  | 类型     | 必填 | 说明                         |
| :---- | :------- | :--- | :--------------------------- |
| `url` | `string` | 是   | 要用手机浏览器打开的网页地址 |

```javascript
jagat.openWebPageInBrowser('https://example.com');
```

### `openVipOrderPageInApp(): void`

打开 VIP 订阅页（同步，无参数）。

```javascript
document.getElementById('vip-btn').addEventListener('click', () => {
  if (jagat.inApp()) {
    jagat.openVipOrderPageInApp();
  } else {
    jagat.openJagatApp();
  }
});
```

---

## 七、标准接入范式

新页面接入时，可按下面这套结构起步：先做环境判断，再并行拉取身份信息，全程兜底。

```javascript
async function initPage() {
  // 1. 同步环境判断，决定走哪套逻辑
  const isInApp = jagat.inApp();
  const isAndroid = jagat.isAndroid();

  if (!isInApp) {
    // 非 App 环境：展示引导打开 App 的入口，不调用任何 App 专属能力
    document.getElementById('open-app-guide').style.display = 'block';
    return;
  }

  // 2. 设置 App 内标题
  jagat.setTitle('活动详情');

  // 3. 并行拉取身份信息，统一兜底
  try {
    const [locale, user] = await Promise.all([jagat.getLocale(), jagat.getUser()]);

    if (user) {
      document.getElementById('nickname').textContent = user.nickname;
      document.getElementById('avatar').src = user.avatar;
    }
  } catch (e) {
    console.error('身份信息获取失败，已降级展示默认内容', e);
  }

  // 4. 上报页面曝光
  jagat.track({ eventName: 'page_view', eventAttr: { page: 'detail' } });
}

initPage();
```

---

## 八、TypeScript 类型声明

TypeScript 项目可将以下声明加入 `global.d.ts`，获得完整类型提示：

```typescript
declare const jagat: {
  /** 是否是安卓 */
  isAndroid: () => boolean;
  /** 是否是ios */
  isIos: () => boolean;
  /** 是否处于app内 */
  inApp: () => boolean;
  /** 获取当前环境值 */
  getEnv: () => Promise<string | null>;
  /** 是否处于国内app */
  isChinaApp: () => Promise<boolean>;
  /** 退出 */
  exit: () => void;
  /** 调用app分享 */
  share: (params: {
    /** 用户id */
    businessId: number;
    /** 分享类型 */
    shareType: string;
    /** 分享文本 */
    text: string;
    /** 分享图 */
    picData: string;
  }) => Promise<boolean>;
  /** 设置app内网页的标题 */
  setTitle: (title: string) => void;
  /** 开启app右上角头部分享 */
  enableShare: (params: {
    enable: boolean;
    /** 分享数据 */
    data: any;
    /** 分享文本 */
    text: string;
    /** 分享图 */
    picData: string;
    businessType: number;
    businessId: number;
  }) => Promise<void>;
  /** 调用app埋点方法 */
  track: (params: { eventName: string; eventAttr: Record<string, any> }) => void;
  /** 获取当前app的语言 zh-CN */
  getLocale: () => Promise<string>;
  /** 获取当前用户信息 */
  getUser: () => Promise<{ nickname: string; avatar: string } | null>;
  /** 唤醒app，并在app内打开网页 */
  openWebPageInApp: (url: string) => void;
  /** 打开AJAGAT APP */
  openJagatApp: () => void;
  /** 使用手机浏览器打开网页 */
  openWebPageInBrowser: (url: string) => void;
  /** 打开Vip订阅页 */
  openVipOrderPageInApp: () => void;
};

interface Window {
  jagat: typeof jagat;
}
```

---

## 九、常见问题排查

| 现象                              | 根因                                         | 修复方式                                                                                           |
| :-------------------------------- | :------------------------------------------- | :------------------------------------------------------------------------------------------------- |
| `jagat is not defined`            | 脚本未加载，或在模块化环境中试图 `import` 它 | 确认 `<script src="./jagat.js"></script>` 已在业务代码前加载；该库是全局挂载，不要用 `import` 引入 |
| 拿到的是 `Promise` 对象而不是数据 | 对异步方法漏了 `await`                       | 给 `getEnv`/`isChinaApp`/`getLocale`/`getUser`/`share`/`enableShare` 补上 `await`                  |
| `Cannot read properties of null`  | 未对 `null` 返回值判空                       | `getEnv`、`getUser` 返回值使用前先判断                                                             |
| 调用后毫无反应、控制台也无报错    | 当前不在 App 内，桥接层未注入                | 先用 `inApp()` 判断，非 App 环境走降级分支                                                         |
| 分享面板不弹、App 唤不起来        | 在非用户手势的时机调用                       | 把调用移入 `click` 等用户手势回调                                                                  |
| `setTitle` 设置后页面标题没变     | 当前不在 App 内                              | 该能力仅 App 内生效，普通浏览器需改 `<title>`                                                      |
| 埋点数据在 App 端缺失             | `eventAttr` 传了嵌套对象                     | 拆平为一层基础类型键值对                                                                           |
| 环境判断逻辑失效                  | 硬编码了小写或变体字符串                     | 只使用 `'TEST'` / `'BETA'` / `'PROD'` 三个大写枚举值                                               |

---

## 十、速查表

### 需要传参的方法

| 方法                   | 参数           | 类型                  | 说明                     |
| :--------------------- | :------------- | :-------------------- | :----------------------- |
| `share`                | `businessId`   | `number`              | 用户 id                  |
| `share`                | `shareType`    | `string`              | 分享类型                 |
| `share`                | `text`         | `string`              | 分享文本                 |
| `share`                | `picData`      | `string`              | 分享图                   |
| `enableShare`          | `enable`       | `boolean`             | 是否开启右上角分享       |
| `enableShare`          | `data`         | `any`                 | 分享数据                 |
| `enableShare`          | `text`         | `string`              | 分享文本                 |
| `enableShare`          | `picData`      | `string`              | 分享图                   |
| `enableShare`          | `businessType` | `number`              | 业务类型                 |
| `enableShare`          | `businessId`   | `number`              | 用户 id                  |
| `setTitle`             | `title`        | `string`              | 网页标题                 |
| `track`                | `eventName`    | `string`              | 事件名称                 |
| `track`                | `eventAttr`    | `Record<string, any>` | 事件属性                 |
| `openWebPageInApp`     | `url`          | `string`              | App 内打开的网页地址     |
| `openWebPageInBrowser` | `url`          | `string`              | 手机浏览器打开的网页地址 |

### 无参方法

`isAndroid()`、`isIos()`、`inApp()`、`getEnv()`、`isChinaApp()`、`exit()`、`getLocale()`、`getUser()`、`openJagatApp()`、`openVipOrderPageInApp()`

### 返回值速查

| 方法                                                                                           | 返回值类型                              | 说明                      |
| :--------------------------------------------------------------------------------------------- | :-------------------------------------- | :------------------------ |
| `isAndroid()`                                                                                  | `boolean`                               | 是否安卓                  |
| `isIos()`                                                                                      | `boolean`                               | 是否 iOS                  |
| `inApp()`                                                                                      | `boolean`                               | 是否在 App 内             |
| `getEnv()`                                                                                     | `Promise<string \| null>`               | 环境值，取不到为 `null`   |
| `isChinaApp()`                                                                                 | `Promise<boolean>`                      | 是否国内 App              |
| `getLocale()`                                                                                  | `Promise<string>`                       | 语言标识，如 `zh-CN`      |
| `getUser()`                                                                                    | `Promise<{ nickname, avatar } \| null>` | 用户信息，取不到为 `null` |
| `share()`                                                                                      | `Promise<boolean>`                      | 是否分享成功              |
| `enableShare()`                                                                                | `Promise<void>`                         | 无返回数据                |
| `exit()` / `setTitle()` / `track()`                                                            | `void`                                  | 无返回值                  |
| `openWebPageInApp()` / `openJagatApp()` / `openWebPageInBrowser()` / `openVipOrderPageInApp()` | `void`                                  | 无返回值                  |
