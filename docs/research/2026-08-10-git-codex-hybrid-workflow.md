# 本地 Codex × GitHub × Codex cloud 混合开发工作流研究

> 研究日期：2026-08-10
>
> 适用场景：从空 GitHub 仓库起步，本地 Codex/本地开发与网页版 Codex cloud 通过 GitHub 分支和 Pull Request（PR）交接。
>
> 来源边界：只采用 OpenAI 与 GitHub 官方文档。本文将“官方事实”“建议”“推断”分开标注。

## 结论摘要

推荐把 GitHub 作为唯一共享状态，把 PR 作为人、本地 Codex 与 Codex cloud 的共同任务界面：

1. 本地先建立可复现的最小项目骨架、`AGENTS.md`、测试命令和 GitHub Actions，并形成第一个 `main` 提交。
2. 每项工作使用独立短分支；本地交给云端前，必须提交并推送一个明确的“交接提交”。
3. 在网页版 Codex cloud 选择该仓库环境及分支/提交，或在现有 PR 中用 `@codex` 发起带 PR 上下文的云端任务。
4. 云端只向功能分支回写，不直接把 `main` 当工作区；结果以 diff 和 PR 接受审查。
5. GitHub Actions、分支保护和本地复验是合并门禁；`@codex review` 是额外审查层，不替代这些门禁。
6. 对个人仓库，优先要求“必须走 PR + CI 通过 + 对话已解决”，但不要贸然要求一个无法由自己满足的独立人工批准；有合作者后再要求至少一名人工审批。

关键澄清：用户口中的“网页版 GPT 接手代码”在官方产品路径中应落到 **Codex cloud**。它从网页启动、使用 ChatGPT 账户登录，但代码交接能力来自 Codex cloud 的 GitHub 仓库与云环境集成，而不是普通 ChatGPT 对话。[OpenAI：Codex cloud](https://learn.chatgpt.com/docs/cloud)

## 一、官方事实

### 1. Codex cloud 的工作与交付边界

- Codex cloud 在隔离的云环境中运行任务，可以从网页或 GitHub 等入口启动；任务完成后可检查摘要和 diff、继续追问或打开 PR。[OpenAI：Codex cloud](https://learn.chatgpt.com/docs/cloud)
- 创建云任务时，Codex 会检出选定的分支或 commit SHA，运行环境设置脚本，然后由代理修改代码并尝试执行检查。[OpenAI：Cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment)
- 云环境可以固定运行时版本、安装依赖/格式化器/检查器，并设置环境变量。常见包管理器可自动安装依赖，复杂项目可使用自定义 setup script。[OpenAI：Cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment)
- setup script 阶段可以联网；代理阶段默认关闭联网，可按需要配置受限或不受限访问。[OpenAI：Cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment)
- 云环境中的普通环境变量在 setup 和代理阶段都存在；“secret”只在执行 setup script 时解密，进入代理阶段前会被移除。[OpenAI：Cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment)
- 云端容器可能缓存；恢复缓存后会检出任务指定分支并可运行 maintenance script。修改 setup、maintenance、环境变量或 secret 会自动使缓存失效。[OpenAI：Cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment)

### 2. `AGENTS.md` 是本地与云端的一致性契约

- Codex 在工作前读取 `AGENTS.md`。项目级指令从 Git 根目录向当前工作目录逐层发现，越靠近目标文件的指令优先级越高；默认合并上限为 32 KiB。[OpenAI：AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- OpenAI 明确举例把 lint、test 等仓库约定写在项目根 `AGENTS.md`；云环境文档也说明 Codex 会从其中寻找项目特定的 lint/test 命令。[OpenAI：AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)，[OpenAI：Cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment)
- Codex GitHub Code Review 同样读取仓库内适用的 `AGENTS.md`；仓库级审查规则放根文件，局部规则可放更深目录。[OpenAI：GitHub Code Review](https://learn.chatgpt.com/docs/third-party/github)

### 3. GitHub 集成与 Codex 审查

- 仓库完成 Codex cloud 设置后，可在 PR 评论中写 `@codex review` 请求代码审查，也可启用每个新 PR 的自动审查。[OpenAI：GitHub Code Review](https://learn.chatgpt.com/docs/third-party/github)
- 在 PR 中用 `@codex` 加上 review 以外的任务，会以该 PR 为上下文启动云端任务；例如 `@codex fix the CI failures`。在具备权限时，Codex 可以把修复推回该分支。[OpenAI：GitHub Code Review](https://learn.chatgpt.com/docs/third-party/github)
- 官方建议把高影响、仓库特定的语义约束写进 `## Code Review Rules`，而把格式化、lint 等确定性检查留给 CI。[OpenAI：GitHub Code Review](https://learn.chatgpt.com/docs/third-party/github)
- OpenAI 明确指出：Codex 审查规则不能替代测试、分支保护或必需审批。[OpenAI：GitHub Code Review](https://learn.chatgpt.com/docs/third-party/github)

### 4. GitHub 的基础协作门禁

- GitHub Flow 是轻量的分支工作流：为每组无关更改建立独立分支，在分支上提交并推送，建立 PR、处理反馈，再合并。GitHub 建议提交是隔离且完整的变化。[GitHub：GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow)
- GitHub Actions 工作流以仓库中 `.github/workflows/` 下的 YAML 定义，可在 PR 事件上运行构建与测试。[GitHub：Workflows](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflows)
- 受保护分支可以要求 PR 审查、状态检查、对话解决、线性历史，并默认禁止强推或删除；必需检查失败时无法合并。[GitHub：About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- 必需状态检查必须针对最新提交成功；同名 job 出现在多个 workflow 中可能造成检查歧义并阻塞合并。[GitHub：Troubleshooting required status checks](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks)，[GitHub：About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- 将本地代码首次发布到空 GitHub 仓库的官方路径包括：形成初始提交、确认 `origin`、再推送 `main`；GitHub 同时警告不要提交或推送密码/API 密钥。[GitHub：Adding locally hosted code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)

## 二、推荐的仓库最小骨架

以下为**建议**，目的是让本地与云端使用同一套可执行事实：

```text
AGENTS.md
README.md
docs/
  development-workflow.md
  research/
.github/
  pull_request_template.md
  workflows/ci.yml
src/                   # 按实际技术栈调整
tests/                 # 按实际技术栈调整
```

根 `AGENTS.md` 建议只放耐久规则：

- 项目目标与关键目录；
- 唯一安装命令、lint/test/typecheck/build 命令；
- 完成定义（Definition of Done）；
- 安全边界与禁止提交的敏感信息；
- PR 前必须给出的验证证据；
- 两三条真正需要语义判断的 `## Code Review Rules`。

**推断：** 如果命令只存在于某台本地电脑的全局配置中，Codex cloud 无法从 Git 仓库重建它。因而，凡是云端任务必须知道的规则和脚本，都应版本化进仓库或云环境配置，而不是只保存在本地提示词中。该推断来自 Codex cloud“检出仓库 + 运行所选环境配置”的执行模型。

## 三、从空仓库起步的实施顺序

### 阶段 A：建立可共享的基线

1. 在本地空 clone 中创建最小项目骨架、`AGENTS.md`、README 和一条能成功运行的烟雾测试。
2. 创建一个针对 `pull_request`（也可加 `push` 到 `main`）的 `ci.yml`，使用固定且唯一的 job 名称，例如 `quality`。
3. 本地依次运行仓库声明的 lint/test/typecheck/build，记录输出。
4. 创建初始提交并推送 `main`。首次提交前检查 `.gitignore` 和暂存区，确保没有 token、密码、证书或本机配置。
5. 让 Actions 至少成功运行一次，再把实际出现的检查名设为 `main` 的 required status check。GitHub 要求必需检查必须在仓库内近期成功出现；过早配置一个尚未产生的检查名可能永久等待状态。
6. 为 `main` 启用：Require a pull request before merging、Require status checks、Require conversation resolution；保持禁止 force push 和禁止删除。
7. 在 Codex cloud 连接 GitHub 仓库，创建云环境，固定语言/包管理器版本，并让 setup script 调用与本地相同的安装路径。

### 阶段 B：一次标准开发循环

#### 1. 本地开工

```bash
git switch main
git pull --ff-only
git switch -c codex/<task-slug>
```

先写清任务契约：目标、非目标、验收标准、验证命令。它可以进入 Issue、draft PR 或仓库内计划文档。

#### 2. 本地实现或形成交接点

本地 Codex 可以完成探索、测试先行、小步实现和本地验证。准备转交云端时：

```bash
git status
git diff --check
git add <明确文件>
git commit -m "chore: checkpoint for cloud handoff"
git push -u origin codex/<task-slug>
git rev-parse HEAD
```

在 draft PR 中记录该 commit SHA、已完成内容、未完成内容、验证结果和下一步请求。

**推断：** 未提交或未推送的本地改动不会被 Codex cloud 看到，因为云端从 GitHub 上的分支或 commit SHA 检出代码。因而“交接提交”是可靠接力的最小原子单位。

#### 3. 网页版 Codex cloud 接手

两种入口各有用途：

- **从 Codex cloud 网页发起：** 适合较长、独立、需要在后台运行的实现任务。选择正确仓库环境和明确的源分支/commit SHA，在提示中引用验收标准与验证命令。
- **从 PR 评论发起：** 适合已有 PR 上的定向续作，例如 `@codex fix the CI failures and run the commands listed in AGENTS.md`；PR 自带 diff、讨论和检查上下文。

任务提示建议包含：

```text
Base: <branch> @ <commit SHA>
Goal: <可观察结果>
Non-goals: <本次不要做的事项>
Constraints: Follow AGENTS.md; do not modify <范围>
Verify: <精确命令>
Deliver: Push only to this feature branch and summarize files/tests/risks
```

#### 4. 云端回传与 PR 审查

云端完成后先看 summary 与 diff，再决定继续追问或更新 PR。PR 描述至少保留：

- 变更目的与范围；
- 本地交接 SHA 和云端最终 SHA；
- 执行过的测试及结果；
- 尚未覆盖的风险；
- 截图/迁移说明（如适用）；
- 回滚路径。

随后运行 GitHub Actions，并用 `@codex review` 做额外高信号审查。机械失败交给 CI，架构、安全边界和行为回归交给人类与 Codex 语义审查。

#### 5. 合并前的本地最终复验

```bash
git fetch origin
git switch codex/<task-slug>
git pull --ff-only
git status
git diff --check origin/main...HEAD
<仓库声明的安装命令>
<lint>
<typecheck>
<test>
<build>
```

只在以下条件全部满足后合并：工作树干净、本地验证通过、Actions 通过、PR 对话已解决、Codex/人工发现的严重问题已处置、diff 与任务范围一致。

合并后本地同步：

```bash
git switch main
git pull --ff-only
<关键烟雾测试>
```

### 阶段 C：沉淀反馈

每次工作结束只沉淀能复用的事实：

- 某类失败能被确定性检测：加入测试或 CI；
- 某条仓库语义约束反复被审查指出：加入 `AGENTS.md` 的 Code Review Rules；
- 云端安装反复失败：修正 setup/maintenance script 并固定版本；
- 一次性的上下文：留在 Issue/PR，不污染长期 `AGENTS.md`。

## 四、并行与所有权规则

以下为**建议**：

1. 同一功能分支在同一时间只允许一个主动写入者（本地人/本地 Codex/云端 Codex 三者取一）。交接时先提交、推送，再声明所有权已转移。
2. 如果本地与云端必须并行，拆成互不重叠的分支，例如 `codex/<slug>-local` 与 `codex/<slug>-cloud`，最后通过第三个集成分支或 PR 合并，不让双方同时强推一个分支。
3. 每个云任务绑定一个明确 SHA。云端完成前，本地不要改写该分支历史；只追加提交。
4. 不允许代理直接向 `main` 写入；所有代理产出通过 PR 暴露 diff、检查和讨论。
5. 大任务拆成“可独立验证的小 PR”，避免本地与云端在长期分支上漂移。

**推断：** Git 能传递的是版本化文件和提交图，不能自动传递本地进程、未提交状态、IDE 临时设置或隐含意图。因此，稳定的混合工作流要把意图写入 PR，把规则写入 `AGENTS.md`，把依赖写入锁文件/云环境，把证据写入检查结果。

## 五、安全与权限建议

- 永不把 API key、密码、cookie、证书或 `.env` 提交到 Git。首次推送和每次 PR 前都检查 staged diff。
- Codex cloud 的 GitHub 访问只授权需要的仓库；云端环境仅配置完成任务所需的最小权限。
- setup secret 适合安装私有依赖等初始化步骤；不要假定 secret 会在代理运行阶段存在。
- 代理阶段联网保持默认关闭；确需联网时优先白名单/受限访问，并在 PR 记录原因。
- 保护 `main`，禁止强推和删除。代理只写功能分支，合并权保留给用户/团队。
- 个人仓库可把 Codex review 作为强烈建议的审查层，但仍以 CI 和本地复验为硬门禁。

## 六、个人仓库与团队仓库的门禁差异

### 个人仓库（推荐起步配置）

- 必须 PR；
- 必须通过 `quality`（或实际唯一 job 名）；
- 必须解决对话；
- 禁止强推和删除 `main`；
- 每个 PR 运行 `@codex review`；
- 合并前本地复验。

**推断：** 单人仓库如果要求“另一名人类批准”，可能让自己无法正常合并；Codex review 也不应被假定等同于 GitHub 分支保护所要求的人工批准。因此，个人阶段先把确定性门禁做硬，有真实合作者后再开启至少一名人工审批。

### 团队仓库（建议增强）

- 在上述基础上要求至少一名非最后推送者批准；
- 新提交后撤销过期审批或要求最近一次可审查推送再获批准；
- 敏感目录使用 CODEOWNERS；
- 高频合并时再评估 merge queue；
- 重要检查限定可信 GitHub App 来源。

## 七、不建议的做法

- 把普通网页聊天记录当作代码交接介质，而不创建 Git 提交或 PR；
- 本地存在未提交改动时直接要求云端“继续当前工作”；
- 让本地和云端同时改同一远端分支并改写历史；
- 把所有代码风格细节都写成 Codex 审查规则，而不配置 formatter/linter/CI；
- 因为有 `@codex review` 就跳过测试、分支保护或人工判断；
- 把生产 secret 暴露给代理或写进仓库；
- 在空仓库尚无 CI 成功记录时，先配置一个不存在的 required check 名称。

## 八、建议的最小验收演练

建立骨架后，用一个无风险任务演练整条链路：

1. 本地创建 `codex/handoff-smoke-test`，增加一个小函数及失败测试；
2. 提交并推送，建立 draft PR，记录交接 SHA；
3. 让 Codex cloud 补实现并运行 `AGENTS.md` 中的命令；
4. 检查云端 diff、Actions 与 `@codex review`；
5. 本地拉取 PR 分支，完整复验；
6. 满足门禁后合并；
7. 同步 `main` 并再跑烟雾测试；
8. 记录链路中唯一真实失败，把它沉淀到测试、环境脚本或 `AGENTS.md`。

这次演练通过，才能说明“本地 Codex → 云 GitHub → 网页 Codex cloud → PR → 本地验证”不是纸面流程，而是可复现工作流。

## 官方来源索引

- [OpenAI：Codex cloud](https://learn.chatgpt.com/docs/cloud)
- [OpenAI：Cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment)
- [OpenAI：Custom instructions with AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- [OpenAI：Review GitHub pull requests with Codex](https://learn.chatgpt.com/docs/third-party/github)
- [GitHub：GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow)
- [GitHub：Workflows](https://docs.github.com/en/actions/concepts/workflows-and-actions/workflows)
- [GitHub：About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [GitHub：Troubleshooting required status checks](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/troubleshooting-required-status-checks)
- [GitHub：Adding locally hosted code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)
