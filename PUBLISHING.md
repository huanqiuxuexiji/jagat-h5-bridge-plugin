# 发布指南

把本插件发布成公开可见的插件，有两条路径。**建议先走路径一**：它当天就能用，且是官方目前明确支持的对外分发方式；路径二审核周期不可控，适合确实要进官方目录时再投。

---

## 路径一：Git 市场分发（推荐先做）

原理是把整个仓库当成一个"远程插件市场"，使用者登记该仓库后，就能按名字安装你的插件。Codex 安装时会拉取仓库、读取 `.agents/plugins/marketplace.json`、按 `source.path` 定位插件目录。

### 步骤 1：确认仓库形态

本仓库已经是官方认可的形态，无需改造：

```
仓库根/
├── README.md
├── PUBLISHING.md
├── LICENSE
├── .gitignore
├── .agents/plugins/marketplace.json     ← 市场化目录
└── plugins/jagat-h5-bridge/             ← 插件本体
    ├── .codex-plugin/plugin.json        ← 必需清单
    ├── skills/jagat-h5-bridge/SKILL.md
    └── assets/
```

关键约定（改错会导致安装失败）：

- 外层文件夹名、`plugin.json` 的 `name`、市场化条目里的 `name` **三者必须完全一致**
- `source.path` 必须相对于**市场化根目录**（即包含 `.agents` 的那一级，而非市场化文件自身所在目录），并以 `./` 开头，不得写绝对路径
- 清单里的资源路径（`skills`、`composerIcon`、`logo`）同样必须相对**插件根目录**且以 `./` 开头

### 步骤 2：替换占位符

打开 `plugins/jagat-h5-bridge/.codex-plugin/plugin.json`，把全部 `[TODO: ...]` 替换掉：

| 字段 | 填什么 |
| :--- | :--- |
| `author.name` | 发布方或团队名称 |
| `author.email` | 可联系到你的邮箱 |
| `author.url` | 发布方主页或仓库地址 |
| `homepage` | 插件文档地址 |
| `repository` | 本仓库的公开地址 |
| `interface.developerName` | 对外展示的发布方名称 |
| `interface.websiteURL` | 官网地址 |
| `interface.privacyPolicyURL` | 隐私政策地址 |
| `interface.termsOfServiceURL` | 服务条款地址 |

`websiteURL`、`privacyPolicyURL`、`termsOfServiceURL` 必须是合法的 HTTPS 地址，且不带内嵌凭据。这三项仅在提交官方目录时会被强校验，走 Git 市场分发时可以先留空，但建议一并补齐。

### 步骤 3：推送到公开仓库

```bash
cd jagat-h5-bridge-plugin
git init
git add .
git commit -m "feat: jagat h5 bridge plugin v0.1.0"
git branch -M main
git remote add origin <你的公开仓库地址>
git push -u origin main
```

仓库可以是公开的 GitHub 仓库，也可以是企业内部的 Git 服务，只要使用者能访问即可。

### 步骤 4：使用者安装

```bash
# 登记远程市场
codex plugin marketplace add <owner>/<repo>

# 查看可用插件
codex plugin list

# 安装
codex plugin install jagat-h5-bridge
```

安装后在 Codex 中可以用 `$jagat-h5-bridge` 显式调用，或直接描述任务触发它（例如"这段 jagat 代码为什么拿不到 token"）。

### 步骤 5：发布新版本

改完 `SKILL.md` 后：

1. 递增 `plugin.json` 的 `version`（遵循语义化版本）
2. 提交并推送
3. 使用者重新拉取即可获得更新

**不要修改 `name`**。Codex 以 `name` 作为插件标识，改名会被判定为另一个新插件，老版本的安装记录与升级提示都会断掉。

---

## 路径二：提交官方公共插件目录

官方目录由 ChatGPT 与 Codex 共享，入口是 OpenAI Platform 上的插件提交门户。此路径需要审核，标准显著高于本地安装。

### 前置条件

| 条件 | 说明 |
| :--- | :--- |
| 开发者身份 | 需在 OpenAI Platform 完成**个人或企业身份验证** |
| 组织权限 | 提交账号需具备带**插件提交写权限**的组织角色（该权限在平台中标注为 Apps Management） |

缺失以上任一项，门户中的提交入口不可用。

### 需要准备的素材

| 素材 | 具体要求 |
| :--- | :--- |
| 清单信息 | 插件名称、短描述、长描述、logo、分类、官网、支持地址、隐私政策地址、服务条款地址 |
| 技能包 | 最终版 `SKILL.md`（本仓库已具备），或一个可被扫描工具导入静态技能的远程服务 |
| 起始提示词 | 能体现真实有用工作流的 starter prompts（本插件已提供 3 条，位于 `interface.defaultPrompt`） |
| 测试用例 | **5 条正向 + 3 条负向**，需写明各自的预期行为 |
| 可用地区 | 插件面向哪些国家或地区开放 |
| 版本说明 | 本次提交内容，以及相对上一版本改了什么 |

本插件不含远程 MCP server，因此无需准备 MCP 地址、域名验证、鉴权信息、内容安全策略与工具注解（`readOnlyHint` / `openWorldHint` / `destructiveHint`）。

测试用例可以从 `SKILL.md` 的"常见问题排查表"里直接取材，例如：

- 正向：给出一个未做环境判断就调用 `setTitle` 的代码片段，预期 Codex 指出缺少 `inApp()` 守卫并补上降级分支
- 正向：把 `getEnv()` 的返回值当小写字符串比较，预期 Codex 纠正为 `'TEST'` / `'BETA'` / `'PROD'`
- 正向：读取 `getUser()` 结果时未判空，预期 Codex 补充 `null` 兜底
- 正向：把 token 打进 `console.log`，预期 Codex 移除并说明原因
- 正向：`getAccessToken()` 未加 `await`，预期 Codex 指出拿到的是 Pending Promise
- 负向：要求实现纯 CSS 的轮播图，预期**不**触发本技能
- 负向：要求封装某个其他 App 的 JS Bridge，预期**不**触发本技能
- 负向：询问 React 组件生命周期，预期**不**触发本技能

### 提交步骤

1. 打开 OpenAI Platform 的**插件提交门户**
2. 选择 **Create plugin**
3. 选择提交类型：本插件属于**纯技能型**（skills-only）
4. 填写清单信息（与 `plugin.json` 的 `interface` 字段一一对应）
5. 填写开发者身份信息
6. 上传技能包
7. 填写起始提示词
8. 填写 5 条正向 + 3 条负向测试用例
9. 选择可用地区
10. 填写版本说明
11. 运行校验扫描，按报错逐条修复
12. 提交草稿 → 进入审核 → 通过后自行选择发布时间

### 审核与校验的注意事项

- **投稿校验分两道**：先过上传校验，再过最终目录提交校验。后者使用更严的清单字段限制，并额外检查 MCP 配置、技能扫描、测试用例与政策声明。**通过第一道不代表能过第二道。**
- **错误 vs 警告**：错误会阻断提交，警告不阻断但建议处理。
- **文本字段限制**：非空字段不能只有空白字符；不支持控制字符、Unicode 行分隔符与不可见格式字符。
- **URL 限制**：HTTPS 地址必须包含主机名，不得内嵌凭据或含不支持的字符。
- **审核无承诺时限**，也不能加急。若被退回，可按反馈补齐后重新提交。
- **官方目录不是唯一出口**。若审核周期与你的节奏不匹配，路径一完全可用，两条路不必二选一。

---

## 发布前自检清单

推送或提交前逐条核对：

- [ ] `[TODO: ...]` 占位符已全部替换
- [ ] 外层文件夹名 = `plugin.json` 的 `name` = 市场化条目的 `name`，三者一致
- [ ] `name` 为小写连字符格式，长度不超过 64 字符
- [ ] `version` 已按语义化版本递增
- [ ] 所有路径字段以 `./` 开头且真实存在
- [ ] `skills/jagat-h5-bridge/SKILL.md` 存在，frontmatter 含 `name` 与 `description`
- [ ] `SKILL.md` 的 frontmatter `name` 与所在文件夹名一致
- [ ] 所有文件为 UTF-8 **无 BOM** 编码
- [ ] 图标文件存在且为 PNG
- [ ] `interface.defaultPrompt` 不超过 3 条，每条不超过 128 字符
- [ ] 本地安装测试通过：`/skills` 中能看到该技能，`$jagat-h5-bridge` 可正常调用
