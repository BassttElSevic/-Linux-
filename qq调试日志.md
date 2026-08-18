# QQ Flatpak 空白窗口问题调试日志

- 记录日期：2026-08-19
- 应用：`com.qq.QQ`（Flatpak，Flathub 渠道）
- 应用版本：3.2.32（flatpak 构建号 51802，commit `8487d0758`，2026-08-09 构建）
- 运行时：`org.freedesktop.Platform/x86_64/25.08`
- Flatpak 版本：1.18.0（系统级安装）
- 主机环境：Kali（主机名 kalidragon），Wayland 会话（`XDG_SESSION_TYPE=wayland`），合成器为 **niri**，X11 由 **xwayland-satellite** 提供（DISPLAY=:0）
- GPU：Intel UHD Graphics 620（Kaby Lake GT2），Mesa 26.1.5，硬件加速可用
- 输入法：fcitx5（日志中可见其输入窗口）

---

## 一、时间线总览

| 时间 | 事件 |
| --- | --- |
| 08-12 19:03 | 用户安装/首次运行 flatpak 版 QQ 3.2.32（构建 51802），热更新检查结果为"无需更新"，运行正常 |
| 08-13 00:03 | 再次运行，热更新检查仍为"无需更新"，正常 |
| 08-19 01:06 | xwayland-satellite 启动（提供 X11 显示 :0） |
| 08-19 03:24:15 | QQ 启动，热更新检查发现服务器推送新构建 3.2.32-52194 |
| 08-19 03:25 | 下载 52194 安装包（约 376MB，实际为 179MB 的 zip.zip + 解压产物） |
| 08-19 03:28:01 | 热更新完成，`appRelaunch` 自动切换到 52194 构建运行 |
| 08-19 03:29:05 | 用户运行 `flatpak run com.qq.QQ`，界面空白，日志异常（本次问题报告起点） |
| 08-19 03:37:35 | 应用 X11 override 后重新运行，GLib 错误依旧，确认与 X11/Xvfb 分支无关 |
| 08-19 03:39 | 尝试各种截屏手段，均无法确认画面内容；进程分析发现无 renderer 进程 |
| 08-19 03:43:28 | 移走 versions 目录后重启，加载 flatpak 自带 51802 构建，出现真实渲染活动（DroppedFrame） |
| 08-19 03:43:43 | 热更新器立即重新下载 52194（status: DOWNLOADING） |
| 08-19 03:44 | 尝试 chmod 555 锁定 versions 目录，导致 launcher 抛异常、主窗口无法出现，方案失败 |
| 08-19 03:47:07 | 采用 onErrorVersions 方案，更新器跳过 52194，不再下载，QQ 稳定运行于 51802 构建，问题解决 |

---

## 二、现象（用户报告，03:29:05）

用户运行 `flatpak run com.qq.QQ`，终端输出关键片段：

```
Adding default flags...
Pass `--enable-wayland-ime --wayland-text-input-version=3 --ozone-platform=wayland` to main process...
X11 socket is not available, using Wayland + Xvfb...
not mini app.
[preload] succeeded. /app/extra/QQ/resources/app/major.node
[I] <MMKV.cpp:233::initializeMMKV> root dir: /home/bassttelsevic/.var/app/com.qq.QQ/config/QQ/global/nt_data/mmkv
[17:0819/032907.750786:ERROR:content/browser/browser_main_loop.cc:291] GLib-GObject: invalid unclassed type '(null)' in class cast to '(null)'   （共 8 条）
[17:0819/032916.816053:ERROR:components/dbus/xdg/request.cc:165] Request ended (non-user cancelled).
[17:0819/032916.816165:ERROR:ui/base/accelerators/global_accelerator_listener/global_accelerator_listener_linux.cc:287] Failed to call BindShortcuts (error code 5).
```

用户反馈："为什么我的 flatpak 版本的 QQ 在更新后出现了这个问题？"（实际症状为窗口空白）

### 对日志的初步解读

- `X11 socket is not available, using Wayland + Xvfb...` —— 来自 flatpak 打包者写的启动脚本，见下文分析。
- `GLib-GObject: invalid unclassed type '(null)'` ×8 —— 浏览器主进程 GTK 初始化阶段报错，当时怀疑与渲染失败相关（后证实为无害噪声）。
- `Request ended (non-user cancelled)` —— xdg-desktop-portal 的请求被非用户操作取消（与 QQ 新版更新检查器调用 portal 有关）。
- `Failed to call BindShortcuts (error code 5)` —— 全局快捷键 portal（org.freedesktop.portal.GlobalShortcuts）调用失败，niri 未实现该 portal，导致 QQ 的全局截图快捷键（Ctrl+Alt+A）无法注册。此为 Wayland 下已知限制，与本次问题无关。

---

## 三、第一阶段排查：X11 检测逻辑与 fallback-x11 权限

### 检查方法

```bash
flatpak info com.qq.QQ            # 查看应用版本、运行时、commit、安装位置
echo "$DISPLAY $WAYLAND_DISPLAY $XDG_SESSION_TYPE"   # 查看会话环境变量
ls /tmp/.X11-unix/                # 主机侧 X11 socket 是否存在
cat /var/lib/flatpak/app/com.qq.QQ/x86_64/stable/active/files/bin/qq   # 读取启动脚本
cat /var/lib/flatpak/app/com.qq.QQ/x86_64/stable/active/metadata       # 读取权限声明（finish-args）
```

说明：`flatpak info` 用于确认版本与来源；`cat .../bin/qq` 直接读取打包者编写的启动脚本源码，这是判断"X11 socket is not available"消息来源的最直接手段；`metadata` 文件是 flatpak 应用沙箱权限的声明清单。

### 检查结果

1. 主机侧：`DISPLAY=:0`、`WAYLAND_DISPLAY=wayland-1`、`/tmp/.X11-unix/X0` 存在（主机有 X11）。
2. 启动脚本（`/app/bin/qq`）核心逻辑：

```sh
if [ -z "$(ls -A /tmp/.X11-unix 2>/dev/null)" ]; then
  if [ -n "$WAYLAND_DISPLAY" ]; then
    FLAGS+=(--ozone-platform=wayland)
    echo "X11 socket is not available, using Wayland + Xvfb..."
    exec xvfb-run zypak-wrapper /app/extra/QQ/qq "${FLAGS[@]}" "$@"
  ...
```

即：**在沙箱内**检查 `/tmp/.X11-unix` 是否为空；为空且存在 Wayland 时，用 `xvfb-run` 套一层虚拟 X 屏并以 `--ozone-platform=wayland` 启动。

3. 沙箱权限：`sockets=fallback-x11;pulseaudio;wayland`。

### 沙箱内实测（关键验证）

```bash
flatpak run --command=sh com.qq.QQ -c 'echo "DISPLAY=[$DISPLAY] WAYLAND=[$WAYLAND_DISPLAY] X11dir=[$(ls -A /tmp/.X11-unix 2>/dev/null)]"'
```

说明：`--command=sh -c` 是在**不改动任何配置**的前提下，进入该应用沙箱执行一条命令，用于观察沙箱内部的真实环境。

实测结果：`DISPLAY=[] WAYLAND=[wayland-1] X11dir=[]` —— **沙箱内 DISPLAY 被清空、X11 socket 不存在**。

### 结果解读

对照 Flatpak 官方文档（sandbox-permissions）：`--socket=fallback-x11` 的语义是"Show windows using X11, **if Wayland is not available**"（仅在 Wayland 不可用时才暴露 X11）。用户会话是 Wayland，因此 Flatpak 1.18 不会把 X11 socket 挂进沙箱，也不传递 DISPLAY。

### 结论

- `X11 socket is not available, using Wayland + Xvfb...` 是启动脚本在该沙箱环境下的**必然结果**，属设计行为，本身不是故障点。
- 该逻辑自 2025-12 起存在（flathub 仓库 commit `a522449`/`c53e10f`，为规避 Wayland 剪贴板崩溃问题 #212 引入 xvfb 包装）。
- 顺带说明：GLib-GObject 错误在两种启动分支下都存在，初步怀疑与渲染失败相关，但后续证伪（见第四阶段）。

---

## 四、第二阶段：应用 X11 override，验证假设

### 检查方法

```bash
flatpak override --user --socket=x11 com.qq.QQ
```

说明：`override` 为用户级覆盖应用权限，`--socket=x11` 显式授予 X11 socket。因为 manifest 用的是 `fallback-x11`（会被显式 `x11` 覆盖），这样沙箱内就会挂载 X11 并传递 DISPLAY，启动脚本会走正常的 XWayland 分支。

```bash
flatpak override --user --show com.qq.QQ          # 确认覆盖已生效
flatpak run --command=sh com.qq.QQ -c '...'       # 再次探沙箱
```

### 检查结果

沙箱内变为 `DISPLAY=[:0] WAYLAND=[wayland-1] X11dir=[X0]`，override 生效。

### 用户复测（03:37:35）

```
Adding default flags...
Pass `--enable-wayland-ime --wayland-text-input-version=3` to main process...
[preload] succeeded. ...
[2:0819/033737.947511:ERROR:content/browser/browser_main_loop.cc:291] GLib-GObject: ...
```

- `X11 socket is not available` 消息消失，说明已走正常分支；
- **GLib-GObject 错误仍然存在** —— 证伪了"该错误由 Xvfb/Wayland 混合分支导致"的假设，它只是 Chromium 在沙箱内 GTK 初始化时的固定噪声。

### 进程与窗口分析（关键发现）

```bash
ps aux | grep -iE "qq|QQ|Xvfb"        # 查看所有 QQ 相关进程
ps -eo pid,lstart,cmd | grep ...      # 带启动时间的完整进程
niri msg windows                      # 合成器视角的窗口列表
DISPLAY=:0 xwininfo -root -tree       # X11 视角的窗口树
ps -eo cmd | grep -- '--type=renderer'   # 是否有渲染进程
```

说明：`niri msg windows` 是 niri 合成器的 IPC 查询命令，用于看 Wayland 侧实际有哪些窗口；`xwininfo -root -tree` 列出 X server 上的窗口层级；`--type=renderer` 是 Chromium/Electron 渲染进程的命令行特征，**有 renderer 进程才有页面绘制**。

发现：

1. 存在**两个 QQ 实例**：03:29 启动的旧实例（其 `Xvfb :99` 已变孤儿进程，主进程已退出）与 03:37 的新实例。
2. niri 窗口列表中只有一个 "QQ" 窗口，其 PID 1456 实为 **xwayland-satellite 自身**（说明该窗口是经 xwayland-satellite 呈现的 X11 窗口）。
3. X server :0 上存在 930x995 的 "QQ" 主窗口，Map State 为 IsViewable（窗口已映射）。
4. **新实例只有主进程、zygote、network utility，没有任何 `--type=renderer` 进程，也没有 GPU 进程。**

### 结果解读

- 窗口对象存在且已映射，但 **renderer 进程从未被创建**。Chromium 架构中 renderer 负责解析 HTML/JS 并绘制页面，无 renderer 等于页面从未加载/绘制 —— 这从机制上解释了"空白窗口"。
- 之前怀疑的 X11/Xvfb、GLib 错误均不是空白窗口的原因。

---

## 五、第三阶段：渲染层排查（排除 GPU/GL 因素）

### 检查方法

```bash
DISPLAY=:0 import -window 0x400004 /tmp/qq_shot.png      # ImageMagick 按窗口抓图
ffmpeg -y -f x11grab -video_size 1920x1080 -i :0 -frames:v 1 /tmp/screen_qq.png   # X11 全屏抓取
niri msg action screenshot-screen --path /tmp/niri_shot.png   # niri 原生 Wayland 截图
flatpak run com.qq.QQ --enable-logging=stderr > /tmp/qq_dbg.log 2>&1    # 带日志重启
flatpak run com.qq.QQ --use-gl=desktop --enable-logging=stderr > /tmp/qq_dbg2.log 2>&1  # 换 GL 后端重启
setsid bash -c '...' < /dev/null > /dev/null 2>&1 &       # 后台脱离终端运行，避免随 shell 退出
```

说明：`import` 走 XGetImage 抓取指定窗口；`ffmpeg x11grab` 抓取整个 X 显示；两者都**不能**作为最终判断依据——Chromium 在 XWayland 下走 DRI3 直通，像素不经 X server 帧缓冲，抓回来往往是空的。`niri msg action screenshot-screen` 是合成器级截图（最可靠），但本机未成功输出文件。`--enable-logging=stderr` 让 Chromium 把内部日志输出到终端以便分析。`--use-gl=desktop` 强制使用系统 GL 而非 Chromium 自带 ANGLE。`setsid` 使后台进程脱离当前会话，避免命令结束时被一并终止。

### 检查结果

- 所有 X11 截图文件极小（1.5KB / 8.7KB），基本为空白像素，但如前所述不可作准。
- 调试日志新增关键行：

```
[2:0819/034043.397602:WARNING:electron/shell/browser/electron_browser_main_parts.cc:232] Wayland+XWayland detected; defaulting to --ozone-platform=x11 (workaround for #48859).
[46:...] GetVSyncParametersIfAvailable() failed for 1 times!
```

说明：Electron 40 检测到"Wayland + XWayland 同时存在"时强制走 X11（Electron issue #48859 的 workaround）；GPU 进程（进程号 46）确实启动过，`GetVSyncParametersIfAvailable` 失败只是垂直同步参数查询失败，会自动降级，无害。

- 轮询 24 秒：renderer 进程数始终为 0；`--use-gl=desktop` 同样为 0。

### 结果解读

排除 GPU/GL/垂直同步层面问题。问题收敛到：**浏览器主进程创建了窗口，但从未 spawn renderer**，即主进程 JS 侧"创建窗口后、加载页面之前"的环节出了问题。

---

## 六、第四阶段：转折 —— 发现 QQ 自带热更新（hotUpdate）

### 检查方法

```bash
ls -la ~/.var/app/com.qq.QQ/config/QQ/versions/                 # QQ 热更新版本目录
cat ~/.var/app/com.qq.QQ/config/QQ/log/app_launcher-20260819.log   # QQ 启动器自身日志
cat ~/.var/app/com.qq.QQ/config/QQ/log/updater.log              # QQ 热更新日志
```

说明：Flatpak 应用的持久化数据在 `~/.var/app/<应用ID>/` 下。QQ 自带的热更新机制会把下载的版本放在 `config/QQ/versions/`，并把运行过程写进 `log/` 下的自有日志。**排查应用自身日志是定位此类问题的关键一步**。

### 检查结果

versions 目录内容（03:25 生成）：

```
3.2.32-52194/          解压后的新版本
3.2.32-52194.zip       （376MB）
3.2.32-52194.zip.zip   实际下载的安装包（179MB）
config.json  channel.json  setting.json
```

app_launcher 日志（03:37:36）：

```
[INFO] use path /home/bassttelsevic/.var/app/com.qq.QQ/config/QQ/versions/3.2.32-52194/application.asar
[INFO] 从版本目录加载主进程代码: .../versions/3.2.32-52194/application.asar/background.js
```

updater.log 关键记录：

```
[08-12 19:03] [QQ hotUpdate] ----- startAutoUpdate curVersion: 3.2.32-51802 -----
[08-12 19:03] [QQ hotUpdate] [startAutoUpdate] 无需更新
[08-13 00:03] [QQ hotUpdate] [startAutoUpdate] 无需更新
[08-19 03:24:15] [QQ hotUpdate] hotUpdateApi start check  3.2.32-52194  IsOnErrorVersion
[08-19 03:25] 下载 3.2.32-52194.zip.zip
[08-19 03:28:01] [QQ hotUpdate] hotUpdateApi appRelaunch doUpdate suc
```

### 结果解读（核心结论）

1. Flatpak 内置构建为 **3.2.32-51802**；腾讯服务器在 08-19 推送了更新构建 **3.2.32-52194**。
2. QQ 的 hotUpdate 机制在 03:25 自动下载、03:28 自动重启切换到 52194。
3. **用户实际运行的是 52194 构建**（而非 flatpak 自带的 51802），该构建在本环境渲染空白（主进程创建窗口但从不创建 renderer）。
4. 这解释了"更新后出现问题"：用户感知的"更新"实际是腾讯服务器侧推送的热更新，而不是 flatpak 本身的更新。
5. 与 flathub 打包无关：8 月 12、13 日检查均为"无需更新"，运行正常；8 月 19 日服务器端出现新构建后才触发。

### 验证（对照组实验）

```bash
pkill -9 -f 'extra/QQ/[q]q'      # 结束所有 QQ 进程（[q] 为防自匹配写法，见附录）
mv ~/.var/app/com.qq.QQ/config/QQ/versions ~/.var/app/com.qq.QQ/config/QQ/versions.hotupdate-bak   # 移走热更新目录
```

说明：`mv` 把整个热更新目录移走（保留备份），QQ 启动时版本目录不存在，启动器会回退到 flatpak 安装目录加载代码。

结果：重启后 app_launcher 显示 `从安装目录加载主进程代码: /app/extra/QQ/resources/app/application.asar`（回到 51802），随后日志出现 **`DroppedFrame` 帧活动**、X 窗口树出现 **`资料卡`（403x440）等真实 UI 窗口**——**基础构建渲染正常**，证实 52194 是问题版本。

副作用：热更新器检测到版本缺失后**立即重新下载**（`onStatusChanged status: DOWNLOADING`），说明仅移走目录不能阻止复发。

---

## 七、第五阶段：阻止热更新的两次尝试

### 尝试 A：只读锁定 versions 目录（失败）

```bash
rm -rf ~/.var/app/com.qq.QQ/config/QQ/versions && mkdir -p ~/.var/app/com.qq.QQ/config/QQ/versions && chmod 555 ~/.var/app/com.qq.QQ/config/QQ/versions
```

说明：清空后用 `chmod 555` 将目录设为只读，意图是让 QQ 无法写入下载包。

结果：启动日志出现：

```
(node:2) UnhandledPromiseRejectionWarning: Error: EACCES: permission denied, open '.../versions/config.json'
    at Launcher.markErrorVersion (.../app_launcher/launcher.js)
```

解读：QQ 启动器在更新检查失败后需要写 `config.json` 记录错误版本（markErrorVersion），目录只读导致写入失败，未捕获的 Promise 拒绝使启动流程中断，**主窗口根本没有出现**。该方案失败——versions 目录必须保持可写。

### 尝试 B：利用 onErrorVersions 跳过机制（成功）

从 updater.log 观察到检查流程：`start check 3.2.32-52194 IsOnErrorVersion` → `checkIsOnErrorVersion result: false` → 继续下载。config.json 中存在 `onErrorVersions` 数组，被列入的版本会被跳过。

```bash
pkill -9 -f 'extra/QQ/[q]q'
V=~/.var/app/com.qq.QQ/config/QQ/versions
rm -rf $V && mkdir -p $V
printf '{\n  "baseVersion": "3.2.32-51802",\n  "curVersion": "3.2.32-51802",\n  "prevVersion": "",\n  "onErrorVersions": ["3.2.32-52194"],\n  "buildId": "51802",\n  "unzipRetryCount": 0\n}\n' > $V/config.json
```

说明：手工重建 versions 目录并写入 config.json，把 52194 列入 `onErrorVersions`。`printf` 用于精确写入格式化 JSON（避免 echo 的转义问题）。

### 验证结果（03:47:07 重启后）

- updater.log：`start check 3.2.32-52194 IsOnErrorVersion` → `checkIsOnErrorVersion result: true`（跳过）。
- versions 目录只有 config.json / setting.json，**无任何下载文件**。
- 加载路径：`从安装目录加载主进程代码: /app/extra/QQ/resources/app/application.asar`（51802）。
- X 窗口：主窗口 1880x995 及多个 UI 窗口，日志出现 `DroppedFrame` 帧活动，QQ 正常渲染。

---

## 八、最终解决方案

1. **保留 X11 权限覆盖**（已生效，niri 下更稳定）：

```bash
flatpak override --user --socket=x11 com.qq.QQ
```

2. **让热更新器跳过坏版本**（核心修复）：在 `~/.var/app/com.qq.QQ/config/QQ/versions/config.json` 中把 `3.2.32-52194` 写入 `onErrorVersions`（上述 printf 命令已完成）。

3. **清理热更新残留**（释放约 380MB）：

```bash
rm -rf ~/.var/app/com.qq.QQ/config/QQ/versions.hotupdate-bak
```

4. **预防复发**：若腾讯后续推送 52195 等新构建且再次导致空白，把新版本号追加进 onErrorVersions：

```bash
sed -i 's/\["3.2.32-52194"\]/["3.2.32-52194","3.2.32-52195"]/' ~/.var/app/com.qq.QQ/config/QQ/versions/config.json
```

说明：`sed -i` 原位替换 JSON 中的数组内容。

5. 需要恢复默认权限时：`flatpak override --user --reset com.qq.QQ`。

---

## 九、遗留问题与说明

| 现象 | 结论 |
| --- | --- |
| 日志中的 8 条 `GLib-GObject: invalid unclassed type` 错误 | 沙箱内 Chromium GTK 初始化的固定噪声，两种启动分支均存在，无害，无需处理 |
| `GetVSyncParametersIfAvailable() failed` | GPU 进程垂直同步参数查询失败，自动降级，无害 |
| 全局截图快捷键（Ctrl+Alt+A）无法注册 | niri 未实现 GlobalShortcuts portal，Wayland 平台限制，与本次问题无关 |
| 热更新到新版本（52195+）可能复发 | 本质是腾讯 52194 构建的缺陷（在本环境空白渲染），等待腾讯修复或 flathub 跟进后再考虑放开 |

---

## 附录：关键命令行速查

| 命令 | 作用 | 为什么这样写 |
| --- | --- | --- |
| `flatpak info com.qq.QQ` | 查看应用版本/运行时/commit | 确认问题版本的基线 |
| `flatpak run --command=sh com.qq.QQ -c '...'` | 在沙箱内执行探测命令 | 不改配置即可观察沙箱真实环境 |
| `cat .../active/files/bin/qq` | 读取启动脚本源码 | 直接确认 "X11 socket is not available" 消息来源与分支逻辑 |
| `cat .../active/metadata` | 读取沙箱权限声明 | 确认 sockets=fallback-x11 等权限 |
| `flatpak override --user --socket=x11 com.qq.QQ` | 覆盖权限，显式授予 X11 | fallback-x11 在 Wayland 会话下不暴露 X11，显式 x11 可覆盖 |
| `flatpak override --user --show com.qq.QQ` | 验证覆盖是否生效 | 防止"以为生效实则没有"的误判 |
| `ps -eo pid,lstart,cmd \| grep ...` | 带启动时间查进程 | 区分多次运行的实例，识别孤儿进程 |
| `niri msg windows` | 查询 niri 合成器窗口列表 | 确认 Wayland 侧真实可见的窗口 |
| `DISPLAY=:0 xwininfo -root -tree` | 查询 X11 窗口树 | 确认 X 侧窗口的存在与尺寸 |
| `ps -eo cmd \| grep -- '--type=renderer'` | 检查 Chromium 渲染进程 | 无 renderer = 无页面绘制 = 空白窗口的机制解释 |
| `flatpak run com.qq.QQ --enable-logging=stderr > /tmp/qq_dbg.log 2>&1` | 带 Chromium 日志重启 | 获取 renderer/GPU/平台选择等内部信息 |
| `setsid bash -c '...' < /dev/null > /dev/null 2>&1 &` | 后台脱离会话运行 | 避免后台进程随命令结束被终止 |
| `pkill -9 -f 'extra/QQ/[q]q'` | 结束所有 QQ 进程 | `[q]` 正则技巧防止 pkill 匹配到自身命令行而被误杀 |
| `tail ~/.var/app/com.qq.QQ/config/QQ/log/updater.log` | 查看 QQ 热更新自有日志 | 定位热更新下载/切换的时间与版本证据 |
| `mv versions versions.hotupdate-bak` | 移走热更新目录做对照实验 | 验证"基础构建可渲染、52194 不可渲染" |
| `chmod 555 versions` | 只读锁定（失败方案） | 验证"阻止写入"路线不可行（破坏 launcher 流程） |
| `printf '...' > versions/config.json` | 写入含 onErrorVersions 的配置 | 利用 QQ 更新器的错误版本跳过机制 |
| `sed -i 's/.../.../' versions/config.json` | 追加新版本号到 onErrorVersions | 复发时的维护手段 |

---

## 十、排查方法复盘

1. **先确认基线**：版本、运行时、会话类型、沙箱权限，避免在错误前提上推导。
2. **读源码胜过猜**：直接读 flatpak 启动脚本，确认消息来源；直接读 QQ 自身日志，发现热更新机制。
3. **沙箱内实测**：用 `--command=sh` 探针验证沙箱环境与文档语义一致（fallback-x11）。
4. **对照组实验**：移走 versions 目录证明基础构建正常、热更新构建异常，形成因果闭环。
5. **失败方案同样有价值**：chmod 555 的失败排除了"只读锁定"路线，并揭示了 launcher 依赖 config.json 可写这一约束。
6. **现象分层**：把 GLib 错误、vsync 错误、快捷键错误逐条判定为无害噪声，避免被干扰项带偏，最终收敛到真正的主因（热更新切换到坏构建 + renderer 不启动）。
