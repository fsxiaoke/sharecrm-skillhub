# ShareDev CLI 指南

这份指南只保留和 `sharedev-apl-init` 直接相关的命令，并区分本地交互模式与代理非交互模式。

## 1. 项目初始化

### 交互模式

```bash
sharedev init -e <enterpriseEA> -d <domain> -c <certificate>
```

适用场景：

- 用户在本地终端里自己操作
- 允许手动回答后续提示

### 非交互模式

```bash
sharedev init --yes --agent <trae|claude|codex> --pull-all-apl --pull-all-pwc -e <enterpriseEA> -d <domain> -c <certificate>
```

适用场景：

- Trae/Claude/Codex 代理执行
- 无法把交互菜单直接交给用户操作

说明：

- 非交互模式下必须带 `--yes`
- 建议显式带 `--agent`
- `--pull-all-apl` 和 `--pull-all-pwc` 可以减少后续补步骤

## 2. 文档下载

```bash
sharedev docs download
```

说明：

- 当前实现会要求输入输出目录
- 在代理 PTY 中执行时，不应默认认为用户能手动输入
- 如果没有共享给用户的前台终端，优先回退到 `sharedev init --yes ...`

期望结果：

- `.sharedev/docs/apl/`
- `.sharedev/docs/pwc/`

## 3. PWC 拉取

```bash
sharedev pwc pull --all --type component
```

补充：

```bash
sharedev pwc pull --all --type plugin
```

注意：

- `plugin` 可能返回“没有资源”，这不应被误判为整个初始化失败

## 4. Spec 模板初始化

### 交互模式

```bash
sharedev spec init
```

说明：

- 这个命令会弹出平台选择菜单
- 只适用于用户可直接操作当前终端的场景

### 非交互模式建议

不要把 `sharedev spec init` 当成代理默认路径。

优先做法：

```bash
sharedev init --yes --agent <target_platform> ...
```

如果这条命令后已经生成：

- `.trae/`
- `.claude/`
- `.codex/`

就直接复用，不再额外执行 `sharedev spec init`。

## 5. Dev Metadata 下载

```bash
sharedev dev-metadata download --all
```

期望结果：

- `.sharedev/dev-metadata/objects/`
- `.sharedev/dev-metadata/namespaces/`

说明：

- 允许部分对象下载失败，但要在结果里明确汇报失败数量

## 6. APL 代码同步

```bash
sharedev apl pull --all
```

期望结果：

- `package/fx/custom/apl/script/`

## 7. 编译与静态分析

```bash
sharedev apl compile <apiName>
sharedev apl analyze <apiName>
```

用途：

- 初始化后验证单个函数是否能通过平台检查
- 代码审查阶段作为强制质量门禁

## 8. APL 推送

```bash
sharedev apl push <apiName> -m "commit message"
```

说明：

- 这是“推到纷享平台服务端”，不是 `git push`
- 远端版本更新成功后，应明确回报版本号

## 9. 推荐初始化模板

### Trae，本地交互

```bash
sharedev init -e <enterpriseEA> -d <domain> -c <certificate>
cd <enterpriseEA>
sharedev docs download
sharedev pwc pull --all --type component
sharedev spec init
sharedev dev-metadata download --all
sharedev apl pull --all
```

### Claude，代理非交互

```bash
sharedev init --yes --agent claude --pull-all-apl --pull-all-pwc -e <enterpriseEA> -d <domain> -c <certificate>
```

然后检查是否仍缺失：

- docs
- dev-metadata
- `.claude/`

### Codex，代理非交互

```bash
sharedev init --yes --agent codex --pull-all-apl --pull-all-pwc -e <enterpriseEA> -d <domain> -c <certificate>
```

然后检查是否仍缺失：

- docs
- dev-metadata
- `.codex/`

## 10. 已安装技能兼容检查

如果初始化前已经安装过旧版本技能，需要检查：

- `sharedev-apl-implement`
- `sharedev-apl-code-review`

重点检查两类路径：

- docs 根目录是否指向 `<enterpriseEA>/.sharedev/docs/`
- object 元数据是否指向 `<enterpriseEA>/.sharedev/dev-metadata/objects/`

旧的 object 路径通常不再正确：

```text
<enterpriseEA>/.sharedev/dev-metadata/objects/
```

另外还要检查文案中直接写死的旧提示，例如：

```text
.sharedev/object/
**自定义对象导航** (查看 `<enterpriseEA>/.sharedev/dev-metadata/objects/` 目录下以 `__c` 结尾的文件)
```

应迁移为：

```text
<enterpriseEA>/.sharedev/dev-metadata/objects/
```

注意：

- 这不是通用 CLI 命令
- 不要扫描全部 skills
- 只对上述两个已知技能做受控兼容修复

## 11. 不推荐做法

以下做法不应继续作为默认方案：

1. 根据 `~/.codex` 或 `~/.claude` 是否存在来判断本次目标平台
2. 在代理环境里默认执行交互式 `sharedev spec init`
3. 在安装后批量 `sed` 修改 skills markdown 路径
4. 把 `codex --enable skills` 写成 Codex 初始化的必需步骤
