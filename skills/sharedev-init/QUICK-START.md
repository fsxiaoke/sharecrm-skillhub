# sharedev-apl-init 快速上手

## 这个技能现在怎么用

这个技能不再默认按“Trae 交互式初始化”处理所有场景，而是先区分：

1. 目标平台：`trae` / `claude` / `codex`
2. 执行模式：`interactive` / `noninteractive`
3. 安装范围：`project` / `global`

并且必须严格执行到闭环：

1. 不能把非零退出码口头描述成“基本成功”
2. 不能因为目录部分出现就提前宣布初始化完成
3. 必须按 Step 1 到 Step 10 输出 `completed` / `incomplete` / `blocked`

## 前置准备

在项目根目录放一个 `settings.json`：

```json
{
  "enterpriseEA": "your-enterprise-ea",
  "domain": "https://www.fxiaoke.com",
  "certificate": "your-api-token"
}
```

## 三个常用场景

### 1. Trae IDE，本地手动交互

推荐参数：

- `TARGET_PLATFORM=trae`
- `EXECUTION_MODE=interactive`
- `INSTALL_SCOPE=project`

典型流程：

```bash
sharedev init -e <enterpriseEA> -d <domain> -c <certificate>
cd <enterpriseEA>
sharedev docs download
sharedev pwc pull --all --type component
sharedev spec init
sharedev dev-metadata download --all
sharedev apl pull --all
```

然后把 skills/specs 安装到项目里。

### 2. Claude Code，代理自动执行

推荐参数：

- `TARGET_PLATFORM=claude`
- `EXECUTION_MODE=noninteractive`
- `INSTALL_SCOPE=project`

优先执行：

```bash
sharedev init --yes --agent claude --pull-all-apl --pull-all-pwc -e <enterpriseEA> -d <domain> -c <certificate>
```

然后补齐缺失资源，并把 skills 安装到：

```text
.claude/skills/
```

### 3. Codex，代理自动执行

推荐参数：

- `TARGET_PLATFORM=codex`
- `EXECUTION_MODE=noninteractive`
- `INSTALL_SCOPE=global`

优先执行：

```bash
sharedev init --yes --agent codex --pull-all-apl --pull-all-pwc -e <enterpriseEA> -d <domain> -c <certificate>
```

然后把 skills 安装到：

```text
~/.codex/skills/
```

然后继续核对：

- `.sharedev/docs/`
- `.sharedev/dev-metadata/`
- `.codex/`
- `package/fx/custom/apl/script/`

如果上述任一项未补齐，就只能记为 `incomplete`，不能说“初始化完成”。

### 4. Trae agent，代理自动执行

推荐参数：

- `TARGET_PLATFORM=trae`
- `EXECUTION_MODE=noninteractive`
- `INSTALL_SCOPE=project`

优先执行：

```bash
sharedev init --yes --agent trae --pull-all-apl --pull-all-pwc -e <enterpriseEA> -d <domain> -c <certificate>
```

说明：

- 只有 Trae 用户本人能直接操作前台终端时，才使用第 1 种交互模式
- Trae agent 代执行时，不要默认走交互模式

## 最终目录长什么样

```text
<workspace>/
├── <enterpriseEA>/
│   ├── .sharedev/
│   ├── package/fx/custom/apl/script/
│   ├── pwc/components/
│   └── deliverables/
├── .trae/skills/            # Trae 项目级
├── .claude/skills/          # Claude 项目级
└── ~/.codex/skills/         # Codex 全局
```

不是所有平台都会同时出现这些目录，取决于本次选择的 `TARGET_PLATFORM` 和 `INSTALL_SCOPE`。

## 重要变化

### 不再依赖“猜环境”

不要根据以下条件推断目标平台：

- 是否存在 `~/.codex`
- 是否存在 `~/.claude`
- 是否存在 `.trae`

这些只能说明机器上装过什么，不能说明“本次应该给哪个平台安装”。

### 不再默认要求交互式 `sharedev spec init`

如果当前是在代理 PTY 里执行，而不是用户可直接操作的终端，就不要把 `sharedev spec init` 当成默认路径。

### 不再要求 `codex --enable skills`

Codex 部分只关注把技能安装到正确目录，不把 `codex --enable skills` 当作必做步骤写进流程。

### 不再在安装后用 `sed` 暴力改路径

如果 skills 依赖路径，优先改成模板变量，例如：

- `{{WORKSPACE_ROOT}}`
- `{{ENTERPRISE_EA}}`
- `{{SHAREDEV_ROOT}}`

### 已安装旧技能需要做一次兼容修复

如果以下技能之前已经装过旧版本：

- `sharedev-apl-implement`
- `sharedev-apl-code-review`

那还需要补做一次路径兼容检查，因为它们可能还在引用旧路径。

现在标准路径已经是：

- docs：`<enterpriseEA>/.sharedev/docs/`
- objects：`<enterpriseEA>/.sharedev/dev-metadata/objects/`

也就是说，原先的 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 等历史相对路径，应该统一迁移到：

```text
<enterpriseEA>/.sharedev/dev-metadata/objects/
```

处理原则：

1. 只检查已安装的 `sharedev-apl-implement`、`sharedev-apl-code-review`
2. 只修 docs/object 两类已知旧路径
3. 不要扫所有 skills 做大规模替换
4. 不要把命中的旧路径改写成“历史对象目录”之类的模糊描述；必须替换成标准路径表达

这里的“旧 object 路径”不只包括链接：

- `<enterpriseEA>/.sharedev/dev-metadata/objects/`
- `<enterpriseEA>/.sharedev/dev-metadata/objects/`

还包括文案里直接写死的纯文本：

- `` `<enterpriseEA>/.sharedev/dev-metadata/objects/` ``
- `**自定义对象导航** (查看 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 目录下以 `__c` 结尾的文件)`

这些都应该统一改到：

```text
<enterpriseEA>/.sharedev/dev-metadata/objects/
```

换句话说，兼容流程必须把：

```text
.sharedev/object/
```

视为旧路径，并强制调整为：

```text
<enterpriseEA>/.sharedev/dev-metadata/objects/
```

修复后的技能源码应直接保留占位变量形式：

```text
<enterpriseEA>/.sharedev/docs/
<enterpriseEA>/.sharedev/dev-metadata/objects/
```

## 常见失败点

### `sharedev init` 提示需要 `--yes`

说明你在非交互环境里执行。改用：

```bash
sharedev init --yes --agent <target_platform> --pull-all-apl --pull-all-pwc -e <enterpriseEA> -d <domain> -c <certificate>
```

### `sharedev init` 返回非零退出码，但目录已经创建

这只能说明“基础目录可能已落盘”，不能直接说明初始化已完成。

必须继续做两件事：

1. 检查 `.sharedev/settings.json`、`.git/`、平台 spec 目录是否存在
2. 逐项补核 docs、PWC、APL、dev-metadata、object-dev

只要 `package/fx/custom/apl/script/`、对象元数据目录或必需 spec/skills 仍缺失，就必须把结果记为 `incomplete` 或 `blocked`。

### `sharedev docs download` 卡在输入目录

说明命令需要交互输入。代理环境下不要依赖它的交互提示，优先走 `sharedev init --yes ...`。

### `sharedev spec init` 不能让用户手动选

说明当前命令跑在代理自己的 PTY 里，不是用户前台终端。此时应切到非交互路径，而不是继续假设用户能直接操作菜单。
