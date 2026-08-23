# pi 粘贴图片失效 —— niri + Kitty + Noctalia5 完整排查与避坑

> **场景**：在 **niri（Wayland 平铺合成器）** 会话下，用 **Kitty** 终端跑 **pi**，按 `Ctrl+V` 想要粘贴一张图片到对话里，结果怎么也粘不进去（只能贴文本）。
> **结论先行**：pi 在 Wayland 下读剪贴板图片，是**直接调用系统里的 `wl-paste` 二进制**。这台机器上 `wl-paste`（属于 `wl-clipboard` 包）**根本没安装**，只有 X11 的 `xclip`，而 xclip 在 Wayland 下读不到图。这是根因。
> **状态**：已修复，可正常粘贴。**装 `wl-clipboard` 一个包就够了。**

写于**2026年8月23日-星期日-北京时间22：11：44**

---

## 1. 环境信息（复现此问题的系统配置）

| 项目 | 值 | 说明 |
| --- | --- | --- |
| 发行版 | Kali GNU/Linux Rolling `2026.3`（`ID_LIKE=debian`，apt 系） | 装包用 `apt` |
| 会话类型 | `wayland`（`XDG_SESSION_TYPE=wayland`, `WAYLAND_DISPLAY=wayland-1`） | 关键前提：pi 走 Wayland 分支 |
| 桌面环境 | **niri**（`XDG_CURRENT_DESKTOP=niri`） | niri 是 wlroots 系，容易出问题 |
| 桌面壳 | **Noctalia v5**（`/usr/bin/noctalia`，PID 1305 在跑） | 自带剪贴板历史（加密存储） |
| X11 兼容 | `DISPLAY=:0`，由 **xwayland-satellite** 提供 | X11 剪贴板与 Wayland **不互通** |
| 终端 | **Kitty**（`TERM=xterm-kitty`），外层 shell 是 **zsh** | Kitty 支持图片协议，**不是终端的问题** |
| 修复前 | 有 `/usr/bin/xclip`，**无 `wl-paste`** | xclip 读取的是 X11 剪贴板，读不到 Wayland 剪贴板 |
| 修复后 | `/usr/bin/wl-paste` | 装了 `wl-clipboard` |

> 期间有个**误判**要交代：我最初用 `ps ... | grep -iE "noctalia|cliphist|q"` 判断"Noctalia 没在跑"，结果 `|q` 把一堆 `kworker` 内核进程也匹配进来了，判断失误。用干净的 `pgrep -a noctalia` 一查——**Noctalia 其实一直在跑**（PID 1305）。**排查进程一定要用 `pgrep`/精确匹配，别在超大正则里混一个宽泛字符。**

---

## 2. 问题现象

- 在 pi 里按 `Ctrl+V`（Windows 为 `Alt+V`），剪贴板明明是图片，却**只塞进来文本或空**。
- 换其他对话/会话也一样。
- **关键**：Kitty 本身没问题（它支持 Kitty 图片协议、能渲染图片）；卡的是"pi **能不能把图读进来**"这一步。

---

## 3. 诊断命令（按顺序执行，附输出解读）

### 3.1 确认会话类型与合成器

```bash
echo "$XDG_SESSION_TYPE $XDG_CURRENT_DESKTOP $WAYLAND_DISPLAY"
# 输出: wayland niri wayland-1   ← 证明是 Wayland，不是 X11
```

### 3.2 确认终端不是问题

```bash
echo "$TERM $TERM_PROGRAM"
# 输出: xterm-kitty          ← Kitty，支持图片协议，先排除终端
```

### 3.3 看 pi 到底能不能读到流程里的关键二进制

```bash
command -v wl-paste xclip xsel
# 修复前: /usr/bin/xclip        ← 只有 xclip，没有 wl-paste
# 修复后: /usr/bin/xclip
#         /usr/bin/wl-paste    ← wl-paste 现身 = 修复完成
```

### 3.4 查剪贴板里到底有没有图（用能用的工具试探）

```bash
xclip -selection clipboard -t TARGETS -o 2>/dev/null
# 修复前: 空/无输出 → X11 剪贴板里没图（图在 Wayland 里，xclip 看不到）
```

### 3.5 确认 Noctalia 是否在跑（不要再用带 `|q` 的正则）

```bash
pgrep -a noctalia
# 输出: 1305 noctalia          ← Noctalia 在跑，桌面壳 + 剪贴板历史是活的
```

---

## 4. 查证过程（wiki、官方文档、pi 原代码 —— 各查到了什么）

这次不是靠猜：**pi 源码、niri wiki、Noctalia 官方文档、wl-clipboard 文档**都读了一遍，才敢下结论。逐条记录查到的内容。

### 4.1 pi 原代码：读图逻辑的实锤（最权威）

读了 `dist/utils/clipboard-image.js` 和 `dist/utils/clipboard-native.js` 两个文件。

`clipboard-native.js` 说明 pi 用什么方式读剪贴板：

```js
const hasDisplay = process.platform !== "linux" || Boolean(process.env.DISPLAY || process.env.WAYLAND_DISPLAY);
const clipboard = !process.env.TERMUX_VERSION && hasDisplay ? loadClipboardNative() : null;
```

- 关键点：`hasDisplay` 只要 `DISPLAY` 或 `WAYLAND_DISPLAY` 任一存在即为真 —— 本机两者都有（`DISPLAY=:0` + `WAYLAND_DISPLAY=wayland-1`）。
- 所以 pi 可以加载原生模块（`@mariozechner/clipboard`），也会走后面的 `wl-paste` 分支。

`clipboard-image.js` 的逻辑（这条最核心）：

```js
if (wayland || wsl) {
    image = readClipboardImageViaWlPaste() ?? readClipboardImageViaXclip();
}
// 只有 !image 且 !wayland 时，才轮到原生模块 readClipboardImageViaNativeClipboard()
```

- **Wayland 下第一优先就是 `wl-paste`**：`readClipboardImageViaWlPaste()` 内部就是
  `runCommand("wl-paste", ["--list-types"], ...)` 和 `runCommand("wl-paste", ["--type", ...], ...)`。
- 查到这儿，答案已经浮出：**pi 就是要 `wl-paste` 这条命令。**

### 4.2 niri wiki：剪贴板与 X11 的事实

- **Configuration: Miscellaneous** 里有 `clipboard { disable-primary }`（关掉中键主剪贴板）；且 niri **自己没有内置剪贴板历史**（只有一个相关 open issue）。
- **Xwayland** 页面明确：rootful Xwayland **不跟合成器共享剪贴板**。要手动桥接：
  - `env DISPLAY=:0 xsel -ob | wl-copy`（X → niri）
  - `wl-paste -n | env DISPLAY=:0 xsel -ib`（niri → X）
- 这条直接解释了**为什么从 X11 应用复制的图，`wl-paste` 读不到**：图压根不在 Wayland 剪贴板里。

### 4.3 Noctalia 官方文档：剪贴板历史是它自己做的（纠正了一个误判）

读了两页：`/noctalia/configuration/shell/` 和 `/noctalia/compositor-settings/niri/`，外加 clipboard 插件页。

查到的关键配置：

```
clipboard_enabled = true                      # false 只禁用历史/面板，基础复制粘贴仍在
clipboard_keep_from_closed_apps = true        # 源应用退出后，由 Noctalia 接管选区保活
clipboard_history_max_entries = 100
clipboard_auto_paste = "auto"                 # image→Ctrl+V / text→Ctrl+Shift+V
clipboard_image_action_command = ""           # 图片附加动作（如 satty -f -）
```

- **重要纠正**：Noctalia 的剪贴板历史是**自带加密**的（密钥来自 Noctalia 主密钥，经 Secret Service / 钥匙圈 `org.freedesktop.secrets` 存储），**完全不依赖 cliphist**。我之前从一条二手搜索摘要误信"Noctalia 靠 cliphist"，一读官方文档就推翻了。
- **Noctalia 官方文档里从头到尾没有出现 cliphist / wl-paste 字样**——它只是个用来查看历史的面板，**不提供** pi 需要的 `wl-paste`。

### 4.4 wl-clipboard 文档：`wl-paste`/`wl-copy` 到底怎么工作

读了项目 README + `wl-clipboard(1)` man page，确认两者行为截然不同：

| 命令 | 默认行为（man page 原话） | 含义 |
| --- | --- | --- |
| `wl-paste` | "pastes once and exits" | 读一次剪贴板就退，只读、瞬时 |
| `wl-copy` | "forks and serves data requests in the background" | 写入后**变成后台选区拥有者**，直到别人接管 |

- `--watch`：让 `wl-paste` 常驻、每次剪贴板变化都跑一次命令（**这就是会与 Noctalia 冲突的入口**）。
- `--paste-once`（`wl-copy` 用）：只服务一次粘贴就退出，适合敏感内容。
- `--sensitive`：标记内容敏感，剪贴板管理器可能不落盘。
- 这一页直接支撑了第 7 节的判断。

### 4.5 查证方法复盘（这次的教训）

| 方法 | 结果 | 教训 |
| --- | --- | --- |
| `ps | grep -iE "noctalia | cliphist | q"` | 误判"Noctalia 没跑"（` | q` 命中一堆 kworker） | 用 `pgrep -a noctalia` 精确匹配 |
| 二手搜索摘要"Noctalia 靠 cliphist" | 含误 | **读一手官方文档**（docs.noctalia.dev）才作数 |
| 直接 fetch niri wiki / Noctalia docs / wl-clipboard man | 可靠 | 一手来源 > 转述 |

> **总结**：能读到一手源码 / 官方文档的，绝不靠转述。这一次每条结论 —— 根因=缺 `wl-paste`、Noctalia 自带历史、`wl-paste` 不冲突 —— 都能在一手来源中找到依据。

---

## 5. 根因分析（读 pi 源码得到的实锤）

pi 的图片粘贴**不是终端递图**，而是 pi **自己 fork 一个命令去读剪贴板**。源码在 `dist/utils/clipboard-image.js`，Wayland 分支：

```js
// Linux / Wayland 下：
if (wayland || wsl) {
    image = readClipboardImageViaWlPaste() ?? readClipboardImageViaXclip();
}
```

而 `readClipboardImageViaWlPaste()` 内部就是直接执行系统里的 `wl-paste`：

```js
runCommand("wl-paste", ["--list-types"], ...)                     // 列出剪贴板类型
runCommand("wl-paste", ["--type", selectedType, "--no-newline"])   // 按类型取字节
```

**完整因果链**：

```
按下 Ctrl+V
→ pi 看出是 Wayland，先尝试 readClipboardImageViaWlPaste()
→ 系统里没有 wl-paste 这个二进制   ←★根因
→ 该路径返回 null，回退 readClipboardImageViaXclip()
→ xclip 读的是 X11 剪贴板（经 xwayland-satellite），
  而图在 Wayland 合成器(→ Noctalia 保活)手里
→ xclip 也读不到 → 整体返回 null
→ pi 认定"没有图片" → Ctrl+V 退化成纯文本粘贴 → 看似"粘不进图"
```

> **一句话根因**：pi 依赖 `wl-paste` 这个命令行工具去"取"图片，本机没有它；回退的 `xclip` 在 Wayland 下又读不到图，所以读图链路整条断掉。

---

## 6. 修复步骤（一个包搞定）

```bash
# 1) 装 wl-clipboard（提供 wl-paste / wl-copy）
sudo apt install wl-clipboard

# 2) 验证
command -v wl-paste          # → /usr/bin/wl-paste

# 3) 重启 pi（或新开会话）
exit && pi

# 4) 从 Wayland 原生应用复制一张图（重要！），在 pi 里 Ctrl+V
```

**为什么只装这一个包就够了**（逻辑链）：

1. pi 读图 = 调用 `wl-paste` 二进制（源码实锤）。
2. 会话是 Wayland；所以优先走 `wl-paste`，`xclip` 只是失效的回退。
3. 本机没有 `wl-paste` → 读图链路断裂。
4. **能提供 `wl-paste` 的，只有 `wl-clipboard` 这一个包**。Noctalia 不提供（它自带的历史是加密存储，不是 CLI 读图工具）；cliphist 是"需要 wl-paste 才能跑"的另一个历史管理器，不是提供者；niri 本身不提供 CLI。
5. 装完后"仍可能失败"的原因，逐一排查：
   - 剪贴板里得真是图片、且来自 Wayland 原生应用（X11 应用的图 `wl-paste` 读不到）→ **使用前提**，不是缺包。
   - 当前模型得支持图片输入（`input: ["text","image"]`）→ **模型层面**，与装包无关。
   - "来源程序退出后才粘贴"需要持久化（Noctalia/cliphist）→ 只影响**边角场景**，正常"复制完立刻 Ctrl+V"时来源还活着，`wl-paste` 直接问它要数据即可。
6. 所以对"让 pi 能粘图"这个目标：装 `wl-paste` 既是**必要**也是**充分**。

---

## 7. 与 Noctalia5 剪贴板的关系与冲突判断

两者不同，不可混淆：

| 谁 | 角色 | 关键依据 |
| --- | --- | --- |
| **Noctalia5 剪贴板** | 一个用来**查看历史的面板**（文本/图片/文件 `1/2/3` 切换、搜索、置顶、内联预览） | 官方 shell 配置：`clipboard_enabled`、`clipboard_keep_from_closed_apps`、`clipboard_history_max_entries`，历史**加密存盘**（Secret Service 钥匙圈）。全文**无 cliphist / wl-paste 字样** |
| **pi 的读图** | 就是**执行 `wl-paste`** 去取一次图，瞬时只读 | pi 源码 `clipboard-image.js` |

**会不会冲突？基本不会，** 依据 `wl-clipboard` man page：

- `wl-paste` 默认 **"pastes once and exits"**（读一次就退），只是向合成器**查询**当前选区。**它不拥有选区、不写剪贴板、不常驻、不记历史** → 对 Noctalia 构不成任何抢夺，反而互补（Noctalia 保活、wl-paste 读取）。
- 真正会产生冲突的只有两种情况，都**不会遇到**：
  1. **主动运行 `wl-copy`** 去写 → 它变成**新选区拥有者**（后台进程）。但 Wayland 剪贴板同时只有一个拥有者，**后写者生效**——这是正常的 last-writer-wins 语义，不是 bug。
  2. **运行 `wl-paste --watch <cmd>`** 当第二个历史记录器 → 才会跟 Noctalia 历史**重复 + 竞争**。但 pi 用的是**不带 `--watch`** 的普通 `wl-paste`，所以这场景不会发生。

> **结论**：装 `wl-clipboard`、让 pi 用普通 `wl-paste`，不会与 Noctalia 冲突。**唯一要避免的是：别再加 cliphist、别开 `wl-paste --watch`。**

---

## 8. 避坑清单（分享给别人的重点）

### 7.1 我一开始说"Noctalia 靠 cliphist"是**错的**

那是二手搜索摘要的说法，直接读 Noctalia 官方 shell 配置文档（`docs.noctalia.dev/noctalia/configuration/shell/`）会发现**根本没提 cliphist**。它历史是**自带加密**的（Secret Service）。**别为"喂给 Noctalia"去装 cliphist，它不需要。**

### 7.2 别叠第二个剪贴板管理器

Noctalia 已自带历史 + "来源退出保活"（`clipboard_keep_from_closed_apps` 默认开）。再叠 cliphist 或 `wl-paste --watch ... cliphist store` = **两个管理器** → 历史重复、选区竞争、且会踩下面那条 Firefox bug。

### 7.3 niri 下 Firefox 复制图片可能卡死

**Mozilla bug 1942284**：在 niri 里从 Firefox 复制图片、粘贴到别处或**剪贴板管理器活跃**时，Firefox 可能卡死。剪贴板管理器越多越容易触发。所以保持**只有 Noctalia 一个管理器**。

### 7.4 X11 应用复制的图，`wl-paste` 读不到

`wl-paste` 读 **Wayland** 剪贴板。niri 用 xwayland-satellite，rootful Xwayland **不同步剪贴板**（niri wiki 明确）。从 X11 应用复制 → 图在 X11 侧 → `wl-paste` 抓空。**从 Wayland 原生应用复制**（Wayland 截图、Wayland 模式的浏览器）才行。

### 7.5 Noctalia 历史要落盘必须有钥匙圈

历史加密存盘依赖 Secret Service（`org.freedesktop.secrets`，如 gnome-keyring）。若 Noctalia 启动时钥匙圈还没起（常见竞争场景），历史只在当前会话有效，等钥匙圈接管后自动恢复，或用 **Settings → Shell → Clipboard → Retry**。

### 7.6 安全：`clipboard_keep_from_closed_apps` 会破坏"清空式"密码管理器

默认 `true` 让 Noctalia 在来源退出后**继续保活**那份选区。若使用"复制后退出即清空剪贴板"的密码管理器，这一条会**让它失效**。有此类需求可设 `clipboard_keep_from_closed_apps = false`。

### 7.7 排查进程，用 `pgrep`，别在超大正则里混宽泛字符

本次教训：`ps | grep -iE "noctalia|cliphist|q"` 里那个 `|q` 匹配到了全部 `kworker/R-*` 内核进程，导致我误判"Noctalia 没跑"。用 `pgrep -a noctalia` 一次到位。

---

## 9. 通用判断流程（下次 5 分钟定位）

```
pi 按 Ctrl+V 粘不进图片
│
├─ echo $XDG_SESSION_TYPE
│   ├─ 不是 wayland → 按 X11 看（一般 terminal 直接支持）
│   └─ 是 wayland → 继续 ↓
│
├─ command -v wl-paste
│   ├─ 没有 → 装 wl-clipboard（本手册第 6 节），并确认重启 pi
│   └─ 有 → 继续 ↓
│
├─ 复制一张图(必须来自 Wayland 原生应用)后:
│   wl-paste --list-types
│   ├─ 列出 image/png 等 → 图在 → 检查模型是否支持图片输入
│   └─ 空/无 → 图不在 Wayland 剪贴板(或来源是 X11) → 换 Wayland 应用复制
│
└─ 依然不行 → 检查是否有 `wl-paste --watch`/第二个历史管理器
    └─ 有 → 停掉，只留 Noctalia 一个
```

---

## 10. 参考链接

- pi 源码（读图逻辑）：`dist/utils/clipboard-image.js` · `dist/utils/clipboard-native.js`
- wl-clipboard 项目（`wl-copy`/`wl-paste`）：<https://github.com/bugaevc/wl-clipboard>
- Noctalia 官方文档 · Shell 配置：<https://docs.noctalia.dev/noctalia/configuration/shell/>
- Noctalia 官方文档 · Niri 合成器设置：<https://docs.noctalia.dev/noctalia/compositor-settings/niri/>
- niri wiki · Xwayland（剪贴板不互通）：<https://github.com/niri-wm/niri/wiki/Xwayland>
- niri wiki · Configuration: Miscellaneous（`clipboard { disable-primary }`）：<https://github.com/niri-wm/niri/blob/main/docs/wiki/Configuration:-Miscellaneous.md>
- Mozilla Bug 1942284（niri 下 Firefox 复制图片冻结）：<https://bugzilla.mozilla.org/show_bug.cgi?id=1942284>

## 11. 相关 Wiki 与 GitHub 仓库（汇总）

### Wiki / 官方文档

| 名称 | 地址 | 用途 |
| --- | --- | --- |
| niri 官方 Wiki | <https://github.com/niri-wm/niri/wiki> | 合成器行为、配置说明 |
| niri · Xwayland | <https://github.com/niri-wm/niri/wiki/Xwayland> | 剪贴板不互通、手动桥接 |
| niri · Configuration: Miscellaneous | <https://github.com/niri-wm/niri/blob/main/docs/wiki/Configuration:-Miscellaneous.md> | `clipboard { disable-primary }` 等 |
| Noctalia 官方文档 | <https://docs.noctalia.dev/> | 桌面壳配置、剪贴板历史 |
| Noctalia · Shell 配置 | <https://docs.noctalia.dev/noctalia/configuration/shell/> | `clipboard_keep_from_closed_apps` 等 |
| Noctalia · Niri 合成器设置 | <https://docs.noctalia.dev/noctalia/compositor-settings/niri/> | 如何在 niri 启用 Noctalia |

### GitHub 仓库

| 名称 | 地址 | 用途 |
| --- | --- | --- |
| niri | <https://github.com/niri-wm/niri> | Wayland 平铺合成器 |
| Noctalia | <https://github.com/noctalia-dev/noctalia> | Wayland 桌面壳 |
| wl-clipboard | <https://github.com/bugaevc/wl-clipboard> | `wl-paste` / `wl-copy`（本次安装的包） |
| cliphist | <https://github.com/sentriz/cliphist> | 独立剪贴板历史管理器（**本次不建议装**） |
| BassttElSevic/-Linux- | <https://github.com/BassttElSevic/-Linux-> | 本调试日志所属仓库 |

> 本文最相关的两个：**niri wiki**（确认剪贴板/X11 事实）与 **Noctalia 官方文档**（纠正"Noctalia 靠 cliphist"的误判）。

---

*记录时间：2026-08-19 · 环境：Kali Rolling / niri / Kitty / Noctalia5 / Wayland*
*结论：**pi 在 Wayland 下要能粘贴图片，只需 `sudo apt install wl-clipboard`；别叠 cliphist、别开 `wl-paste --watch`，否则才会与 Noctalia 冲突。***

<img width="500" height="500" alt="lilbasstt" src="https://github.com/user-attachments/assets/cfc31e08-f0e0-4784-a59e-677e6b1c011d" />

