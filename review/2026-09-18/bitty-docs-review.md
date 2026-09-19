# Bitty 文档体系 Review 报告

我检查的是当前 `main`：

| 仓库                    | 当前 `main` | Markdown 数量 | 定位                                                   |
| --------------------- | --------- | ----------: | ---------------------------------------------------- |
| `bitty-docs`          | `f9ee022` |          81 | 全局 governance / decisions / security / project state |
| `bitty-terminal-docs` | `f150a31` |          99 | Terminal platform                                    |
| `bitty-plugins-docs`  | `a3cc069` |          66 | Plugin ecosystem                                     |
| `bitty-ai-docs`       | `6148f5d` |          75 | AI / Wheel architecture                              |

实际仓库名是 **`bitty-plugins-docs`**，不是单数 `bitty-plugin-docs`。

这次我不只是看 README，而是检查了四个仓库的完整 tree、topic indexes、跨仓库引用、submodule pin、最新 CI、metadata/documentation workflow，并重点深读了 Terminal / Plugin / AI 的核心 architecture、RFC、candidate、product/state 文档。

先给结论：

> **这轮重整整体方向是正确的，而且已经基本形成了一套成熟的 polyrepo documentation architecture。现在最大的问题已经不是“目录乱”，而是“重构后的同步与语义收敛还没完成”。**
>
> 尤其是 **Plugin docs 已经整理得非常好；Terminal docs 主体也比较稳定；真正需要下一轮集中清理的是 `bitty-docs` 的全局同步，以及 `bitty-ai-docs` 内多代设计同时存在造成的 semantic drift。**

我没有发现需要定义为 P0 的文档架构灾难，但有 **5 类 P1 问题**值得优先处理。

---

## 1. P1：`bitty-docs` 已经不是最新三个子文档仓库的真实聚合视图

这是目前最明确的结构问题。

`bitty-docs` 根目录的三个 gitlink 当前分别落后：

| Submodule        | `bitty-docs` pin | 当前 upstream `main` |             落后 |
| ---------------- | ---------------- | ------------------ | -------------: |
| `bitty-terminal` | `0b2fcfb`        | `f150a31`          | **17 commits** |
| `bitty-plugins`  | `ee19a0d`        | `a3cc069`          | **45 commits** |
| `bitty-ai`       | `39b4c75`        | `6148f5d`          | **77 commits** |

所以现在直接 clone `bitty-docs --recurse-submodules` 的人，看到的实际上不是你这几天重新整理后的结构，尤其 AI 和 Plugins 差距很大。

而且这不是未知问题：`bitty-docs/TODO.md` 自己已经记录了：

> bump the three root submodule pointers once the sibling alignment pull requests land

以及计划增加：

> `just docs-status`

这两个 TODO 现在正好应该执行。[bitty-docs TODO](https://github.com/bitty-terminal/bitty-docs/blob/main/TODO.md)

### 建议

这次 review 之后，可以直接做一个独立的 docs integration PR：

```text
bitty-docs
 ├─ bump bitty-terminal
 ├─ bump bitty-plugins
 ├─ bump bitty-ai
 ├─ repair cross-repo links
 ├─ refresh project-state
 └─ run corpus-wide integration checks
```

而且我很赞成 TODO 里的 `just docs-status`。它以后应该直接输出：

```text
terminal-docs
  aggregator pin : 0b2fcfb
  upstream main  : f150a31
  behind         : 17

plugins-docs
  ...
```

这会非常适合你现在的 polyrepo 模型。

---

# 2. P1：topic tree 重构以后，存在一批已经失效的跨仓库绝对链接

这是本轮重构产生的最典型 migration residue。

Plugin docs 已经从以前的：

```text
specifications/
```

拆成：

```text
runtime/
sdk/
packaging/
architecture/
specifications/
```

这个拆分本身很好。

但是其他仓库仍然大量引用旧位置。

例如 `bitty-docs/README.md` 还存在类似：

```text
bitty-plugins-docs/specifications/isolation-resource-rfc.md
bitty-plugins-docs/specifications/package-followup-rfc.md
bitty-plugins-docs/specifications/plugin-reuse-and-providers.md
```

现在实际位置分别是：

```text
runtime/isolation-resource-rfc.md
packaging/package-followup-rfc.md
packaging/plugin-reuse-and-providers.md
```

`bitty-terminal-docs/specifications/README.md` 更明显，仍然引用了多个旧路径：

| 旧路径                                               | 当前路径                                            |
| ------------------------------------------------- | ----------------------------------------------- |
| `specifications/plugin-api-v1-lua-surface-rfc.md` | `sdk/plugin-api-v1-lua-surface-rfc.md`          |
| `specifications/plugin-host-runtime-rfc.md`       | `runtime/plugin-host-runtime-rfc.md`            |
| `specifications/lua-runtime-rfc.md`               | `runtime/lua-runtime-rfc.md`                    |
| `specifications/isolation-resource-rfc.md`        | `runtime/isolation-resource-rfc.md`             |
| `specifications/package-lifecycle-rfc.md`         | `packaging/package-lifecycle-rfc.md`            |
| `specifications/package-followup-rfc.md`          | `packaging/package-followup-rfc.md`             |
| `specifications/plugin-reuse-and-providers.md`    | `packaging/plugin-reuse-and-providers.md`       |
| `specifications/ui-extensibility-architecture.md` | `architecture/ui-extensibility-architecture.md` |

AI 也有类似问题，例如 Terminal candidate 文档仍链接：

```text
bitty-ai-docs/specifications/ai-architecture.md
```

但现在是：

```text
bitty-ai-docs/architecture/ai-architecture.md
```

甚至新一些的 `terminal-platform-boundaries-candidate.md` 自己仍然链接到了 Plugin 的旧 `specifications/...` 路径。[Terminal Platform Boundaries](https://github.com/bitty-terminal/bitty-terminal-docs/blob/main/specifications/terminal-platform-boundaries-candidate.md)

### 为什么 CI 全绿却没发现？

这个原因我也查到了。

四个仓库的 `.github/scripts/check-docs.mjs` 都有类似逻辑：

```js
if (/^[a-z][a-z\d+.-]*:/i.test(destination) || destination.startsWith("//")) {
  return;
}
```

也就是说：

> **只要是 `https://...`，link validator 就直接跳过。**

所以 relative link 很严格，而恰恰你拆分 docs repo 后最重要的 **cross-repository absolute links 完全没有被验证**。

这也是为什么当前四个仓库最新 `Docs quality` workflow 都是 green，但仍存在这些问题。

### 建议

不要让每个 project docs repo 自己联网检查所有 sibling。

更合适的是把 **cross-repository integration validation 放到 `bitty-docs`**：

```text
project repo CI
    └── validate local corpus

bitty-docs CI
    └── validate the whole documentation graph
        ├── local links
        ├── cross-repo links
        ├── anchors
        ├── submodule pins
        ├── duplicate authority
        └── state freshness
```

这和 `bitty-docs` 的角色非常匹配。

可以增加：

```bash
just docs-check-local
just docs-check-cross-repo
just docs-status
just docs-integrity
```

---

# 3. P1：`project-state.json` 已经明显落后于真实 Bitty

这个比普通 stale prose 更严重，因为文件自己声明：

> `source_of_truth`

当前：

```json
snapshot_date = 2026-09-14
implementation.revision = bea338d
crates = 19
latest_release = v0.0.20
```

但是我同时检查了当前 `bitty/main`：

```text
main = 0e95b53
```

从 `bea338d` 到当前 `main`：

> **已经 ahead 137 commits**

当前 `Cargo.toml` 的 workspace members 也是：

> **21 crates**

不是 project-state 里的 19。

例如已经包含：

```text
bitty-panels
bitty-test-vm
```

事实上 `bitty-terminal-docs/architecture/README.md` 自己已经知道 live workspace 是 21 crates，但全局 `project-state.json` 仍然是 19。

所以现在已经形成：

```text
bitty actual implementation
        ↓
21 crates / 0e95b53

terminal docs
        ↓
部分已经知道 21 crates

bitty-docs project-state.json
        ↓
19 crates / bea338d
```

这是典型的 **single source of truth 反而成为 stale source of truth**。

[project-state.json](https://github.com/bitty-terminal/bitty-docs/blob/main/docs/project/project-state.json)

另外我也看到最近一次 scheduled `State freshness` workflow 是 failure；我不能仅凭结果断定失败原因就是这里，但结合现在实际已经落后 137 commits，freshness 本身确实需要处理。

### 建议

优先运行一次完整：

```text
state refresh
→ regenerate derived README/state summaries
→ compare against bitty/main
→ update 19 → 21
→ update revision
→ rerun state check
```

而且既然 Bitty 的开发速度这么快，单纯按日期 schedule 很容易落后。

以后可以考虑：

```text
bitty main changed
      ↓
repository_dispatch / workflow_call
      ↓
bitty-docs state freshness
```

或者至少把 “超过 N commits” 视为 stale，而不只是时间。

---

# 4. P1：生命周期/status 语言发生了明显矛盾

你的生命周期模型其实设计得很好：

```text
Draft
→ Experimental Implementation
→ Accepted
→ Verified
→ Compatible
→ Release-ready
```

真正的问题是一些老的 empty-state 文档仍然把：

> “尚未 Verified / Stable”

写成了：

> “尚未实现 / 尚未发布”

这已经不符合现实。

例如：

`bitty-docs/docs/development/README.md`：

> Bitty is currently documentation-first and pre-implementation.

但现在 Bitty 已经有大量 implementation。

`bitty-docs/docs/releases/README.md`：

> No Bitty product release exists.

但 GitHub 当前明确存在：

> `v0.0.20`，2026-09-11 发布，而且包含实际跨平台 assets。

[Bitty v0.0.20](https://github.com/bitty-terminal/bitty/releases/tag/v0.0.20)

Terminal docs 里还有：

```text
reference/README.md
    Bitty is pre-implementation

troubleshooting/README.md
    Bitty has not been implemented or released

user-guide/README.md
    There is no ... released executable

how-to/README.md
    Bitty has no released commands or workflows
```

与此同时，同一个仓库又有：

```text
Formal Release 0.0.1
Release Ladder
Release Distribution Matrix
v0.0.20
compatibility evidence
```

所以真正应该表达的是：

```text
Implemented       = yes
Experimental release exists = yes
Stable/supported interface   = no
Verified          = incomplete
Compatible        = not claimed
Release-ready     = not claimed
```

### 我建议统一一个非常重要的措辞原则

以后不要再用：

```text
not implemented
no release exists
pre-implementation
```

除非真的不存在代码。

改成：

```text
No stable/supported public contract exists yet.

Experimental implementation exists but is not yet Verified or
compatibility-guaranteed.

Pre-alpha releases exist, but no Stable/Compatible release is claimed.
```

这样会和你自己的 lifecycle 完全一致。

---

# 5. P1：`bitty-ai-docs` 现在存在“多代架构同时活着”的问题

四个仓库中，**AI docs 是现在最需要第二轮 semantic consolidation 的。**

目录整理其实已经不错：

```text
agent/
architecture/
context/
providers/
persistence/
interfaces/
integration/
product/
specifications/
```

问题不在 taxonomy，而在：

> 新设计不断修正旧设计，但旧设计仍然作为平级 Draft 存在。

于是产生了多个明确矛盾。

## Provider ownership

`architecture/ai-architecture.md` 的 MPC-5 仍然提出：

```text
Rust implements canonical wire adapters:
- openai_compatible
- anthropic_messages
- gemini_content
```

同时还说 Rust 负责 streaming / credential resolution 等。

但更近期的：

`providers/provider-plugin-boundary.md`

明确提出：

```text
Core:
    ModelProvider abstraction
    registry / routing / budgets / errors

Provider plugins:
    vendor HTTP integration
    local endpoints
    subscription adapters
    CLI adapters
    credentials behind opaque handles
```

这实际上已经是两种架构：

```text
A:
bitty-ai Rust Core
 ├─ OpenAI wire
 ├─ Anthropic wire
 └─ Gemini wire

B:
bitty-ai Core
 └─ ModelProvider trait
      ├─ provider-openai
      ├─ provider-anthropic
      ├─ provider-gemini
      ├─ provider-bedrock
      └─ provider-subscription/cli...
```

按照你现在整个 Bitty “small core + plugin” 的方向，B 显然是最近正在形成的新边界，但文档目前没有正式说：

```text
MPC-5 superseded by Provider Plugin Boundary
```

所以实现者会不知道听谁的。

[AI Architecture](https://github.com/bitty-terminal/bitty-ai-docs/blob/main/architecture/ai-architecture.md)
[Provider Plugin Boundary](https://github.com/bitty-terminal/bitty-ai-docs/blob/main/providers/provider-plugin-boundary.md)

---

# 6. P1/P2：Agent ↔ Panel 模型目前存在两套互相冲突的 Draft

`agent/agent-coordination.md` 明确说：

> reject mandatory one-headless-panel-per-agent

它采用：

```text
Agent → ExecutionContext ← Panel
```

并认为：

```text
Panel = projection/presentation
headed/headless = presentation property
Agent 可以没有 Panel
Execution 可以没有 Panel
多个 Panel 可以观察 Execution
```

这是一个比较一般化、边界清晰的模型。

但是：

`specifications/event-sourced-agent-workspace-candidate.md`

又写：

> agents act only in headless panels

以及：

> headed panels stay human-owned and agent touch means forking an execution snapshot

Terminal 对应 candidate 也复制了这个方向：

> Agents work only in headless panels.

这两套规则并不兼容。

### 我认为这里需要一个正式的 reconciliation

建议最终只留下类似：

```text
Agent != Panel
ExecutionContext != Panel

Agent
  └─ operates on ExecutionContext

Panel
  └─ projects / interacts with ExecutionContext

Headless / Headed
  └─ presentation mode, not authority model
```

至于：

```text
Agent 默认是否创建 headless Panel
```

可以是 Wheel policy。

但不应该升级成：

```text
Agent 必须存在于 headless Panel
```

这样的 Core invariant。

这实际上正好符合你现在逐渐形成的“机制与 policy 分离”。

---

# 7. P2：`Subagent` 旧术语还没有完全清掉

这在 AI 文档中有不少残留。

例如：

`ai-architecture.md`：

```text
Lua owns ... agent and subagent roles
```

`wheel-scope-and-framework-candidate.md`：

```text
Primary plus Subagents
```

`wheel-config-and-context-git-model-candidate.md`：

```text
subagent exploration as cheap branches
subagent isolation
```

`command-tool-architecture.md` 还有：

```text
bitty-ai-core ... Subagent
/plan -> subagent / mode
spawn subagents
```

但 `event-sourced-agent-workspace-candidate.md` 自己又明确：

```text
with no subagents
```

所以 AI corpus 内部现在已经同时存在：

```text
Main Agent + Subagents
```

和：

```text
all are Agents + graph relationships
```

两代模型。

这里我建议彻底把 `Subagent` 降级成：

> 外部 harness 的术语 / historical terminology

Wheel 自己统一：

```text
Agent
AgentRole
AgentRelation
Delegation
Caller
Coordinator
Reviewer
Worker
```

而不是建立 `Subagent` 类型。

---

# 8. P2：`.wheel` / `.agents` 的最新边界已经写对了，但旧草图仍会误导

`wheel-config-and-context-git-model-candidate.md` 这一部分我认为已经整理得很好：

```text
.agents
    = portable ecosystem capabilities
    = Skills / MCP

.wheel
    = Wheel-native behavior
    = filtering
    = rules
    = custom commands
    = custom tools
    = policy
```

并且明确说：

> `.wheel` references, filters, constrains, and composes what `.agents` exposes instead of re-storing skills or MCP material.

这个设计已经非常清楚。

但 `context/prompt-layering-design.md` 仍保留旧 source sketch：

```text
.wheel/
  config.lua
  instructions.md
  agents/
  skills/
  tools/
  mcp/
  hooks/
```

它虽然随后解释这是 sketch、canonical coverage 在其他文档，但读者看到这里还是很容易把 `.wheel/skills`、`.wheel/mcp` 当成候选结构。

### 建议

不要仅靠 qualifier。

直接改成：

```text
.wheel/
  init.lua
  instructions.md
  ...

.agents/
  skills/
  ...
  MCP compatibility material
```

或者干脆不重复 filesystem tree，只链接 canonical config document。

---

# 9. P2：Index completeness 还有少量遗漏

你的新规则是：

> root topic tree 应有 route-only `README.md`

这个思想很好，而且 Plugins 做得最完整。

我按目录直接 child Markdown 与 `README.md` 引用做了比对。

### `bitty-plugins-docs`

**没有发现 topic-tree direct child orphan。**

这是四个仓库里整理得最干净的。

### `bitty-terminal-docs`

两个真正值得补的：

```text
architecture/graphics-appearance.md
```

没有出现在：

```text
architecture/README.md
```

以及：

```text
specifications/text-compatibility.md
```

没有出现在：

```text
specifications/README.md
```

这两个实际上都有内容，尤其 `text-compatibility.md` 还包含 Unicode width / IME / terminfo 等不少实现证据，不该成为 hidden document。

### `bitty-ai-docs`

问题明显一些。

`architecture/README.md` 目前本质上是：

> Architecture Diagrams

而不是 architecture topic 的完整 route index。

因此这些同目录文档没有被它索引：

```text
ai-architecture.md
code-intelligence-sharing-r4.md
command-tool-architecture.md
context-retention-r3.md
host-boundary-trait-design.md
persistence-profile-r6.md
prototype-promotion-checklist.md
tool-transport-r2.md
```

这是对你自己的：

> every root topic tree has a route-only index

规则的直接破坏。

### 最好的调整

我建议：

```text
architecture/
├── README.md              # 真正的 architecture route index
├── ai-architecture.md
├── command-tool-architecture.md
├── ...
└── diagrams/
    ├── README.md
    ├── glossary.yaml
    ├── d2/
    └── final/
```

这样最干净。

---

# 10. P2：Plugin docs 有一个类似但较轻的双 architecture namespace

目前 Plugins 有：

```text
architecture/
```

作为真正的 canonical architecture topic tree；

同时：

```text
docs/architecture/
```

用于 Mermaid/SVG/glossary diagram suite。

逻辑上说得通，但对 contributor 很容易产生：

> “architecture 到底在哪？”

的问题。

Terminal 也有 diagram concept，但 Plugins 这个结构特别容易混淆。

我会建议和 AI 一样最终统一到：

```text
architecture/
├── README.md
├── plugin-ecosystem-model.md
├── plugin-ipc-boundary.md
├── ui-extensibility-architecture.md
└── diagrams/
```

而把 `docs/` 严格留给：

```text
documentation workflow
handoff
todo
repo process
```

这样你所定义的：

> canonical content = root topic trees
> repository process = docs/

就没有例外了。

---

# 11. P2：AI 的 `ai-architecture.md` 已经过于巨大，逐渐变成“第二个总仓库”

当前几个最大文档：

| 文档                                              |        大约大小 |
| ----------------------------------------------- | ----------: |
| `bitty-docs/decisions/open-questions.md`        |     139 KiB |
| `bitty-ai-docs/architecture/ai-architecture.md` | **133 KiB** |
| `bitty-docs/decisions/index.md`                 |     121 KiB |
| `bitty-docs/roadmap/now-next-later.md`          |     102 KiB |
| `bitty-ai-docs/ipc-agent-rfc.md`                |      84 KiB |

Register 很大是合理的。

但 `ai-architecture.md` 达到 133 KiB，同时下面已经有：

```text
context/
providers/
agent/
persistence/
interfaces/
integration/
```

那么它就不应该再保存所有细节。

目前它还在定义：

```text
ModelProvider
ContextProvider
Agent
Tool Bus
provider config
wire protocols
context budgets
...
```

这就是产生前面那些 semantic conflict 的主要原因。

### 建议把 `AI Architecture` 降成真正的 architecture spine

只留下：

```text
Goals / Non-goals
Layer model
Ownership
Trust boundaries
Major dataflow
Core invariants
Accepted authority
Topic routing
```

例如：

```text
AI Architecture
│
├── Provider → providers/
├── Context → context/
├── Agent → agent/
├── Persistence → persistence/
├── Host interfaces → interfaces/
├── Cross-repo → integration/
└── versioned contract → specifications/
```

详细 API、budget、provider wire、storage profile 不再复制。

这会大幅减少未来 semantic drift。

---

# 12. P2：Terminal 的 empty-state taxonomy 是好的，但公开网站可能太“空”

这些目录：

```text
examples/
how-to/
migrations/
requirements/
troubleshooting/
tutorials/
user-guide/
```

现在很多只有 README。

而且写得其实很好：都有 admission criteria，并且明确不伪造未稳定功能。

我认为保留目录没有问题。

但多数：

```yaml
website_publish: true
```

如果 `bitty.run` 直接把这些全部放到导航，就会出现很多：

> No tutorial available
> No troubleshooting available
> No examples available

这在 repo corpus 里是“严谨”，在产品 docs 网站里却会增加大量 empty navigation。

### 建议

Repo 继续保留这些 README。

Website consumer 则做：

```text
if index has zero publishable children:
    hide from public sidebar
```

或者 temporarily：

```yaml
website_publish: false
```

等第一个真实 document 出现再打开。

---

# 13. P3：`repo.toml` 四个仓库仍然都是 `owner = "TBD"`

四个：

```text
bitty-docs
bitty-terminal-docs
bitty-plugins-docs
bitty-ai-docs
```

全部：

```toml
owner = "TBD"
```

这不算 bug，因为 ADR 0011 本来就记录了 ownership / CODEOWNERS 还需要 owner decision。

但既然 `repo.toml` 是 machine-readable workspace identity，我建议最终不要长期保留 `TBD`。

可以以后统一成 GitHub team：

```toml
owner = "bitty-terminal/docs"
```

或者：

```toml
owner = "@Xuepoo"
```

取决于未来组织结构。

---

# 各仓库单独结论

| Repo                    | 当前评价                               | 最需要处理                                                                       |
| ----------------------- | ---------------------------------- | --------------------------------------------------------------------------- |
| **bitty-docs**          | 架构职责已经很清楚，但 integration truth 已滞后  | project-state、submodule pins、cross-repo links、旧 release wording             |
| **bitty-terminal-docs** | 整体稳定，topic taxonomy 很完整            | old links、status wording、两个 orphan docs                                     |
| **bitty-plugins-docs**  | **四个里面目前整理得最干净**                   | diagram 路径统一、等待 sibling links 跟上                                            |
| **bitty-ai-docs**       | 内容最有价值、设计发展最快，同时 semantic drift 最大 | provider ownership、Panel model、Subagent、`.wheel/.agents`、AI Architecture 拆分 |

特别是 Plugins 这次：

```text
runtime/
sdk/
packaging/
architecture/
specifications/
product/
extensibility/
```

我认为比以前所有东西堆在 `specifications/` 里明显更合理。

你现在已经把“文档类型”进一步提升成了“领域 ownership”。

这一步是正确的。

---

# 我建议下一轮按这个顺序做

1. **先 refresh `project-state.json`**：`bea338d → 0e95b53`，19 crates → 21 crates，并重新验证 State freshness。
2. **更新 `bitty-docs` 三个 submodule pin**，让 aggregator 真正反映最近的重构。
3. **全局修复 cross-repo moved links**，尤其 `specifications/ → runtime/sdk/packaging/architecture` 以及 AI 的 `specifications/ → architecture/interfaces/...`。
4. **给 `bitty-docs` 加 corpus-level cross-repo link checker**；现在四个仓库的 checker 都会跳过 HTTPS。
5. **统一 release/status vocabulary**：有 Experimental/Pre-alpha release，但没有 Stable/Compatible/Release-ready release；不要再写 “not implemented”。
6. **集中做一次 `bitty-ai-docs` semantic convergence**：Provider、Agent/Panel、Subagent、`.wheel/.agents` 四项一起解决。
7. **把 AI `architecture/README.md` 改回真正的 route index**，diagram 下沉到 `architecture/diagrams/`；顺手补 Terminal 两个 orphan route。
8. **逐步瘦身 `ai-architecture.md`**，让它成为 architecture spine，而不是另一个 133 KiB 的总规范。

---

## 最后的整体判断

这轮整理之后，我认为 Bitty 文档体系已经从：

```text
一个项目的大型 docs/
```

真正变成：

```text
                         bitty-docs
                  shared governance plane
                           │
          ┌────────────────┼────────────────┐
          │                │                │
 terminal-docs       plugins-docs        ai-docs
 Terminal domain     Plugin domain       AI/Wheel domain
          │                │                │
          └────────────────┴────────────────┘
                    implementation repos
```

**这个顶层划分我不建议再大改。**

下一阶段重点也不应该继续“重新分类目录”，而应该转向两个词：

> **Synchronization + Convergence**

即：

* 让跨仓库链接、submodule、project state 永远同步；
* 让新设计正式 supersede / reconcile 旧设计，而不是无限累计平级 Draft。

如果这两件事完成，你这套 docs architecture 就会从“组织得很好”进一步变成一种真正可以长期维护的 **documentation system**，而不仅是一批 Markdown 文件。
