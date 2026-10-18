# DSHA fix4报告

相对基线的累计改动。所有原生库（jniLibs）与大型离线资产（rootfs、python、tools、dsh-runtime）未改动。

## 改动
1. **WebPortPolicy.available**：任何 `BindException`（含 EPERM）都当作“该端口不可用”，`choose()` 回退到动态端口(0)，探测错误不再中止启动。非 bind 类 IOException 仍抛出。新增单测（EPERM → 不可用 → 动态端口；其它 IOException 照常抛出）。
2. **应急 DSH 工具检查**：`recovery-agent.js` 的 catch 现在输出 `DSHA_RECOVERY_AGENT_FAILED:<错误名: 消息>`（单行、去控制字符、遮蔽令牌/密钥/长十六进制、截断 300 字符）；`RecoveryController.agentFailure` 把它显示为 `RECOVERY_TOOL_CHECK_FAILED: <详情>`（错误码解析不变，仍被识别为 RECOVERY_TOOL_CHECK_FAILED）。新增 2 个单测。
3. **维护文案**：首次写入检查点时附带内部标记，界面显示“检查点已保存，需真正重启手机后再点‘恢复中断维护’；重启会断开 shell/会话，文件不丢。”恢复/安全逻辑未改，只是异常消息多了一个后缀标记；原有 27 个 BoundedGuestBootRecovery 测试全部通过。
4. **WebLifecycleController**：因未完成维护而拒绝启动、且没有 onStatus 回调时，也写 `Log.w` 与诊断日志（WEB_START_STOP）。

## 应急 DSH 失败原因：源码里是否明显？
**不明显，未改逻辑。** 证据：03:06:31 STARTING → 03:06:36 FAILED，只隔约 5 秒，说明不是 `/v1/diagnostics` 的 5 秒超时（那样至少要在加载完成后再多等 5 秒），而是 roster / preset / workspace / diagnostics 响应之一很快抛错。agent 原来吞掉了错误，本版会把名字和消息带到应急页和诊断里。

## 重要：fix4 会触发一次“运行时更新”
runtimeId 由 `launcherInputs`（包含被改动的 Java 源文件哈希）决定，已按构建门禁重新生成：`3360fcf1…`（fix3）→ `2c6e28f5…`（fix4）。已安装的运行时与新描述不一致，**覆盖安装后需要在“安装与修复”页点“更新运行时”（受管更新，保留数据）**；在此之前 Web 启动会提示需要更新。这是正式版升级的标准路径，但在这台手机上从未跑过，**无法在盒子里模拟**。应急运行时 id 也变了（`0128bd51…` → `d7e00a55…`），应急舱会重新解包一次（数据文件不变，空间充足）。
不要在更新途中强退或重启。

## 已验证（盒子内）
- 构建：Standard、Low 均 BUILD SUCCESSFUL；全量 JUnit 1717 例，仅 `AdbWheelPathsTest.externalBindingsForeignHostsAndLinkCyclesNeverCreateCache` 失败（盒子环境原因，fix1 起就失败，与本改动无关）；新增/相关测试均通过。
- `prepare-runtime-descriptor --check`、descriptor 构建门禁、recovery 资产映射（recovery_apk_assets.archive_locations，两个 APK）通过；agentSha256 与 APK 内 recovery-agent.js 相符。
- aapt2：versionCode 178；Standard minSdk30 / Low minSdk23；arm64-v8a。apksigner 验证通过。
- 对 fix3 的 APK 逐条目比对：仅 classes3–7.dex、recovery-agent.js、recovery-runtime.json、recovery-asset-locations.json、runtime-descriptor.json、AndroidManifest（版本）变化（Low 另含签名文件）；无增删条目；jniLibs、大型资产不变。
- `failureDetail` 用 Node 24 实测：令牌、`Bearer`、`sk-` 被遮蔽，长度 ≤300，换行折叠。
- 冷安装探针、rootfs 离线命令的 qemu/proot 模拟（fix3 轮）所用代码与资产未变，无需重做。

## 未验证（只能手机上）
- “更新运行时”这条受管更新路径在本机（proot）上的实际表现。
- EPERM 为何出现（SELinux/ROM）仍不清楚；本修复只保证不中止启动，改用动态端口（02:32 的启动证明动态端口在 guest 内可绑定）。
- 应急 DSH 是否就绪取决于真实失败原因，要看下一次诊断里的详情。

## 风险与建议
- 风险：更新运行时流程（中）；应急 DSH 仍可能失败，但会给出具体原因（低风险，信息性）。
- **Go/No-Go：有条件 GO。** Web 启动 EPERM 的修复把握约 80%；诊断/文案类改动把握很高但不解决根因。若老板再次下载，请告知“安装后需点一次更新运行时、期间别重启”。
- 备选（不下载）：把 Web 端口改为固定端口（如 18080）可绕过当前 EPERM，约 55% 把握。
