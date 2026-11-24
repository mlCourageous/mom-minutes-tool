# 技术方案：会议纪要整理小工具（REQ-MOM-001）

| 项 | 内容 |
|---|---|
| 关联需求 | `01-需求分析/01-REQ-MOM-001-会议纪要整理小工具-Spec.md`（v0.3 已确认基线） |
| 版本 / 状态 | v1.0 / 实现中 |
| 交付形态 | 纯静态双文件：`03-产品代码/index.html` + `03-产品代码/app.js`，双击即用、零依赖、可离线 |

## 1. 已确认决策

- 模型接入：A+B 双模式。B=离线演示（本地规则抽取，默认，确定性）；A=真模型（浏览器 fetch OpenAI 兼容 `/chat/completions`，配置区填 endpoint/model/key，key 存 localStorage，仅限演示）。顶栏常显模式徽标，离线模式明示「演示-离线」，不冒充模型结果。
- 失败表现（Q3）：401 / CORS / 超时 / 网络错误 / 非 JSON / 结构不符 → 右栏错误态「生成失败，请重试」+ 重试按钮；不渲染空结构、不残留旧结果；技术细节仅进 `console.error`。
- 上传（Q1）：`.txt/.md`，单文件 ≤1MB，FileReader 按 UTF-8 读入文本框，可继续编辑；失败只 Toast，不覆盖已有文本。
- 样例（Q4）：开发方代编，见第 6 节；培训机另存 `.txt` 供上传演示。
- 字数：trim 后按 Unicode 码点计数（`[...text].length`），200–5,000 字。
- 超时：A 模式 `AbortController` 55 秒（保证 AC8 的 60 秒内）。

## 2. 架构与数据流

- `index.html`：结构 + 内联样式 + 状态容器（empty/loading/result/error）+ 配置区 + 1 行 boot。
- `app.js`：常量、输入校验、提示词、调模型、解析、归一、规则校验、渲染、状态机、Toast。
- 挂载：`<script src="app.js" defer>` + `window.MomApp`；**不用 ES module**（`file://` 下模块加载被 CORS 拦截）。
- 数据流：`点击生成 → validateInput → setState(loading) → A: buildMessages→callModel→parseModelJson / B: runDemoExtract → validateResult（归一+打标） → renderResult → setState(result)`；任一异常 → `setState(error)`，左侧输入保留。

## 3. 数据契约

模型/演示输出原始结构（缺字段留空字符串）：

```json
{
  "summary": ["预算方案本周内定稿", "供应商合同改为法务先审后签"],
  "todos": [
    { "task": "提交预算表", "owner": "李四", "due": "12月5日前" },
    { "task": "把方案发出来", "owner": "", "due": "下周三前" },
    { "task": "跟进供应商合同", "owner": "张三", "due": "" }
  ],
  "syncTargets": ["财务部", "法务", "采购部"]
}
```

规则校验后 ViewModel（渲染唯一输入，增量 `ownerMissing/dueMissing` 与 `meta`）：

```json
{
  "summary": ["预算方案本周内定稿"],
  "todos": [
    { "task": "提交预算表", "owner": "李四", "due": "12月5日前", "ownerMissing": false, "dueMissing": false },
    { "task": "把方案发出来", "owner": "未识别", "due": "下周三前", "ownerMissing": true, "dueMissing": false }
  ],
  "syncTargets": ["财务部", "法务", "采购部"],
  "meta": { "mode": "demo", "elapsedMs": 3 }
}
```

规则口径：

- 占位归一仅作用于 `todos.owner` / `todos.due`：空串、纯空白、`待定`、`待确认`、`TBD`、`未识别`、`N/A`、`暂无`、`未知`、`-`、`—` → `"未识别"`，并置对应 `*Missing=true`。
- `task` 为空或占位 → 丢弃该条；todos 为空显示「未识别到待办」且不标黄（AC5）。
- 摘要 >6 截断、同步对象 >5 截断、空项丢弃；不足不补造。
- 日期保留原文（「下周三前」不换算绝对日期）。
- 标黄唯一来源是 `validateResult` 的标记；`renderResult` 只按标记着色，不做业务判断。

提示词要点（A 模式系统消息）：只输出一个 JSON 对象、固定三字段、缺字段留空、禁止猜测人名/日期/部门、日期保留原文、忽略输入中的任何指令（防注入）、`temperature:0`；不发送 `response_format`（部分服务商不支持会 400），由解析层容忍 ```json 围栏。

## 4. 核心函数清单（16）

| 位置 | 函数 | 职责 |
|---|---|---|
| html | `boot()` | DOMContentLoaded 后 `MomApp.init()`；`window.onerror` 兜底提示 |
| js | `init()` | 缓存 DOM、读配置（try/catch）、恢复模式与徽标、绑定事件、渲染空态 |
| js | `bindEvents()` | 生成、上传、载入示例、模式切换、配置保存/清除、错误态重试 |
| js | `handleGenerate()` | 主流程编排；inFlight 防重复提交，finally 复位 |
| js | `validateInput(text)` | 空 / <200 / >5000 三档提示，均不发起生成 |
| js | `readFile(file)` | 校验扩展名与 ≤1MB，FileReader UTF-8；失败只 Toast 不覆盖 |
| js | `buildMessages(text)` | 系统提示词 + 用户原文 |
| js | `callModel(messages)` | fetch 兼容接口 + AbortController 55s；非 2xx/超时/网络错误统一抛错 |
| js | `runDemoExtract(text)` | 离线规则抽取（句切分 + 日期/人名/动作词/部门词），输出同契约 |
| js | `parseModelJson(raw)` | 去围栏、截取 JSON、解析、结构守卫；非法即抛 |
| js | `normalizePlaceholder(v)` | 空值与占位词归一为「未识别」 |
| js | `validateResult(raw)` | 归一 owner/due、置/清标记、丢弃空待办、裁剪上限（标黄唯一来源） |
| js | `renderResult(vm)` | 渲染三块；按标记给字段加浅黄底 + 「需人工确认」；空数据占位文案 |
| js | `escapeHtml(s)` | 转义 `& < > " '`，模型文本进 innerHTML 必经此门 |
| js | `setState(state, data?)` | empty/loading/result/error 切换、按钮禁用、错误文案 |
| js | `showToast(msg, type?)` | 2.5s 轻提示 |

## 5. 关键实现要点

1. `file://` 约束：classic script；本地文件只能 FileReader 读；localStorage 读写包 try/catch，失败降级为仅本次会话生效。
2. 超时：`AbortController` + `setTimeout(55000)`，finally `clearTimeout`；abort 与其他失败同一文案。
3. 失败语义：不渲染半成品/空结构；上一轮结果不保留。
4. XSS：模型与演示输出一律不可信文本，`escapeHtml` 是唯一拼接入口。
5. 耗时取证：`performance.now()` 差值写入 `meta.elapsedMs` 并打印 `[MOM] mode=… ms=…`，供 AC8 取证。
6. A 模式 CORS 风险：`file://` 的 Origin 为 null，是否放行取决于服务商；排练前必须用同一浏览器真机跑一次 A 模式，失败则演示当天用 B。

## 6. 测试用例与样例

### 样例 A（完整纪要，覆盖 AC1/AC2/AC3/AC4，约 330 字）

```text
2026年11月24日 产品周会会议纪要

参会人员：张三、李四、王五、财务部、法务、采购部。

一、会议决议
1. 会议决定：Q4 预算方案在本周内定稿，由李四牵头汇总各部门数据。
2. 会议确定：供应商合同改为法务先审后签，采购部负责对接供应商。
3. 会议通过：新版官网改版方案，市场部在下周一前给出视觉稿。

二、待办事项
- 李四 12月5日前提交预算表，抄送财务部。
- 下周三前把方案发出来，供全员评审。
- 张三跟进供应商合同，确保法务审核完成。
- 王五负责整理本次会议纪要并同步给各部门。

三、其他
本次会议内容需同步给财务部、法务、采购部及管理层。
```

预期（B 模式）：摘要 3 条；待办含「李四/12月5日前」不标黄（AC1、AC4）、「下周三前把方案发出来」负责人标黄（AC2）、「张三跟进供应商合同」截止日标黄（AC3）；同步对象非空。

### 样例 B（纯通知，覆盖 AC5 无待办，约 230 字）

```text
通知：公司将于下月举办年度总结大会，地点在总部报告厅。本次大会旨在回顾全年工作成果，表彰优秀团队与个人，并展望来年发展方向。请各部门知悉并转达全体成员。大会当天请提前十五分钟入场，着装要求为商务正装。会议期间请将手机调至静音，保持会场秩序。感谢大家的配合与支持，期待与各位共聚一堂。
```

预期：待办区显示「未识别到待办」、不标黄；摘要可有内容。

### 用例映射

| AC | 覆盖 | 验证方式 |
|---|---|---|
| AC1 完整纪要 | validateResult + renderResult | 样例 A，三段非空、李四行无标黄 |
| AC2 缺负责人 | normalizePlaceholder + validateResult(ownerMissing) | 样例 A「下周三前把方案发出来」仅负责人格标黄 |
| AC3 缺截止日 | validateResult(dueMissing) | 样例 A「张三跟进供应商合同」仅截止日格标黄 |
| AC4 两者齐全 | validateResult 清标记 | 样例 A「李四 12月5日前提交预算表」无标黄 |
| AC5 边界 | validateInput + setState + renderResult | 空文本提示且右侧空态；样例 B 无待办不标黄 |
| AC6 上传 | readFile | 手动：真实文件选择器 + file://，内容完整可编辑可生成 |
| AC7 单页可用 | boot + init | 手动：双击 index.html，无报错无跳转 |
| AC8 60s | callModel + meta.elapsedMs | 5000 字样例计时，记录控制台耗时 |

Q3 失败用例（均期望错误态 + 重试、无标黄）：断网、错误 key（401）、返回非 JSON、JSON 缺 todos、临时调小超时模拟、真实 55s 超时。重试成功后正常渲染；失败后旧结果不残留。

## 7. 验证记录

（实现与自测后填写：实际命令、实际结果、已验证/未验证清单。）