# APL 开发环境初始化参考

这份参考文档服务于 `sharedev-apl-init`，目标是解释为什么初始化流程要先区分平台，再区分交互方式。

## 核心原则

初始化前先明确三件事：

1. `TARGET_PLATFORM=trae|claude|codex`
2. `EXECUTION_MODE=interactive|noninteractive`
3. `INSTALL_SCOPE=project|global`

不要用“目录是否存在”来猜本次平台，例如：

- `~/.codex` 存在，不代表这次就该装到 Codex
- `.trae` 存在，不代表这次用户就想用 Trae

## 两阶段模型

### Phase A: Workspace Bootstrap

负责把 `<enterpriseEA>/` 工作区准备好：

- `sharedev init`
- docs
- pwc
- apl
- dev-metadata

### Phase B: Agent Integration

负责把目标平台需要的 skills/specs 装到正确位置：

- Trae: `.trae/skills/`
- Claude 项目级: `.claude/skills/`
- Claude 全局: `~/.claude/skills/`
- Codex 全局: `~/.codex/skills/`

## 交互与非交互

### 交互模式

只适用于：

- 用户确实在自己终端前操作
- 用户可以直接回答 CLI 提示

典型命令：

```bash
sharedev init -e <enterpriseEA> -d <domain> -c <certificate>
sharedev spec init
sharedev docs download
```

### 非交互模式

适用于：

- Trae 代理执行
- Claude Code
- Codex

推荐命令：

```bash
sharedev init --yes --agent <target_platform> --pull-all-apl --pull-all-pwc -e <enterpriseEA> -d <domain> -c <certificate>
```

原因：

- 避免卡在 `sharedev init` 的交互确认
- 避免卡在 `sharedev spec init` 的平台选择
- 避免卡在 `sharedev docs download` 的目录输入

## 为什么不能默认走 `sharedev spec init`

因为很多代理工具虽然能启动 PTY，但那个 PTY 不是用户真实可交互的终端。

结果就是：

- 代理能看到菜单
- 用户看不到或不能直接操作那个菜单
- 命令会卡住，或者只能由代理代答

所以：

- 本地终端用户可以手选
- 代理环境优先用非交互初始化

## 为什么不建议安装后批量改 markdown 路径

安装后用 `sed` 扫描一批 `.md` 文件替换路径，问题很多：

1. 假设原始路径模式固定
2. 假设技能目录结构固定
3. 容易误改非目标文本
4. 项目级与全局安装路径规则不同，维护成本高

更好的做法是：

1. 在 skill 文档里使用模板变量
2. 安装时只做一次明确渲染
3. 或者改成运行时根据 workspace 规则解析路径

推荐模板变量：

- `{{WORKSPACE_ROOT}}`
- `{{ENTERPRISE_EA}}`
- `{{SHAREDEV_ROOT}}`

## 为什么仍然需要一个兼容修复步骤

虽然新规则是不再默认批量改路径，但已经安装过的旧版技能仍然可能残留历史路径。

尤其是这两个技能：

- `sharedev-apl-implement`
- `sharedev-apl-code-review`

它们最容易受两个迁移影响：

1. 文档根路径迁移到 `<enterpriseEA>/.sharedev/docs/`
2. object 查找路径统一为 `<enterpriseEA>/.sharedev/dev-metadata/objects/`

所以 `sharedev-apl-init` 仍应保留一个“受控兼容修复”流程，但必须满足两个限制：

1. 只修这两个已知技能
2. 只修 docs/object 两类已知 legacy 路径

这和“安装后对所有 skills 做大规模 `sed` 替换”不是一回事。

## 兼容修复时要检查什么

至少检查：

- 旧的相对 docs 路径是否还存在
- 旧的 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 或相对 objects 路径是否还存在
- 旧的纯文本 `` `<enterpriseEA>/.sharedev/dev-metadata/objects/` `` 是否还存在
- `**自定义对象导航** (查看 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 目录下以 `__c` 结尾的文件)` 是否还存在
- 修复后的目标目录是否真实存在

新的 object 目录必须指向：

```text
<enterpriseEA>/.sharedev/dev-metadata/objects/
```

也就是：

```text
.sharedev/object/
```

在兼容流程里必须被当成旧路径，统一调整到：

```text
<enterpriseEA>/.sharedev/dev-metadata/objects/
```

## 推荐平台策略

### Trae

- 用户亲自操作且终端可直接交互时，默认 `interactive + project`
- Trae agent / 代理执行时，默认 `noninteractive + project`

### Claude Code

- 默认 `noninteractive + project`

### Codex

- 默认 `noninteractive + global`

## 成功判定

初始化完成后，至少应看到：

```text
<enterpriseEA>/.sharedev/
<enterpriseEA>/deliverables/
```

对象定义目录至少存在一处：

```text
<enterpriseEA>/.sharedev/dev-metadata/objects/
<enterpriseEA>/tenant-config/objects/
<enterpriseEA>/object-dev/objects/
```

`package/fx/custom/apl/script/` 和 `pwc/components/` 只有在远端实际返回 APL/PWC 资源时才应作为成功产物要求。以及目标平台对应的 skills 目录。
