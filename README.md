# Jagat H5 Bridge Plugin

面向接入 Jagat App 内嵌 WebView 的 H5 开发者的 Codex 插件。它把 `window.jagat` 桥接库的完整使用规范封装成一个技能，让 Codex 在编写、修改或排查 `jagat.xxx()` 调用代码时，能直接按正确的方式产出代码，而不是靠猜。


## 技能覆盖范围

技能详解桥接库的 17 个方法，并归纳为四类：

- **环境判断**：`isAndroid()`、`isIos()`、`inApp()`、`getEnv()`、`isChinaApp()`
- **用户与身份**：`getLocale()`、`getAccessToken()`、`getUser()`
- **App 交互**：`share()`、`enableShare()`、`setTitle()`、`track()`、`exit()`
- **页面跳转**：`openWebPageInApp()`、`openJagatApp()`、`openWebPageInBrowser()`、`openVipOrderPageInApp()`

其中 `getEnv()` 的返回值为 `'TEST'`、`'BETA'`、`'PROD'` 三个固定枚举。

技能正文的最前面提炼了 6 条生成代码前必须满足的约束，这是接入时最容易踩坑的地方：

1. 先用 `inApp()` 做环境守卫，再调用 App 专属能力，并为非 App 环境准备降级方案
2. 严格区分同步方法与 7 个返回 Promise 的异步方法，异步调用统一包 `try/catch`
3. 分享类操作必须由用户手势直接触发
4. 对 `null` 返回值做兜底处理
5. token 不得打印到控制台或上报到日志
6. 埋点属性保持扁平结构

## 使用安装

把插件挂到个人市场，即可在本机所有仓库中使用：

```bash
# 登记远程市场
codex plugin marketplace add https://github.com/huanqiuxuexiji/jagat-h5-bridge-plugin.git

# 查看可用插件
codex plugin list

# 安装
codex plugin install jagat-h5-bridge
```

## 本地安装（开发者自测）

把插件挂到个人市场，即可在本机所有仓库中使用：

```bash
mkdir -p ~/.agents/plugins
cp -r plugins/jagat-h5-bridge ~/plugins/jagat-h5-bridge
```

然后在 `~/.agents/plugins/marketplace.json` 中登记一条记录（若文件不存在则新建）：

```json
{
  "name": "personal",
  "interface": {
    "displayName": "My Plugins"
  },
  "plugins": [
    {
      "name": "jagat-h5-bridge",
      "source": { "source": "local", "path": "./plugins/jagat-h5-bridge" },
      "policy": { "installation": "AVAILABLE", "authentication": "ON_INSTALL" },
      "category": "Developer Tools"
    }
  ]
}
```

注意 `source.path` 是相对于**市场化根目录**解析的（即包含 `.agents` 的那一级），而不是相对市场化文件本身。因此 `~/.agents/plugins/marketplace.json` 里的 `./plugins/jagat-h5-bridge` 实际指向 `~/plugins/jagat-h5-bridge`。

安装后重启 Codex，用 `/skills` 确认 `jagat-h5-bridge` 出现在列表中，再以 `$jagat-h5-bridge` 显式调用验证。

## 团队成员安装（仓库级）

把本仓库作为团队的仓库级市场，成员克隆后即可在本仓库内使用：

- 市场化文件位于本仓库的 `.agents/plugins/marketplace.json`
- 插件源码位于 `plugins/jagat-h5-bridge/`

仓库级市场的优点是版本随代码走，插件更新与接口变更同步，不会出现"某个人本地还是旧版说明"的问题。

## 公开发布

公开发布有两条路径，详见 `PUBLISHING.md`：

- **Git 市场分发**（当前即可用）：把本仓库推到公开仓库，使用者通过 `codex plugin marketplace add` 登记后一键安装。适合团队与外部开发者分发，见效最快。
- **官方插件目录**（需审核）：提交到公共插件目录，需通过清单校验、素材合规检查与测试用例评估。审核标准高于本地安装，且不保证时限。

`PUBLISHING.md` 中给出了两条路径的完整操作步骤、提交前必须替换的占位符清单，以及官方校验的常见失败原因。

## 发布前待办

以下字段在 `plugin.json` 中：

- `author.name`、`author.email`、`author.url`
- `homepage`、`repository`
- `interface.developerName`
- `interface.websiteURL`、`interface.privacyPolicyURL`、`interface.termsOfServiceURL`

这些字段不影响本地与仓库级安装，但会直接导致官方目录提交校验失败。

## 版本约定

- `name` 一经发布**不可更改**，Codex 用它作为插件标识与组件命名空间，改名会被视为另一个插件而非版本更新
- `version` 遵循语义化版本，市场依赖它判断是否提示使用者升级
- 技能内容变更时同步递增 `version`

## 许可证

MIT，见 `LICENSE`。
