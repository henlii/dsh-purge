# dsh-purge（本地二开）

上游 [YuJunZhiXue/dsh-purge](https://github.com/YuJunZhiXue/dsh-purge) 的 fork
[henlii/dsh-purge](https://github.com/henlii/dsh-purge)，只做一件事：让它在
**dsh 0.2.1-rc.2** 上能正常跑。

`{{cwd}}` 和宿主 assemble 尾巴这两块取自上游客的 PR #135（sp1128，未合并）；
`toolFilter` 和子智能体叶子约束是本地补充的。

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
| `cordis.patch.yml` | 删掉 `personaSuffix`（不再写 `{{cwd}}`）；工作目录由运行时按真实路径补 |
| `lib/identity.js` | 新增 `cwdSuffixText()` / `cwdOfContext()` / `assemblyHasCwdContext()` / `fillCwdPlaceholders()`：宿主已下发 working-directory context 时后缀留空，否则用真实路径拼；最后一道闸把任何残留 `{{cwd}}` 换掉；`isCwdOnlyText` 用旧句指纹识别，避免误清用户内容 |
| `lib/core.js` | 删掉无人引用的 `ALLOW_EXEC_PERSONA_BLOCK` / `ALLOW_EXEC_PERSONA_LINE`；新增 0.2.1 版 `system-prompt/assemble` 尾巴 patch（保留 `requiredContexts` 与 contexts 过滤） |
| `lib/host-clean.js` | 识别 0.2.1 的 `fold-complete inject first` 标记和 `: [completeSection]` 尾巴，判断宿主是否已经洗过 |
| `presets/redteam/agent.cordis.yml` | 删除 `toolFilter`；`subagent` 行加 `agentPreset: redteam-worker`；`subagent_fork` 显式 `maxDepth: 1`；保留 `workflow` / `ralph`（指挥端完整工具面） |
| `presets/redteam-worker/` | 新增：执行角色预设，从指挥预设裁剪掉 `delegation` 与锚点门，只留 shell / 文件 / 技能 / 红队落库工具 |
| `lib/identity.js`（续） | `isRedteamPresetId` 把 `redteam-worker` 也算红队预设，角色稿注入才能在子会话生效 |
| `lib/redteam/index.js` | 注册和安装两份预设；先注册执行端，因为指挥端的 `tool-subagent` 行挂载时会 `readDocument('redteam-worker')` |
| `package.json` | 版本 `1.1.73-dsh021.2`，`dshTarget` 改为 `0.2.1-rc.2` |

### 叶子约束怎么保住的

删掉 `toolFilter` 时第一版只留了 `maxDepth: 1`，复核发现不够：

- `subagent_fork` 行省略 `maxDepth` 时会读服务默认值（本插件的宿主补丁会把默认抬到 10），
  子智能体可以用 fork 继续嵌套 → 现在显式写 `1`。
- `workflow-ptc` / `tool-workflow` / `tool-ralph` 派生任务时走
  `subagents.startActivation`，请求里不带 `maxDepth`，`resolveChildDepth` 直接放行，
  深度 1 的子智能体仍能经它们往下派。

所以最终方案是 0.2.1 推荐的那条路：**子会话用另一份预设跑**。
`tool-subagent` 行指定 `agentPreset: redteam-worker`，子会话进入那份预设后
根本没有派发工具（它连 `delegation` 段都没有）；指挥端继续拥有 workflow / ralph。
`maxDepth: 1` 留作二道保险。

已知残余风险：Agent Teams 的 `spawn_teammate` 仍在指挥端预设里，不受 `maxDepth`
约束，上游 `toolFilter` 时代也没有 deny 它，只靠人设和 `redteamRole` 角色稿约束。

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
