<p align="center">
  <a href="./README_en.md">English</a> · <strong>简体中文</strong>
</p>

<p align="center">
  <img src="docs/assets/banner.svg" alt="stop-guessing — 你的 agent 从来不读报错信息。这个技能让它读。" width="100%">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-3fb950?style=flat-square" alt="license MIT">
  <img src="https://img.shields.io/badge/type-agent%20skill-2f81f7?style=flat-square" alt="type agent skill">
  <img src="https://img.shields.io/badge/core-one%20SKILL.md-f0883e?style=flat-square" alt="core: one SKILL.md">
  <img src="https://img.shields.io/badge/version-1.0.0-a371f7?style=flat-square" alt="version 1.0.0">
  <img src="https://img.shields.io/badge/PRs-welcome-3fb950?style=flat-square" alt="PRs welcome">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/install-npx%20skills%20add%20cv--superding%2Fstop--guessing-8957e5?style=flat-square" alt="install">
  <img src="https://img.shields.io/badge/works%20with-Claude%20Code%20%C2%B7%20Codex%20%C2%B7%20Cursor%20%C2%B7%20Gemini%20CLI%20%C2%B7%20OpenCode-111111?style=flat-square" alt="works with">
  <img src="https://img.shields.io/badge/status-stable%20%C2%B7%20zero%20config-2f81f7?style=flat-square" alt="status">
</p>

<p align="center"><em>你的 agent 从来不读报错信息。这个技能让它读。</em></p>

一个给编程 agent 用的调试纪律技能，用固定流程替换 **"大概是 X 吧，我改一下试试"**：复现 → 定位 → 证据 → 最小修复 → 验证。加载这个技能后，agent 不能再把一个猜的修法叠在另一个猜的修法上面——必须先给你看证据。

兼容所有支持 [Agent Skills](https://agentskills.io) 的 agent：Claude Code、Codex CLI、Cursor、Gemini CLI、OpenCode 等。

## 纪律，画成一张图

<p align="center">
  <img src="docs/assets/diagram.png" alt="stop-guessing 流程图：复现 → 读完整报错 → 定位 → 唯一假设 → 最小修复 → 验证，带证伪回环和 Stop &amp; Ask 升级路径" width="100%">
</p>

上图由 [Archify](https://github.com/tt-a1i/archify) 生成——在浏览器中打开 [`docs/stop-guessing-diagram.html`](docs/stop-guessing-diagram.html) 可查看交互版（缩放、路径追踪、暗色主题）。

## 问题

让 agent 修一个 bug，通常会看到这样的对话：

> **Agent：** 啊，这大概是因为环境变量没加载。*（加了 `dotenv.config()`）* 我再顺手加个空值检查以防万一。*（又改了一处）* 现在应该可以了。

没有复现、没有证据、两次修改之间什么都没运行。bug 还活着，只是被一层"症状补丁"盖住了——接下来你要同时调试 bug 和补丁。

stop-guessing 就是一个 Markdown 文件，把上面这种行为改写成：

> **Agent（装了技能后）：** 已复现：`parse.ts:88` 抛 `TypeError`，进程退出码 1。
> 读了完整堆栈——崩溃点是"结果"不是"原因"。顺着数据查：`config.headers` 只在没有 auth token 时为 undefined。
> **根因：** `buildConfig()` 漏了 `headers` 的默认值。
> **修复：** `buildConfig()` 里改 1 行，未动其他代码。
> **验证：** 原命令退出码 0；`auth` 测试套件 14/14 通过。

## 安装

用 [Skills CLI](https://skills.sh)：

```bash
npx skills add cv-superding/stop-guessing --global
```

去掉 `--global` 只装在当前项目；加 `--agent '*'` 给所有检测到的 agent 安装。

Claude Code 2.1.142+ 也可以用插件方式：

```text
/plugin marketplace add cv-superding/stop-guessing
/plugin install stop-guessing@stop-guessing
```

手动安装：把 `SKILL.md` 复制进你的 agent 技能目录即可。

## 使用

不需要学任何新东西。只要你让 agent 修 bug、贴了报错、或者说"挂了" / "测试跑不过" / "为什么崩了"，技能就会触发。

也可以显式调用：

```text
/stop-guessing
auth.test.ts 在 CI 上挂了，但本地能过。查一下为什么。
```

## 它强制了什么

| 没有技能 | 装了技能 |
|---|---|
| 只看报错第一行就下结论 | 读完整堆栈；崩溃点 ≠ 病灶 |
| "大概是环境问题" | 一次只有一个假设，且必须附带观测到的证据 |
| 修 bug + 顺手重构 + 升级依赖塞进一个 diff | 最小 diff；顺手改动单独提议 |
| "改好了，应该可以了" | 重跑原始复现命令，贴出输出 |
| 同一条命令默默重试 5 次 | 两个假设被证伪就停下：报告已排除项，向你要只有你掌握的信息 |

完整纪律（含六种编号反模式及 Before/After 示例：霰弹枪式修复、症状补丁、口头诊断、顺手重构、玄学重试、静默认输）见 [`SKILL.md`](SKILL.md)。

## 实测：诚实的数据

两轮 A/B 对照实验：同一个 bug，两个互不知情的 agent 分别修，B 组遵循本 skill。

| 轮次 | Bug 类型 | 修复结果 | 无 skill | 有 skill |
|---|---|---|---|---|
| 1 | 编码坑：UTF-8 BOM → `KeyError`，带诱饵修法（加 `.get()` 症状补丁） | **两组一致**：都绕过诱饵，找到真根因，同一行修复 | 86k tokens | 118k tokens |
| 2 | 静默逻辑 bug：可变默认参数导致缓存污染，实际有两层缺陷 | **两组一致**：都找齐两层根因 | 132k tokens，场景通过即收工 | 284k tokens，多组回归验证 + **主动上报**一个超范围的既有缺陷 |

实测结论：

- **它不会让经典 bug 修得更"对"**——当前模型不装它也能修对；
- **它改变的是过程**：证据意识（缓存 dump、字节级检查）、验证彻底度（回归 + 幂等检查）、报告结构稳定可预期、不静默认输；
- **代价是 token**：约 1.4~2.2 倍。对简单 bug 是纯开销；对复杂多文件 bug、弱一档的模型、或你需要审计 agent 工作结果的场景，值得。

Use knowingly.（样本有限：两轮、4 个 agent、单一模型。想验证就跑你自己的对照。）

## 验证效果

给你的 agent 一个 bug，检查回复是否包含四个必需部分：**根因 → 证据 → 修复 → 验证**。如果它交付了未验证的修复，说明技能没加载——检查 agent 的技能列表。

## 许可

MIT
