---
name: sharedev-init
description: APL 开发环境自动化准备和项目初始化技能。支持 Trae IDE、Claude Code、Codex 三种目标平台，区分交互式和非交互式执行模式。
---

# APL 开发环境自动化准备技能

这个技能用于初始化 ShareDev APL/PWC 工作区，并把对应平台的 spec/skills 接入到当前开发环境。

## 目标

完成以下事情：

1. 检查并安装 `sharedev` CLI
2. 读取项目根目录 `settings.json`
3. 初始化 `<enterpriseRoot>/` 工作区
4. 拉取 docs、PWC、APL、dev-metadata、object-dev
5. 初始化目标平台的 spec 模板
6. 安装目标平台需要的 skills/specs
7. 修复已安装旧技能中的遗留路径引用
8. 验证最终目录结构

## 先决规则

**必须遵守以下规则：**

1. 先决定 `TARGET_PLATFORM`，再执行任何平台相关命令。
2. 先决定 `EXECUTION_MODE`，再决定是否允许交互命令。
3. 不要通过 `~/.codex`、`~/.claude`、`.trae` 是否存在来“猜”本次目标平台。
4. 对代理环境优先使用非交互命令；只有用户明确要求手动交互，且当前终端真的是用户可操作终端时，才走交互流程。
5. 不要把“初始化项目”和“安装技能”混成一个不可拆分的黑盒；执行时要分成两个阶段汇报。
6. 不要在安装后使用大规模 `sed`/字符串替换去篡改 skills markdown 路径。路径问题应通过模板变量或稳定引用方案解决。
7. 不要要求 Codex 额外执行 `codex --enable skills`。当前文档不应把它当作必需步骤。
8. 必须执行受控兼容修复，但只限已安装的 `sharedev-apl-implement`、`sharedev-apl-code-review` 两个技能，且只修复已知 legacy 路径模式。
9. 任何关键命令返回非零退出码时，都不能直接口头视为“整体成功”；必须根据该步骤的验证条件重新判定 `completed|incomplete|blocked`。
10. 只有“命令已执行”且“该步骤的验证条件满足”时，才能把该步骤标记为 `completed`；否则必须明确标记为 `incomplete` 或 `blocked`。
11. 只要 docs、PWC、APL、dev-metadata、object-dev 任一必需资源尚未核实完成，就不得对外宣称“初始化完成”。
12. Step 8.5 的兼容修复只能按文档列出的 legacy 模式做定向替换，不能用“改写措辞”“弱化表述”来替代修复。
13. 最终汇报必须按 Step 1 到 Step 10 逐步给出状态，不能只给笼统结论。

## 输入参数

### 必须输入

从项目根目录 `settings.json` 中读取：

- `enterpriseEA`
- `domain`
- `certificate`

### 运行时变量

执行前必须区分“工作区根目录”“企业工程名”“企业工程目录”，不能混用：

| 变量 | 含义 |
|---|---|
| `<workspaceRoot>` | 当前项目根目录，即 `settings.json` 所在目录 |
| `<enterpriseEAName>` | `settings.json` 中的 `enterpriseEA` 字段值，例如 `fktest8507`、`Project` |
| `<enterpriseRoot>` | 企业工程目录，默认是 `<workspaceRoot>/<enterpriseEAName>`；如果当前目录本身已经包含 `.sharedev/`、`package/`、`pwc/`，则当前目录就是 `<enterpriseRoot>` |

关键规则：

1. `sharedev init -e ...` 里的 `-e` 参数是企业工程名或相对目标目录，不是绝对工程根路径。
2. 如果当前已经位于 `<enterpriseRoot>`，刷新时必须使用 `-e .`，不能再传 `<enterpriseEAName>`，否则会在当前目录下再次创建一个同名子目录。
3. 只有在 `<workspaceRoot>` 执行首次初始化时，才使用 `-e <enterpriseEAName>`。

### 必须确定

在执行前，必须明确以下三个变量：

- `TARGET_PLATFORM=trae|claude|codex`
- `EXECUTION_MODE=interactive|noninteractive`
- `INSTALL_SCOPE=project|global`

## 平台矩阵

| TARGET_PLATFORM | EXECUTION_MODE | INSTALL_SCOPE | spec 目录 | skills 安装位置 |
|---|---|---|---|---|
| `trae` | `interactive` 优先 | `project` | `<enterpriseRoot>/.trae/` | `<workspaceRoot>/.trae/skills/` |
| `trae` | `noninteractive` | `project` | `<enterpriseRoot>/.trae/` | `<workspaceRoot>/.trae/skills/` |
| `claude` | `interactive` 或 `noninteractive` | `project` | `<enterpriseRoot>/.claude/` | `<workspaceRoot>/.claude/skills/` |
| `claude` | `noninteractive` 推荐 | `global` | `<enterpriseRoot>/.claude/` | `~/.claude/skills/` |
| `codex` | `noninteractive` 推荐 | `global` | `<enterpriseRoot>/.codex/` | `~/.codex/skills/` |

## 推荐默认值

如果用户没有明确指定，使用以下默认规则：

- Trae IDE，且当前终端是用户可直接操作的前台终端: `TARGET_PLATFORM=trae`，`EXECUTION_MODE=interactive`，`INSTALL_SCOPE=project`
- Trae agent / Trae 代理执行 / 共享受限终端: `TARGET_PLATFORM=trae`，`EXECUTION_MODE=noninteractive`，`INSTALL_SCOPE=project`
- Claude Code: `TARGET_PLATFORM=claude`，`EXECUTION_MODE=noninteractive`，`INSTALL_SCOPE=project`
- Codex: `TARGET_PLATFORM=codex`，`EXECUTION_MODE=noninteractive`，`INSTALL_SCOPE=global`

## 执行状态约定

执行过程中，每一步只能落在以下三种状态之一：

- `completed`：该步命令已执行，且文档中的验证条件全部满足。
- `incomplete`：该步部分执行过，但验证条件仍有缺口，必须继续补齐，不能宣称完成。
- `blocked`：由于缺参、命令失败、目录缺失、交互受限或远端数据异常，当前无法安全继续，必须明确汇报阻断原因。

严格要求：

1. 不允许把 `incomplete` 口头描述成“基本完成”“主体完成”或“可视为成功”。
2. 任一步骤一旦是 `blocked`，除非文档明确允许继续做独立收尾，否则不得继续宣称后续流程已完整完成。
3. Step 10 输出时，必须逐步复述每一步的最终状态。

## 执行流程

### Phase A: Workspace Bootstrap

#### Step 1: 检查 sharedev CLI

执行：

```bash
which sharedev
```

如果未安装，执行：

```bash
npm install -g @share-crm/sharedev-cli
```

验证：

- 返回 `sharedev` 可执行路径

#### Step 2: 读取 settings.json

读取项目根目录 `settings.json`，提取：

- `enterpriseEA`（保存到 `<enterpriseEAName>`）
- `domain`
- `certificate`

缺任一项则中止，并提示用户补齐。

随后立即判断：

- 如果当前目录已经存在 `.sharedev/`、`package/`、`pwc/`，则当前目录就是 `<enterpriseRoot>`
- 否则 `<enterpriseRoot> = <workspaceRoot>/<enterpriseEAName>`

#### Step 3: 确定执行模式

先显式确定本次执行的目标：

```text
TARGET_PLATFORM = trae | claude | codex
EXECUTION_MODE = interactive | noninteractive
INSTALL_SCOPE = project | global
```

判定规则：

1. 用户明确指定时，以用户指定为准。
2. 当前代理环境已知时，可按“推荐默认值”选取。
3. 不允许仅根据本机目录存在性反推目标平台。
4. 若当前环境是 Trae agent，而不是用户亲自操作的共享前台终端，则 `EXECUTION_MODE` 必须选择 `noninteractive`，不能默认走 `interactive`。

#### Step 4: 初始化项目

初始化或刷新时，先判断当前处于哪一层目录：

- 首次初始化：当前位于 `<workspaceRoot>`，且 `<enterpriseRoot>` 尚不存在
- 刷新已有工程：当前位于 `<enterpriseRoot>`，或 `<enterpriseRoot>` 已存在

禁止把 `<enterpriseEAName>` 当成“当前工程根目录路径”直接拼接进路径判断。

**交互模式：**

```bash
cd <workspaceRoot>
sharedev init -e <enterpriseEAName> -d <domain> -c <certificate>
```

如果已经位于 `<enterpriseRoot>` 目录内，优先使用：

```bash
cd <enterpriseRoot>
sharedev init -e . -d <domain> -c <certificate>
```

仅当以下条件同时满足时才允许交互：

- 用户明确希望手动选择
- 当前终端确实是用户可直接操作的前台终端

否则必须切换为非交互模式。

**非交互模式：**

```bash
cd <workspaceRoot>
sharedev init --yes --agent <TARGET_PLATFORM> --pull-all-apl --pull-all-pwc -e <enterpriseEAName> -d <domain> -c <certificate>
```

如果已经 `cd <enterpriseRoot>`，统一改为 `-e .`，避免再次创建 `<enterpriseEAName>/` 子目录：

```bash
cd <enterpriseRoot>
sharedev init --yes --agent <TARGET_PLATFORM> --pull-all-apl --pull-all-pwc -e . -d <domain> -c <certificate>
```

说明：

- `--agent` 必须与 `TARGET_PLATFORM` 一致
- 这条命令优先用于 Claude Code / Codex / 代理执行场景
- 这条命令可能已经顺带完成 docs / pwc / apl / metadata / spec 的部分初始化
- **⚠️ 注意：`init --yes` 不会拉取 `object-dev` 资源，必须在 Step 5 中单独执行 `sharedev object-dev object pull --all`**

验证：

- `<enterpriseRoot>/` 已创建
- `<enterpriseRoot>/.sharedev/settings.json` 已创建
- `<enterpriseRoot>/.git/` 已创建

命令结果判定：

1. 如果 `sharedev init ...` 返回码为 `0`，按正常流程进入 Step 5。
2. 如果 `sharedev init ...` 返回非零退出码，不得直接把 Step 4 记为 `completed`。
3. 返回非零退出码时，必须立即检查上述三个基础产物是否都已存在：
   - 若缺任一项，Step 4 直接记为 `blocked`，并停止把本次流程宣称为成功初始化。
   - 若三项都存在，Step 4 只能记为 `incomplete`，随后必须进入 Step 5 对 docs、PWC、APL、dev-metadata、object-dev 做逐项补齐与复核。
4. 只有在 Step 5 的缺失项都被补齐，且不存在未解释的失败项后，才能把整个 Phase A 视为完成。

**错误示例（禁止）**：

如果当前已经在 `/path/to/ShareCRM01/Project`，再执行：

```bash
sharedev init --yes --agent trae --pull-all-apl --pull-all-pwc -e Project -d <domain> -c <certificate>
```

就会得到 `/path/to/ShareCRM01/Project/Project`。这是错误用法。

#### Step 5: 补齐文档和元数据

**重要说明：**
- `sharedev init --yes` **不会**自动拉取 `object-dev` 资源，必须单独执行
- 即使 `sharedev init --yes` 返回成功，也必须逐项检查并补齐缺失资源

**执行流程（必须逐项检查并执行）：**

进入 `<enterpriseRoot>` 后，按以下顺序检查并补齐每项资源：

**1. 检查并补齐 docs**
```bash
cd <enterpriseRoot>
# 检查是否存在
ls .sharedev/docs/apl/ .sharedev/docs/pwc/ >/dev/null 2>&1
# 如不存在，执行：
sharedev docs download
```

**2. 检查并补齐 dev-metadata**
```bash
# 检查是否存在
ls .sharedev/dev-metadata/objects/ >/dev/null 2>&1
# 如不存在，执行：
sharedev dev-metadata download --all
```

**3. 检查并补齐 object-dev（⚠️ 重要：init 不会自动拉取）**

**⚠️ 关键区分**：
- `dev-metadata` 生成 `.sharedev/dev-metadata/objects/`（Markdown文档）
- `object-dev` 生成 `tenant-config/objects/` 或 `object-dev/objects/`（XML定义文件）
- **两者是不同的资源，不能混淆！**

```bash
# 检查 object-dev 是否存在（注意：不是 .sharedev/dev-metadata/objects/）
ls tenant-config/objects/ >/dev/null 2>&1 \
  || ls object-dev/objects/ >/dev/null 2>&1
# 如不存在，必须执行：
sharedev object-dev object pull --all
```

说明：
- 此命令拉取所有对象的定义文件（对象、字段、布局、布局规则等）
- 输出目录通常是 `tenant-config/objects/` 或 `object-dev/objects/`
- 如果只需要自定义对象，可使用 `sharedev object-dev object pull --custom`
- 如果只需要特定包的对象，可使用 `sharedev object-dev object pull --package`

**4. 检查并补齐 PWC 组件**
```bash
# 检查是否存在
ls pwc/components/ >/dev/null 2>&1
# 如不存在，执行：
sharedev pwc pull --all --type component
```

**5. 检查并补齐 APL 函数**
```bash
# 检查是否存在
ls package/fx/custom/apl/script/ >/dev/null 2>&1
# 如不存在，执行：
sharedev apl pull --all
```

**执行要求：**

1. 进入 Step 5 时，必须为以下资源分别建一个状态：`docs`、`pwc_components`、`apl`、`dev_metadata`、`object_dev`。
2. **必须逐项检查**，发现缺失立即执行对应补齐命令，不能假设 `sharedev init --yes` 已拉取所有资源。
3. 只要某项资源的验证条件尚未满足，该资源状态就必须保持 `incomplete` 或 `blocked`，不能因为其他目录已存在而顺带算完成。
4. 如果某条补齐命令返回非零退出码，必须立即重新核对该资源对应产物：
   - 产物缺失：该资源记为 `blocked`
   - 产物已存在但是否最新无法确认：该资源记为 `incomplete`
5. 如果 `sharedev apl pull --all` 返回 "No APL function details were returned" 或其他"未返回函数详情"的结果，而 `<enterpriseRoot>/package/fx/custom/apl/script/` 仍不存在，则 `apl` 必须记为 `incomplete` 或 `blocked`，不得算成功。
6. `object_dev` 的完成判定必须满足以下任一目录存在（⚠️ 注意：不是 `.sharedev/dev-metadata/objects/`）：
   - `<enterpriseRoot>/tenant-config/objects/`（object-dev 默认输出目录）
   - `<enterpriseRoot>/object-dev/objects/`（object-dev 备选输出目录）
7. Step 5 结束时，必须明确列出每项资源状态；只要有一项不是 `completed`，整个初始化都不能对外宣称完成。

验证：

- `<enterpriseRoot>/.sharedev/docs/apl/`
- `<enterpriseRoot>/.sharedev/docs/pwc/`
- `<enterpriseRoot>/.sharedev/dev-metadata/`
- `sharedev pwc pull --all --type component` 若返回资源，则应生成 `<enterpriseRoot>/pwc/components/`
- `sharedev apl pull --all` 若返回函数详情，则应生成 `<enterpriseRoot>/package/fx/custom/apl/script/`
- object-dev 对象定义目录至少存在一处（XML格式）：
  - `<enterpriseRoot>/tenant-config/objects/`（object-dev 默认输出目录）
  - `<enterpriseRoot>/object-dev/objects/`（object-dev 备选输出目录）
- dev-metadata 对象文档目录（Markdown格式）：
  - `<enterpriseRoot>/.sharedev/dev-metadata/objects/`

### Phase B: Agent Integration

#### Step 6: 初始化 spec 模板

**交互模式：**

```bash
cd <enterpriseRoot>
sharedev spec init
```

适用条件：

- 用户要手动选平台
- 当前终端是用户可直接操作的终端

**非交互模式：**

优先复用 `sharedev init --yes --agent <TARGET_PLATFORM>` 已生成的 spec 目录，不再额外执行 `sharedev spec init`。

如果非交互初始化后仍缺失 spec 目录，则把它标记为阻断问题并汇报，不要假装步骤已完成。

验证：

- `<enterpriseRoot>/.trae/` 或 `<enterpriseRoot>/.claude/` 或 `<enterpriseRoot>/.codex/`
- `<spec-dir>/skills/`
- `<spec-dir>/specs/`

#### Step 7: 安装 specs

无论目标平台是什么，都把 `<enterpriseRoot>/<spec-dir>/specs/` 同步到：

```bash
<enterpriseRoot>/.sharedev/specs/
```

推荐：

```bash
mkdir -p <enterpriseRoot>/.sharedev/specs
rsync -a <enterpriseRoot>/<spec-dir>/specs/ <enterpriseRoot>/.sharedev/specs/
```

这是强制步骤，不能因为 `<enterpriseRoot>/.sharedev/specs/` 已存在就跳过。
只要 `<enterpriseRoot>/<spec-dir>/specs/` 存在，就必须执行一次同步，把平台 spec 的最新内容写入 `<enterpriseRoot>/.sharedev/specs/`。

#### Step 8: 安装 skills

按平台安装，不做环境猜测。

这里必须区分“来源目录”和“安装目标目录”：

- Trae 的来源目录：`<enterpriseRoot>/.trae/skills/`
- Trae 的安装目标：`<workspaceRoot>/.trae/skills/`
- Step 8 的含义是把企业工程内已生成的 skills 安装到当前工作区可直接使用的位置，不能把来源目录误当成安装已完成

**Trae 项目级：**

```bash
mkdir -p <workspaceRoot>/.trae/skills
rsync -a <enterpriseRoot>/<spec-dir>/skills/ <workspaceRoot>/.trae/skills/
```

**Claude 项目级：**

在项目根目录（`<workspaceRoot>`）下执行：

```bash
mkdir -p .claude/skills
rsync -a <enterpriseRoot>/<spec-dir>/skills/ .claude/skills/
```

**Claude 全局：**

```bash
mkdir -p ~/.claude/skills
rsync -a <enterpriseRoot>/<spec-dir>/skills/ ~/.claude/skills/
```

**Codex 全局：**

```bash
mkdir -p ~/.codex/skills
rsync -a <enterpriseRoot>/<spec-dir>/skills/ ~/.codex/skills/
```

验证：

- 目标 skills 目录存在
- 每个技能目录包含 `SKILL.md`
- 对于 Trae 项目级：验证 `<enterpriseRoot>/.trae/skills/` 作为来源目录存在，且 `<workspaceRoot>/.trae/skills/` 作为安装目标目录存在并包含技能

#### Step 8.5: 已安装技能兼容修复

这一步用于兼容已经安装过的旧版本技能。

只处理以下两个技能：

- `sharedev-apl-implement`
- `sharedev-apl-code-review`

兼容修复触发条件：

1. 这两个技能已经安装在目标 skills 目录里
2. 技能 markdown 中仍然引用旧路径
3. 当前项目的文档/对象元数据已经位于 `<enterpriseRoot>/` 目录下

执行原则：

- 只要满足上述触发条件，就必须执行兼容修复
- 不允许只做扫描、报告问题后结束
- 不允许把“发现旧路径但未修复”视为初始化成功
- 不允许把命中的旧路径改写成模糊描述来规避替换；命中 legacy 字面量时，必须替换成文档指定的新表达

新的标准路径是：

- docs 根目录：`<enterpriseEA>/.sharedev/docs/`
- objects 目录：`<enterpriseEA>/.sharedev/dev-metadata/objects/`

旧路径到新路径的明确映射如下：

| 旧路径/旧写法 | 必须替换为 |
|---|---|
| `<enterpriseEA>/.sharedev/docs/` | `<enterpriseEA>/.sharedev/docs/` |
| `<enterpriseEA>/.sharedev/docs/` | `<enterpriseEA>/.sharedev/docs/` |
| `<enterpriseEA>/.sharedev/dev-metadata/objects/` | `<enterpriseEA>/.sharedev/dev-metadata/objects/` |
| `<enterpriseEA>/.sharedev/dev-metadata/objects/` | `<enterpriseEA>/.sharedev/dev-metadata/objects/` |
| `` `<enterpriseEA>/.sharedev/dev-metadata/objects/` `` | `` `<enterpriseEA>/.sharedev/dev-metadata/objects/` `` |
| `查看 \`.sharedev/object/\` 目录下以 \`__c\` 结尾的文件` | `查看 \`<enterpriseEA>/.sharedev/dev-metadata/objects/\` 目录下以 \`__c\` 结尾的文件` |
| `[对象索引(动态更新)](<enterpriseEA>/.sharedev/dev-metadata/objects/objects.md)` | `[对象索引(动态更新)](<enterpriseEA>/.sharedev/dev-metadata/objects/objects.md)` |

常见旧路径包括：

- `<enterpriseEA>/.sharedev/docs/`
- `<enterpriseEA>/.sharedev/docs/`
- `<enterpriseEA>/.sharedev/dev-metadata/objects/`
- `<enterpriseEA>/.sharedev/dev-metadata/objects/`
- `[对象文档](<enterpriseEA>/.sharedev/dev-metadata/objects/)`
- `[对象索引(动态更新)](<enterpriseEA>/.sharedev/dev-metadata/objects/objects.md)`
- `**自定义对象导航** (查看 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 目录下以 `__c` 结尾的文件)`

修复规则：

**占位变量规则：**

- 兼容修复后的技能 markdown 中，必须保留 `<enterpriseEA>` 作为占位变量
- 不允许把 `<enterpriseEA>` 渲染成具体工程名，例如 `fktest8507`
- 不允许把 `<workspace>`、`ENTERPRISE_EA`、绝对路径拼接结果直接写回被修复技能源码
- Agent 在运行技能时，应根据当前上下文自行解析 `<enterpriseEA>`，而不是依赖硬编码目录名

**修复后的目标写法：**

- docs 统一写成 `<enterpriseEA>/.sharedev/docs/`
- objects 统一写成 `<enterpriseEA>/.sharedev/dev-metadata/objects/`

这里的要求是强制性的：

- 只要看到历史相对路径或 `<enterpriseEA>/.sharedev/dev-metadata/objects/`，后续兼容流程里就必须把它理解为“旧路径”
- 后续兼容的统一目标位置就是 `<enterpriseEA>/.sharedev/dev-metadata/objects/`
- `<enterpriseEA>/.sharedev/dev-metadata/objects/` 本身是有效目标，不是旧路径
- 不允许在被修复技能里出现具体工程名硬编码，例如 `fktest8507`

执行要求：

1. 先确定本次实际 `SKILL_ROOT`
2. 仅扫描上述两个技能目录中的 `*.md`
3. 先扫描并记录命中的 legacy 路径文件
4. 只要命中 legacy 路径，必须立刻执行替换
5. 仅替换 docs/object 两类已知旧路径
6. 替换时写回占位变量形式，不写回真实工程名
7. 替换完成后，重新扫描并验证旧路径已被清理
8. 最后验证目标目录真实存在
9. 不允许把 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 这类命中的旧字面量改写成“历史对象目录”“旧目录”等模糊说法来绕过校验；必须按映射表写成 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 或与其一致的标准表达

示例：

```bash
SKILL_ROOT="<resolved-skill-root>"

for skill_name in sharedev-apl-implement sharedev-apl-code-review; do
  skill_path="$SKILL_ROOT/$skill_name"
  [ -d "$skill_path" ] || continue

  matched_files=$(grep -RIl "\.\./\.\./\.\./\.sharedev/docs/\|\.\./\.\./\.\./[^/]\+/\.sharedev/docs/\|\.\./\.\./\.\./\.sharedev/object/\|`\.sharedev/object/`\|查看 `\.sharedev/object/` 目录下以 `__c` 结尾的文件" "$skill_path" --include="*.md" || true)
  [ -z "$matched_files" ] && continue

  echo "$matched_files" | while read -r file; do
    [ -n "$file" ] || continue
    perl -0pi -e "s#\\.\\./\\.\\./\\.\\./[^/]+/\\.sharedev/dev-metadata/objects/#<enterpriseEA>/.sharedev/dev-metadata/objects/#g" "$file"
    perl -0pi -e "s#\\.\\./\\.\\./\\.\\./[^/]+/\\.sharedev/docs/#<enterpriseEA>/.sharedev/docs/#g" "$file"
    perl -0pi -e "s#\\.\\./\\.\\./\\.\\./\\.sharedev/docs/#<enterpriseEA>/.sharedev/docs/#g" "$file"
    perl -0pi -e "s#\\.\\./\\.\\./\\.\\./\\.sharedev/object/#<enterpriseEA>/.sharedev/dev-metadata/objects/#g" "$file"
    perl -0pi -e "s#`\\.sharedev/object/`#`<enterpriseEA>/.sharedev/dev-metadata/objects/`#g" "$file"
    perl -0pi -e "s#查看 `\\.sharedev/object/` 目录下以 `__c` 结尾的文件#查看 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 目录下以 `__c` 结尾的文件#g" "$file"
  done

  if grep -RIn "\.\./\.\./\.\./\.sharedev/object/\|\.\./\.\./\.\./[^/]\+/\.sharedev/docs/\|`\.sharedev/object/`" "$skill_path" --include="*.md"; then
    echo "compat-fix failed for $skill_name"
    exit 1
  fi
done
```

注意：

- 上述兼容修复示例写回的是占位变量 `<enterpriseEA>`，不是实际项目名
- 无论项目级还是全局安装，都不要把目标技能 markdown 改写成绝对路径
- 运行期解析可以依赖当前 workspace 和当前企业工程上下文，但技能源码必须保持变量化

验证：

- `<enterpriseEA>/.sharedev/docs/` 存在
- `<enterpriseEA>/.sharedev/dev-metadata/objects/` 存在，或 CLI 已把对象定义拉取到 `<enterpriseEA>/tenant-config/objects/`
- 若命中过旧路径，兼容修复命令已实际执行
- 已安装的 `sharedev-apl-implement` / `sharedev-apl-code-review` 中不再残留旧的相对 docs/object 路径
- 已安装的 `sharedev-apl-implement` / `sharedev-apl-code-review` 中不再残留纯文本 `` `<enterpriseEA>/.sharedev/dev-metadata/objects/` ``
- “自定义对象导航”文案应指向 `<enterpriseEA>/.sharedev/dev-metadata/objects/`
- 已安装的 `sharedev-apl-implement` / `sharedev-apl-code-review` 中不应出现具体工程名硬编码，例如 `fktest8507`

#### Step 9: 路径策略

不要在这里执行“安装后路径纠正”脚本。

正确做法：

1. skill 文档内部优先使用模板变量，例如：
   - `{{WORKSPACE_ROOT}}`
   - `{{ENTERPRISE_EA}}`
   - `{{SHAREDEV_ROOT}}`
2. 安装时如果需要渲染模板，只允许在运行态解析，不要把具体工程名写回技能 markdown
3. 如果当前 skill pack 仍依赖旧的相对路径，优先修模板；只有已安装旧版 `sharedev-apl-implement` / `sharedev-apl-code-review` 才允许走 Step 8.5 的受控兼容修复
4. 只要 Step 8.5 检测到旧路径，就必须执行修复，不能停留在“仅提示”
5. Step 8.5 的兼容修复结果必须保持变量化表达，例如 `<enterpriseEA>`，不得生成 `fktest8507` 这类硬编码目录名

#### Step 10: 最终目录校验

至少验证以下目录：

```text
<enterpriseRoot>/.sharedev/
<enterpriseRoot>/.sharedev/specs/
<enterpriseRoot>/.sharedev/dev-metadata/objects/
<enterpriseRoot>/.trae/skills/        # Trae 来源目录
<workspaceRoot>/.trae/skills/         # Trae 安装目标
<workspaceRoot>/.claude/skills/       # Claude 项目级
~/.claude/skills/                   # Claude 全局
~/.codex/skills/                    # Codex 全局
```

以下目录属于“按资源返回情况决定是否存在”，不应单独作为阻断条件：

```text
<enterpriseRoot>/package/fx/custom/apl/script/
<enterpriseRoot>/pwc/components/
<enterpriseRoot>/tenant-config/objects/
<enterpriseRoot>/object-dev/objects/
```

并额外创建：

```bash
mkdir -p <enterpriseRoot>/deliverables
```

## 输出要求

执行结束后必须清楚输出：

1. `TARGET_PLATFORM`
2. `EXECUTION_MODE`
3. `INSTALL_SCOPE`
4. Step 1 到 Step 10 的逐步状态（`completed` / `incomplete` / `blocked`）
5. 每一步的状态原因
6. 已完成的步骤
7. 未完成的步骤
8. 是否存在阻断项

## 阻断条件

出现以下任一情况必须停止并说明：

1. `settings.json` 缺少关键字段
2. `sharedev init --yes` 失败且无法回退
3. `sharedev spec init` 需要用户交互，但当前终端不是用户可操作终端
4. spec 目录不存在，导致后续 skills/specs 无法安装
5. 网络/权限问题导致关键资源无法下载
6. 兼容修复后 `<enterpriseEA>/.sharedev/docs/` 不存在，或对象定义既不在 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 也不在 `<enterpriseEA>/tenant-config/objects/`
7. 检测到已安装旧版 `sharedev-apl-implement` / `sharedev-apl-code-review` 仍含 legacy 路径，但兼容修复未执行或执行后仍残留旧路径
8. `sharedev init`、`sharedev apl pull --all`、`sharedev dev-metadata download --all` 等关键命令返回非零退出码，且对应资源状态仍为 `incomplete` 或 `blocked`

## 适配建议

### Trae IDE

- 优先项目级 skills
- 用户可直接交互时允许 `sharedev spec init`
- 如果 IDE 终端不是共享前台终端，或当前是 Trae agent 代执行，则必须回退到非交互模式
- Trae agent 不应因为 Step 5 已完成而提前结束；仍必须继续执行 Step 6 到 Step 10

### Claude Code

- 默认非交互
- 默认项目级 skills；只有用户明确要求时才装到 `~/.claude/skills/`
- 不要假设用户可以直接操作代理的 PTY

### Codex

- 默认非交互
- 默认全局安装到 `~/.codex/skills/`
- 不要要求执行 `codex --enable skills`

## 检查清单

- [ ] 检查了 `sharedev` CLI
- [ ] 读取了 `settings.json`
- [ ] 明确了 `TARGET_PLATFORM`
- [ ] 明确了 `EXECUTION_MODE`
- [ ] 明确了 `INSTALL_SCOPE`
- [ ] 完成了 workspace bootstrap
- [ ] 对 Step 4 的退出码做了显式判定，没有把非零返回码直接当成功
- [ ] 对 docs、PWC、APL、dev-metadata、object-dev 分别给出了状态
- [ ] 完成了 object-dev 资源拉取
- [ ] 完成了 specs 安装
- [ ] 完成了 skills 安装
- [ ] 完成了已安装技能兼容修复检查
- [ ] 如命中 legacy 路径，已实际完成兼容修复且复扫通过
- [ ] 完成了最终目录验证
- [ ] 最终按 Step 1 到 Step 10 输出了逐步状态