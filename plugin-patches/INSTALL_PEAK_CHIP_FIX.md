# dsh-peak-chip 圆点位置修正（样式小补丁）— 尚未在手机上验证

改了什么（只改 lib/client.js 的显示位置，功能不变）：
1. 圆点放进「对话/轨迹」那一行，紧跟最后一个标签，固定左间距 8px（原来是 margin-left:auto 推到行尾）。
2. 找不到「对话/轨迹」那一行时，不再浮到屏幕右下角（固定定位可能压住页头）：圆点先隐藏，每秒重试一次，找到就挂回去。
3. 点按不再出现浏览器默认的方块高亮（-webkit-tap-highlight-color: transparent）。
细节见 peak-chip-fix1.diff。

安装：
1. 先确认 Web 能打开，并按需做一次插件备份。
2. 把手机里的 /root/.dsh/plugin-src/dsh-peak-chip 整个文件夹换成本仓库补丁修改后的 dsh-peak-chip 文件夹（先把旧文件夹改名成 dsh-peak-chip.bak 保存）。
3. 点「重启 Web」，等 Web 打开，截图发回。

回退：关闭 dsh-peak-chip，把 dsh-peak-chip.bak 改回 dsh-peak-chip，点「重启 Web」。（或用插件备份恢复。）

未验证：真机上的实际位置、是否真的解决了重叠。
