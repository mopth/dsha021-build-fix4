# 插件备份适配 fix4（dsh 0.2.1-rc.2）报告

范围：只做静态分析 + 在盒子里用 0.2.1-rc.2 的真实 npm 包加载插件；没碰手机、没上传、没改备份原件（补丁都在副本 `patched/`）。备份内容没有打印任何密钥值。
依据：`PATCH_AUDIT_0.2.1-rc.2.md`；npm 上的 0.1.7-rc.2 与 0.2.1-rc.2 官方包逐个比对导出；fix4 源码（PluginRestoreGraph / NativeRestoreTargets / BuiltinPlugins）。

## 0. 先说结论

备份里的 36 个“插件包”其实只有 **6 个是你自己装的插件**，其余 30 个是：
- 14 个是旧版 0.1.7-rc.2 官方包的“改过的副本”（`managedOverride`，是你以前打过补丁/overlay 的 `@deepseek-ai/dsh*`、`libreoffice-kit`）→ **绝不能导入**，否则会盖住 fix4 自带的 0.2.1-rc.2；
- 16 个是第三方依赖（react、zod、sharp、schemastery 等），只给上面 6 个插件用。

6 个用户插件分类：
| 分类 | 个数 | 插件 |
|---|---|---|
| 原样可用 | 2 | dsh-batch-tool-calls、dsh-our-free-model |
| 需小补丁（已做好副本，已验证能加载） | 3 | dsh-peak-chip、dsh-session-health（只改版本范围）；dsh-subagent-model-picker（改版本范围 + 补回被删的 `settleRun`） |
| 能加载但风险高、建议最后/默认禁用 | 1 | billion-context（会改官方配置项，见撞车检查） |
| 不兼容（无法修） | 0 | — |

## 1. 0.1.7-rc.2 → 0.2.1-rc.2 宿主契约差异（与插件相关的部分）

我把插件用到的 14 个官方包两个版本的导出逐个比了：
- 无变化：dsh-credentials、dsh-llm、dsh-tools、dsh-commands、dsh-settings、dsh-jobs、dsh-system-prompt、dsh-client-ui-conversation、dsh-session-projection。
- 只新增：dsh-home-paths(+6)、dsh-session(+3)、dsh-typert-protocol(+3)。
- **被删**：`dsh-subagent` 删了 `settleRun`、`resolveChildCwd`、`validateConfiguredCwd`；`dsh-client-ui-primitives` 删了 `projectUserText`。
  - 只有 model-picker 用到 `settleRun`（加载直接报错）；没有任何插件用 `projectUserText`、`resolveChildCwd`、`validateConfiguredCwd`。
- **版本门槛（最关键）**：0.2.1-rc.2 的 app-boot（`unmetRequirements`）如果插件写了 `engines.dsh` 就只看它，否则逐个检查 `peerDependencies` 里 `@deepseek-ai/dsh*` 的范围，用 `includePrerelease` 对 `0.2.1-rc.2` 判断；不满足就判“需求未满足”，插件不会加载。
  - 备份里的范围是 `^0.1.x-rc.y`，在 0.x 下 caret 不跨 minor，所以 peak-chip / picker / session-health 三个会被判不满足。
  - dsh-our-free-model 的 `engines.dsh` 写了 `^0.2.0-rc.1`，通过；batch-tool-calls、billion-context 没声明 dsh 依赖，通过。
- 加载实测（Node 24，插件放进装了 0.2.1-rc.2 全套包的目录，`import()` 入口）：5 个服务端入口加载成功；model-picker 因 `settleRun` 失败（已补丁后成功）。
- 没法验证的：浏览器端 client.js 在手机网页里的实际表现（peak-chip、session-health 注入 `client-ui-conversation`/`primitives`；它们的导出两个版本一致，所以风险低，但没有真机证据）。

## 2. 36 个包逐个表

### 2.1 用户插件（6）
| 插件 | 版本 | 结论 | 理由 |
|---|---|---|---|
| dsh-batch-tool-calls | 1.1.2 | 原样可用 | 只依赖 schemastery；无 dsh 范围；服务端加载成功 |
| dsh-our-free-model | 2.1.0 | 原样可用 | `engines.dsh` 已含 `^0.2.0-rc.1`；用 llm/credentials/commands/cordis，导出无变化；加载成功。来源 URL 为空 → 必须用备份里的副本 |
| dsh-peak-chip | 4.2.0 | 需小补丁 | peer `^0.1.5-rc.2` 过不了门槛；代码本身加载成功。补丁：peer 加 `|| ^0.2.1-rc.1`（见第 8 节：`^0.2.0-rc.1` 在 fix4 自己的 semver 里判不过）。备份里记录的来源是 4.1.8 的 URL，装的是 4.2.0 的本地版 → **别从 URL 重装，会降级**，用备份副本 |
| dsh-session-health | 0.6.2 | 需小补丁 | 同上，peer 7 个包 `^0.1.0-rc.6`；补丁同上；来源 URL 为空 → 用备份副本。UI 插槽与 dsh-web-mobile 重叠，见撞车检查 |
| dsh-subagent-model-picker | 0.1.1 | 需小补丁 | peer 5 个包 `^0.1.0-rc.5` 不满足；另外依赖已删的 `settleRun`。补丁：peer 加 `|| ^0.2.1-rc.1` + 文件内本地补回 `settleRun`（按 0.1.7-rc.2 实现等价复刻）。**子代理实跑未验证**（只验证了能加载） |
| billion-context | 0.1.196-hardened.2.3 | 能加载，高风险，条件可用 | 没有 dsh 范围，加载成功；但它是“深度改写”类插件（上下文/压缩/重试），行为只能真机验证；见撞车检查 |

### 2.2 官方包的旧副本（14）→ 全部「不导入」
| 包 | 版本 | 结论 |
|---|---|---|
| @deepseek-ai/dsh | 0.1.7-rc.2 | 不导入（宿主本体） |
| dsh-settings、dsh-persona、dsh-plugin-manager、dsh-web-frontend | 0.1.7-rc.2 | 不导入（会盖住 0.2.1-rc.2 同名官方包） |
| dsh-client-locale、client-resources、client-ui-conversation、client-ui-plugin-manager、client-ui-settings、client-ui-sidebar-documentpreview、client-ui-subagent、client-ui-workspace | 0.1.7-rc.2 | 不导入（同上，前端包，换成旧版会让网页界面错位） |
| @deepseek-ai/libreoffice-kit | 0.1.1 | 不导入（fix4 自带的版本为准） |
这 13 个带 `managedOverride` 标记，另有 `@deepseek-ai/dsh` 本体。

### 2.3 第三方依赖（16）→ 随所属插件走，不单独导入
react 18.3.1、loose-envify、js-tokens(managed)、zod 4.1.8、sharp 0.35.5 + @img/colour + @img/sharp-libvips-linux-arm64 + @img/sharp-linux-arm64、detect-libc、semver、fflate、@deepseek-ai/schemastery（3 份）、@deepseek-ai/cosmokit（2 份）、@standard-schema/spec（2 份）。
- 这些是插件私有 node_modules 里的本地副本，放在对应插件目录下，不会影响宿主；
- 注意 schemastery/cosmokit 的本地副本（3.18.4 / 1.8.5）比 fix4 宿主（3.18.5 / 1.8.6）稍旧：batch-tool-calls、session-health 自带旧版，只是插件内部用；peer 声明的 schemastery 由宿主提供。风险低。
- sharp 是 linux-arm64 原生模块，随 billion-context 走；Node 版本变了需要真机 `import sharp` 验证（盒子里不是 aarch64 真机，未验证）。

## 3. 撞车检查

fix4 内置：`dsh-device-shell-guide`、`dsh-task-notifier`、`dsh-status-overlay`、`dsh-web-mobile`、`dsh-computer-use-android`、`dsh-auto-review`、`dsh-tool-vscreen`、`dsh-app-integration`（内部）、`dsh-destructive-guard`（内部），以及官方 `@deepseek-ai/dsh-base`、`dsh-web-app` 及其全部 `@deepseek-ai/*` 依赖（0.2.1-rc.2）。

### 3.1 app 自己的保护（fix4 源码已有）
- `BuiltinPlugins.system()` = 9 个内置 + dsh-base + dsh-web-app。恢复时这些名字**一律丢弃**（`PluginRestoreGraph.discardSystemPlugins`、`NativeRestoreTargets` 返回 `CURRENT_APK_SYSTEM_PLUGIN`），注释写明“旧 profile/归档不得提供同名源码”。
- 恢复只是“隔离区”（`quarantined:true, executed:false`），不会自动启用；内置插件每次启动由 `register-builtin-plugins.py` 重新注册。
- **缺口**：保护只认上面 11 个名字。备份里那 13 个 `managedOverride` 的 `@deepseek-ai/*` 旧副本**不在名单里**，会被保留到隔离区（带警告 `MANAGED_OVERRIDE_RETAINED_FOR_REVIEW`），并且 peak-chip 的边指向旧版 `dsh-settings`、session-health 的边指向旧版 `dsh-client-ui-conversation`——一旦激活就会盖住宿主的 0.2.1-rc.2 包（两份单例）。所以这两条边必须去掉；这需要手工选择性导入（见第 5 节），不需要改 APK。
- 备份里的 `profiles/web/package.json`、`cordis.patch.yml` **不要整份覆盖**：它们含 8 个内置插件的 `link:/root/dsha-*` 声明和 bundles 列表，整份覆盖等于把 0.1.7 时代的内置清单盖掉 fix4 的。只摘取下面标明的补丁条目。

### 3.2 6 个用户插件逐个对撞
| 插件 | 名字/ID | 命令/工具 | 配置键 / patch id | 端口 | 钩子/UI 插槽/提示词 | 撞车结论 |
|---|---|---|---|---|---|---|
| dsh-peak-chip | `dsh-peak-chip`/`peak-chip`，无同名 | 无 | `peak-chip.*`（余额状态很大，含用户账户余额类数据） | 不监听端口；**作为客户端**访问内置 3090 桥 `/app/open`，读 `$DSH_HOME/.bridge_token`（与 dsh-task-notifier 同一个文件，只读） | 注入 conversation 页头 | 共享而非冲突：依赖内置桥，保留。**导入但去掉指向旧版 dsh-settings 的边**；`selfDir` 指向 `/root/.dsh/plugin-src/dsh-peak-chip`，目录必须放在同路径 |
| dsh-batch-tool-calls | `batch-tool-calls` 无同名 | 无工具；提示词段 `batch-tool-calls:policy` order 9500 | `batch-tool-calls.*` | 无 | 和 `dsh:device-shell-guide`（order 150）都往系统提示里加段，内容可能都提到 shell（配置 `mentionShell:true`） | 软重叠：段名与顺序不同，不会覆盖；提示词可能重复，不处理，观察即可 |
| dsh-subagent-model-picker | `subagent-model-picker` 无同名 | 工具 `subagent_model`、`subagent_models` | `subagent-model-picker.*` | 无 | 无 | 功能重叠、名字不撞：官方 `dsh-tool-subagent` 的 `subagent` 工具自带“子代理选模型”（`modelSelectionSettings`，默认关）和 `list_subagent_models`。0.1.7 里也是这样，不是新问题。结论：导入但标「可选」；若以后打开官方 modelSelectionSettings，二选一，别两个都开 |
| dsh-our-free-model | `our-free-model` 无同名 | 无工具；路由 `our-free-model`、`our-free-model-region` | `our-free-model.*`（含 provider 密钥，已掩码未打印） | 无（出站） | 提供 LLM 路由，不碰 `deepseek-official` | 无撞车。保留你的“模型切换” |
| dsh-session-health | `session-health` 无同名 | 斜杠命令 `/health`（宿主和内置里没找到同名） | `session-health.*` | 无 | **插槽重叠**：用 `sidebar.footer.action`、`conversation.session.header.actions`，与内置 dsh-web-mobile 同槽；代码里已写明为手机适配改坐 `header.actions` | 同槽共存（插槽是列表，不是互斥），但真机未验证两者布局。标「禁用冲突项」：先不启用，需要时单独开 |
| billion-context | `bili-native`（loader id），包名 `billion-context` | 无同名 | **改官方配置**：`compaction-basic.auto=false`、`llm-deepseek.retryPolicy`、`llm-deepseek-account.retryPolicy`；知道 `auto-review` 这个内置 patch id，仅检测并提示成本问题，不改它 | 无 | 全方位上下文接管 | 不撞内置，但**改官方 bundle 的 3 个条目**，与你自己在 web 补丁里的 `llm-deepseek` 重试设置重叠（用户补丁在后，后者生效）。标「禁用冲突项」：导入但默认禁用，确认 compaction 行为后再开；不要让它覆盖内置 `dsh-auto-review` |

### 3.3 其余 30 个包的撞车处理
| 类别 | 处理 |
|---|---|
| 14 个官方旧副本（含 `@deepseek-ai/dsh`） | skip / 不导入 |
| 16 个第三方依赖 | 只作为 6 个插件的私有依赖，随插件目录走；不进 profile 顶层 |
| 内置名（dsh-web-mobile / dsh-app-integration / dsh-auto-review / dsh-computer-use-android / dsh-destructive-guard / dsh-device-shell-guide / dsh-status-overlay / dsh-task-notifier / dsh-tool-vscreen） | 备份的插件包里本来没有它们的源码（只出现在 profile 的 `systemPlugins` 启用状态里：8 个均为 enabled）；这些状态与 fix4 默认一致，**不需要**导入 |

## 4. 备份导入路径检查

- 备份是 plugins 范围、`DSHABAK5`，manifest 的 `plugins.version=2`，`PluginRestoreGraph` 接受 1 和 2；没有按“app 版本 0.1.7-rc2”拒绝的门槛（只校验归档格式版本、图版本、包 id/名、`package.json` 名字一致）。
- `plugins.complete=false`、warning `PLUGIN_METADATA_UNREADABLE`：表示备份时有些元数据读不到，恢复会带 `SOURCE_GRAPH_INCOMPLETE` 警告，不会拒绝。
- 556 个 `managed` 节点带 0.1.7 的 proof，新运行时校验不过 → `MANAGED_DEPENDENCY_UNAVAILABLE`，这些边不重建，插件改用宿主自带的 0.2.1-rc.2 官方包（这是对的）。但上面说的两个 `managedOverride` 边会被重建，这是要避免的。
- `plugin-sources.json`：6 项，其中 `dsh-our-free-model`、`dsh-session-health` 的来源为空（本地装的，不能联网重装）；`dsh-peak-chip` 记录 4.1.8 的 URL 而实际 4.2.0；`billion-context` 来源是 GitHub 仓库，实际为本地 hardened 版 —— 所以都应该用备份里的副本，别用 `plugin add` 联网重装。
- `plugin-history`：里面是 batch-tool-calls 1.0.0、peak-chip 4.1.9 的旧快照，只是回滚用，不导入。

## 5. 推荐导入路线（不需要新 APK）

1. **在 fix4 上用 app 自带“恢复”恢复备份的 plugins 范围**：得到隔离区，内置 9 个自动丢弃，无任何覆盖（验证依据是源码，未真机验证）。
2. **不要在界面里“全部启用/整体应用”**。逐个手工启用 6 个插件，用本报告 `patched/` 里的三个补丁副本替换 peak-chip、session-health、subagent-model-picker；另外 3 个原样。
3. 插件放到 `/root/.dsh/plugin-src/<name>`（路径与备份一致），profile 里只加每个插件的 `link:` 和 `dsh.profile.bundles` 一项，**不替换**整份 profile 的 package.json。
4. 从备份的 `cordis.patch.yml` 只摘：`peak-chip`、`ui-settings-general`、`locale`、`ui-theme`、`ui-chat`、`agent-default-model`（deepseek-flash / max）、`llm-deepseek` 重试。每个都是 id 级 patch，不会改内置条目。
5. 顺序：batch-tool-calls → our-free-model → peak-chip → subagent-model-picker → session-health → billion-context，每个启用后点一次“重启 Web”，看有没有报错再下一个。
6. 配置里 `peak-chip` 的 `balance` 等数字是旧快照，新装后会自己刷新。

**需要新 APK 的情况**：
- 想让 app 的恢复流程自动**过滤那 13 个 `managedOverride` 旧官方包**，需要改 `BuiltinPlugins` 名单/恢复逻辑（新 APK，同时手机要重新装，你说过流量紧，不建议）。手工路线能绕过，所以不是必须。
- 其余（补丁、peer 放宽）都在插件目录里，不需要新 APK。

## 6. 你会丢/会得到什么
- 保留：6 个插件、你的 web 配置（语言、主题、默认模型 deepseek-flash+max、重试策略）、our-free-model 的路由（模型切换）。
- 丢弃：13 个旧官方包的 overlay（你以前打的 overlay4 / 补丁；若里面有你改过的界面/设置行为，会回到官方 0.2.1-rc.2 的默认）、profile 的整份旧声明、plugin-history 快照。
- 得到：0.2.1-rc.2 官方的新版前端/设置/会话等；内置的 9 个插件由 fix4 自带，不用从备份导。
- 运行时切换（bxroot/proot）不在插件范围，与此无关；内置插件不会被覆盖。

## 7. 没验证的
- 插件在手机网页里的实际界面表现（client.js），尤其 session-health 与 dsh-web-mobile 同插槽共存、billion-context 全部行为；
- model-picker 补丁后子代理真实调用；
- sharp 原生模块在手机 Node 下是否能加载；
- 恢复流程到隔离区的真机行为（只读了源码）。

## 附：产物
- `patched/dsh-peak-chip`、`patched/dsh-session-health`、`patched/dsh-subagent-model-picker`：补丁后的副本（不含 node_modules）
- `plugin-patches.diff`：这三个的差异
- 备份原件 （本地目录） 未改动

## 8. fix4 自带 plugin-semver.cjs 实测（审阅保留意见 #4）

用 fix4 assets 的 `plugin-semver.cjs`（sha256 前缀 e6502ec42bd53521，与 runtime-descriptor 一致）按它自己的 stdin JSON 协议跑 `satisfies('0.2.1-rc.2', range)`。该脚本**没有** includePrerelease，所以 `0.2.1-rc.2` 只匹配“同一 0.2.1 元组里带预发布号”的比较子。

**第一版补丁（`|| ^0.2.0-rc.1`）：全部失败。** 例：`^0.2.0-rc.1` → false，`^0.1.5-rc.2 || ^0.2.0-rc.1` → false。这是审阅人担心的点，成立。（对 dsh 自己 app-boot 的启动门槛没问题，因为它用 includePrerelease；但 fix4 的插件管理页会判“版本不匹配”。）

**修正：改成 `|| ^0.2.1-rc.1`**（预发布写在 0.2.1 元组上）。重新验证，两套判定都用：

| 插件 | 被检查的 peer 数 | fix4 plugin-semver.cjs | dsh app-boot（includePrerelease） | 结果 |
|---|---|---|---|---|
| dsh-peak-chip | 2（credentials、settings） | 全部 true | 全部 true | PASS |
| dsh-subagent-model-picker | 5（jobs、llm、subagent、system-prompt、tools） | 全部 true | 全部 true | PASS |
| dsh-session-health | 7（client-ui-conversation、client-ui-primitives、commands、session、session-projection、tools、typert-protocol） | 全部 true | 全部 true | PASS |

修正后三个插件重新做了加载测试（用 0.2.1-rc.2 官方包）：全部 LOAD_OK。`plugin-patches.diff` 已更新。

**dsh-our-free-model（原样）的提示**：它的 `engines.dsh`（`^0.1.2-rc.1 || ^0.1.7-rc.2 || ^0.2.0-rc.1 || ^0.2.0`）在 fix4 的 plugin-semver.cjs 里对 0.2.1-rc.2 为 false，只影响插件管理页上的“版本不匹配”提示文字（`plugin-lifecycle.py` 的 `package_metadata`，只是显示，不阻止加载）；真正的启动门槛 app-boot 用 includePrerelease，判通过。所以保持原样，只是界面可能出现这行提示，属预期。

注：第 2.1 表中写的“peer 加 `|| ^0.2.0-rc.1`”已被本节取代，以 `^0.2.1-rc.1` 为准。
