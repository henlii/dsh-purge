# dsh-purge（本地二开）

上游 [YuJunZhiXue/dsh-purge](https://github.com/YuJunZhiXue/dsh-purge) 的 fork
[henlii/dsh-purge](https://github.com/henlii/dsh-purge)，只做一件事：让它在
**dsh 0.2.1-rc.2** 上能正常跑。

## 为什么要有这个二开

上游 README 已声明「暂时停更，新版 dsh 不保证跟上」。1.1.72 在 dsh 0.2.1-rc.2 上有两处硬不兼容：

1. **`{{cwd}}` prompt 变量已被移除**。0.2.1 的 `dsh-agent-loop` 只注册 `provider` / `model`
   两个变量，工作目录改由 `workingDirectory` 服务注入 context 段。上游在
   `cordis.patch.yml` 和运行时的 `lib/identity.js` 里仍往 persona suffix 塞
   `Your working directory is {{cwd}}.`，于是**每一轮对话在渲染系统提示时抛
   `unknown prompt variable`，turn 直接以错误结束**。
2. **`tool-subagent` 拒绝 `toolFilter`**。0.2.1 起该配置项被删除（挂载即抛错），
   红队预设因此被判 `registered-broken`，模式下拉里不可用。

## 改了什么

| 文件 | 改动 |
|---|---|
| `cordis.patch.yml` | `personaSuffix` 由 `Your working directory is {{cwd}}.` 改为 `""` |
| `lib/identity.js` | `CWD_ONLY_SUFFIX` 改为空串；`stripHarnessPersona` 不再把官方身份句重写成 `Working directory: {{cwd}}.`；`isCwdOnlyText` 把旧的 `Your working directory is {{cwd}}.` 也算作 cwd-only |
| `lib/core.js` | `ALLOW_EXEC_PERSONA_BLOCK` / `ALLOW_EXEC_PERSONA_LINE` 去掉 `Working directory: {{cwd}}.` 前缀；旧句加入可丢弃判断 |
| `presets/redteam/agent.cordis.yml` | 删除 `toolFilter`；`subagent` 与 `subagent_fork` 都显式 `maxDepth: 1`；不再挂 `workflow-ptc` / `tool-workflow` / `tool-ralph` |
| `package.json` | 版本 `1.1.73-dsh021.1` |

剩下的 `{{cwd}}` 只出现在「匹配旧官方文本」的字符串里（判定是否可丢），
不参与运行期注入。

### 叶子约束为什么不是只靠 maxDepth

删掉 `toolFilter` 时第一版只留了 `maxDepth: 1`，复核发现不够：

- `subagent_fork` 行省略 `maxDepth` 时会读服务默认值（本插件的宿主补丁会把默认抬到 10），
  子智能体可以用 fork 继续嵌套 → 现在显式写 `1`。
- `workflow-ptc` / `tool-workflow` / `tool-ralph` 派生任务时走
  `subagents.startActivation`，请求里不带 `maxDepth`，`resolveChildDepth` 直接放行，
  深度 1 的子智能体仍能经它们往下派 → 现在整组不挂。

代价：这两个工具在指挥端（父会话）也没了。指挥端人设本来就只允许用 `subagent` 派活，
所以实际影响是没有 workflow 编排和 ralph 循环。已知残余风险：Agent Teams 的
`spawn_teammate` 不受 `maxDepth` 约束，上游的 `toolFilter` 时代也没有 deny 它，
只靠人设（以及 `redteamRole` 角色稿）约束。

## 验证

副本 `DSH_HOME` + headless 一句话：

```bash
# 副本 profile 把 dsh-purge 指向本目录，跑一句
DSH_HOME=<copy> dsh headless "reply with the single word: ok"
```

修复前报 `unknown prompt variable "{{cwd}}"`，修复后正常返回 `ok`；`--dump-config`
里 `personaSuffix` 为 `''`。

## 部署

线上一律按 git ref 装（`~/.dsh/profiles/web/package.json`），不用 `link:`，
避免运行时直接读工作区源码：

```json
"dsh-purge": "github:henlii/dsh-purge#<commit>"
```

改完装依赖后重启 `dsh.service`，再开一条新会话验证 `turn/end` 是 `completed`。

## 升级上游

上游恢复维护后重新评估：`git fetch upstream && git log --oneline HEAD..upstream/master`。
如果上游自己适配了 0.2.x，优先回到上游，不要维护这份 diff。
