# dsha021-build-fix4 🔧

| [中文](README.md) | [English](README.en.md) | [MIT License](LICENSE) |
|---|---|---|

> ⚠️ **Unofficial test modifications, unaffiliated with the upstream authors. No APKs in the repo (the release attachments are debug-signed APKs), and no backups, keys or accounts are included. Use at your own risk.**
> Version: **dsha-fix4-dsh0.2.1-rc.2** (core `@deepseek-ai/dsh 0.2.1-rc.2`).
> Unrelated to `dsha-build` and `dsha-021-test` under 133563825as-ai: this is an independent, unofficial fix4 patch set.

- **Base:** dsha-build tag `v0.2.25-dsh0.2.1-rc2-test`, commit `54062749b44239f59d17eaa710f698eb0793025a` (Gradle's `versionName` carries the upstream 0.2.25 string; needed only to build).
- **Contents:** cumulative patch `patches/dsha-fix4-dsh0.2.1-rc.2.patch` (verified with `git apply --check` and `git apply` on the base), modified files at fix4 state, plugin diffs, reports, `SHA256SUMS`, `AGENTS.md`. No upstream tree. Excluded from the patch: an upstream test fixture and the binary `libdsha-session.so`.
- **fix4 changes:** Web port probe never aborts startup on `BindException` (falls back to a dynamic port); recovery-agent failures report a redacted reason; maintenance text says a real phone reboot is required after the checkpoint; blocked Web starts are logged.
- **Not verified on a real device:** all of fix4, including the managed "update runtime" step required after install; the root cause of EPERM; the relaxed process-identity check kept in the cumulative code; plugin runtime behaviour. Builds are debug-signed.
- **Rollback:** earlier test builds are no longer provided. Reverse-apply the patch and rebuild; Android blocks versionCode downgrades, so uninstall first (back up data).
- **License:** MIT, see `LICENSE`, `NOTICE`, `licenses/` (upstream copyright notices retained).
