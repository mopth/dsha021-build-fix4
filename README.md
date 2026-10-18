# dsha021-build-fix4 🔧

| [中文](README.md) | [English](README.en.md) | [MIT License](LICENSE) |
|---|---|---|

> ⚠️ **测试性声明：这是非官方的测试修改，与上游作者无关，未经官方审核；仓库里没有 APK（发布页附件含调试签名 APK），也没有任何备份、密钥或账号信息。风险自负。**
> 当前版本：**dsha-fix4-dsh0.2.1-rc.2**（内核 `@deepseek-ai/dsh 0.2.1-rc.2`）。
> 与 133563825as-ai 下的 `dsha-build`、`dsha-021-test` 无关：本仓库是独立的非官方 fix4 补丁。

## 📌 基于哪个版本
- 基线：dsha-build，tag `v0.2.25-dsh0.2.1-rc2-test`，提交 `54062749b44239f59d17eaa710f698eb0793025a`（gradle 的 `versionName` 里自带上游版本号 0.2.25，仅为构建所需，不是本仓库的命名）。
- 内核：`@deepseek-ai/dsh 0.2.1-rc.2`（已由 fix4 的 `package.json`、`package-lock.json`、运行时描述核对）。
- 许可：基线与内核均为 MIT，见 [LICENSE](LICENSE)、[NOTICE](NOTICE)、`licenses/`；上游版权声明原样保留。

## 📦 仓库内容
- `patches/dsha-fix4-dsh0.2.1-rc.2.patch`：累计补丁（基线 → fix4），只含我们的改动，不含上游整仓源码。已对基线实际执行 `git apply --check` 与 `git apply`，均无冲突。
- `modified-sources/`：补丁涉及文件在 fix4 状态下的版本（只有被改动的文件）。
- `plugin-patches/`：插件适配 diff（仅 diff）与 peak-chip 圆点位置小补丁。
- `reports/`：fix4 报告、插件适配报告。
- `SHA256SUMS`：本仓库全部文件校验（`sha256sum -c SHA256SUMS`）。
- `AGENTS.md`：给 AI 助手的协作说明。
- 补丁排除：上游测试夹具 `dsh-client-ui-settings-client.js` 与二进制 `libdsha-session.so`（需按源码自行构建）。

## ✨ fix4 改了什么
1. **Web 端口探测**：任何 `BindException`（含 EPERM）视为端口不可用，回退动态端口，不再中止启动；其它 IOException 照常抛出。
2. **应急 DSH 工具检查**：失败时输出具体原因（`DSHA_RECOVERY_AGENT_FAILED:…`，令牌/密钥已遮蔽，≤300 字符），界面显示 `RECOVERY_TOOL_CHECK_FAILED: <详情>`。
3. **维护文案**：首次写检查点后提示“检查点已保存，需真正重启手机后再点‘恢复中断维护’”。
4. **WebLifecycleController**：因未完成维护拒绝启动时写日志。
- 累计内容还包含：LAN 设置补丁锚点适配 dsh 0.2.1-rc.2、bxroot 预检失败回退 proot、V2 会话启动器守卫等。
- 副作用：运行时 ID 变化，覆盖安装后需点一次“更新运行时”（受管更新，保留数据），期间勿强退/重启。

## 🧪 未在真机验证
- **fix4 没有任何真机验证记录。**
- “更新运行时”受管更新路径从未在真机跑过。
- EPERM 的根因未明，仅保证不中止启动；应急 DSH 失败的真实原因待下次诊断。
- 累计内容中放宽了“进程出生身份复核”（/proc、getsid 被拒时容忍），安全性弱于原设计，仍在代码中，待评估是否收紧。
- bxroot 稳定性、运行模式切换、web-mobile 界面、插件运行期、peak-chip 圆点实际位置均未验证。
- 构建产物为 **debug 签名**，不是上游正式签名，无法覆盖安装正式版。

## ⏪ 回滚
- 更早的测试构建不再提供。
- 回到基线：`git apply -R patches/dsha-fix4-dsh0.2.1-rc.2.patch`，然后重新构建。
- Android 不允许 versionCode 降级覆盖安装：需先卸载（会清数据，先备份）。
- 插件：把插件目录换回 `.bak` 备份并重启 Web。

## 🛠 构建说明
1. 取基线：dsha-build 的 tag `v0.2.25-dsh0.2.1-rc2-test`（该提交不在 DSH-APP/DSHA 公开历史中）。
2. `git apply patches/dsha-fix4-dsh0.2.1-rc.2.patch`。
3. 按上游 `BUILD.md` 构建 Standard / Low debug 变体；因启动器输入哈希变化，需重新生成 runtime-descriptor（`prepare-runtime-descriptor`）。
4. `libdsha-session.so` 与测试夹具需从基线/上游获取。

## 📝 更新记录
- fix4：本版内容，见上。
- 此前的 fix1–fix3 已并入累计补丁：适配 dsh 0.2.1-rc.2 的 LAN 补丁、bxroot→proot 回退与 accept 等待、V2 会话启动器替换。

## 🙏 致谢
基于 dsha-build / DSH-APP/DSHA（© qiannianhuanxiang，MIT）与 deepseek-harness（© DeepSeek，MIT）。

## 📄 License
MIT —— 见 [LICENSE](LICENSE) 与 [NOTICE](NOTICE)。
