# Dolphin 在 Kali Linux + niri（Wayland）下的外观与 PDF 关联调试档案

> 场景：在 Kali Linux 的 niri Wayland 会话中使用 Dolphin（Qt6 / KF6），同时遇到两类问题：一类是外观、配色、主题接管异常；另一类是 PDF 默认打开程序无法记住，每次双击都弹选择器，甚至出现空白应用选择窗口。
>
> 结论先行：这不是单一配置项导致的孤立问题，而是两条链路分别出错后叠加出来的结果：
>
> 1. 外观问题的关键在于 Qt6 平台主题、qt6ct、Kvantum 与 Dolphin/KDE 颜色方案分层控制不一致，`qt5ct` 被错误注入只是其中一个重要问题；
> 2. PDF 打开方式问题前期确实有 MIME 关联污染，但仅修 MIME 不够；最终确认让 Dolphin 仍然记不住默认应用、甚至弹空白选择器的关键根因之一，是 niri 非 Plasma 会话缺失 `XDG_MENU_PREFIX`，导致 KDE 菜单数据库无法正确加载应用菜单。
>
> 状态：已定位并形成稳定修复方案，适合长期留档。

---

## 1. 环境信息

| 项目 | 值 | 说明 |
| --- | --- | --- |
| 发行版 | Kali Linux | 非 KDE 默认桌面场景 |
| 会话类型 | Wayland | `XDG_SESSION_TYPE=wayland` |
| 合成器 | niri | `XDG_CURRENT_DESKTOP=niri` |
| 应用 | Dolphin 26.04 | Qt6 / KF6 文件管理器 |
| Qt 侧重点 | Qt6、qt6ct、Kvantum | 决定平台主题、控件样式与配色接管 |
| KDE 侧重点 | `kdeglobals`、`dolphinrc` | 决定 KDE 全局颜色方案与 Dolphin 自身偏好 |

---

## 2. 初始现象

本次排查起点并不是单一症状，而是一组持续反复的问题：

- Dolphin 出现白字、浅底、低对比度，文本可读性很差；
- 某次手动改完配色后看似恢复，但重启 Dolphin 或重新登录后又异常；
- 主窗口、侧边栏、局部控件区域颜色风格割裂，不像一个完整主题；
- 双击 PDF 时无法稳定记住默认打开程序；
- 每次打开 PDF 都会重新弹“选择应用程序”；
- 某些情况下甚至弹出空白或异常的应用选择窗口。

其中最后一类问题的典型表现如下：

![image2](image2)

另外，在把环境变量写回 niri 配置时，还出现过因为文件末尾重复 `environment` 块而导致配置解析失败的中间问题，值得记录：

![image1](image1)

---

## 3. 先讲清楚：这几个配置文件分别管什么

本次问题容易反复，一个重要原因是多个配置层同时参与，且职责边界不同。

### 3.1 `~/.config/environment.d/80-qt6ct.conf`

这是用户级环境变量入口，适合在登录会话早期给 Qt6 应用提供统一环境。这里负责的是：

- 明确指定 `QT_QPA_PLATFORMTHEME=qt6ct`；
- 避免系统在非 KDE 会话下替 Qt 注入错误的平台主题。

### 3.2 `~/.config/niri/config.kdl`

这是 niri 的会话级配置入口，适合补充桌面环境变量，尤其是需要跟随 niri 会话启动的变量。本案例里它承担两件关键事：

- 让 niri 会话稳定带上 `QT_QPA_PLATFORMTHEME=qt6ct`；
- 显式设置 `XDG_MENU_PREFIX "gnome-"`，让 KDE/KF6 组件能找到有效的应用菜单定义。

### 3.3 `~/.config/qt6ct/qt6ct.conf`

这是 Qt6 应用通过 qt6ct 接管后读取的具体外观配置，负责：

- `style=Kvantum`：控件样式引擎用 Kvantum；
- `color_scheme_path=...Kali-Light.conf`：Qt 颜色基准走 qt6ct 指定方案；
- `standard_dialogs=kde`：标准对话框优先使用 KDE 实现。

### 3.4 `~/.config/Kvantum/kvantum.kvconfig`

这是 Kvantum 自己的主题选择配置，只负责 SVG/主题引擎层，不负责 Dolphin 的全部应用颜色偏好。这里的关键值是：

- `theme=KvMojaveLight`

### 3.5 `~/.config/kdeglobals`

这是 KDE Frameworks / KDE 应用共享的全局颜色、字体等配置入口。它控制的是 KDE 配色方案层，不等于 Qt 控件样式层，也不等于 Kvantum 主题本体。

### 3.6 `~/.config/dolphinrc`

这是 Dolphin 自身偏好配置。它可以覆盖或细化 Dolphin 的局部行为与界面选择。本案例里最关键的是：

- `[UiSettings] ColorScheme=...`

这个值会影响 Dolphin 自己选择哪套 KDE 颜色方案。如果它和当前 Kvantum 主题附带的颜色风格明显不匹配，就会出现“外层一种风格、局部又是另一种风格”的割裂感。

---

## 4. 外观与配色问题：诊断过程与修复结论

### 4.1 Dolphin 是 Qt6 / KF6 应用

Dolphin 26.04 已经处于 Qt6 / KF6 体系下，因此 Qt 平台主题、Qt6 配置工具、KDE 配色方案和 Kvantum 引擎之间的衔接是否正确，直接决定最终外观。

这也是为什么“以前某些 Qt5 经验”在这里不能直接照搬。

### 4.2 Kali 非 KDE 会话下会错误注入 `qt5ct`

本次排查确认，在 Kali 的非 KDE 会话里，`/etc/profile.d/kali-themes.sh` 会在以下条件满足时设置平台主题：

- `QT_QPA_PLATFORMTHEME` 为空；
- `XDG_CURRENT_DESKTOP != KDE`

此时它会把平台主题设成 `qt5ct`。

对于 Qt6 应用来说，这个值并不正确。它不会成为“白字问题的唯一根因”，但它确实是主题接管链错误中的重要一环：Qt6 应用没有走到正确的 Qt6 主题接管路径，后续又叠加 qt6ct、Kvantum、KDE/Dolphin 配色分层控制不一致，就容易出现白字、浅底和重启后状态漂移。

更准确的表述应当是：

> `qt5ct` 被错误注入到 Qt6 应用的环境里，是本次主题接管失败的重要问题之一；但真正造成最终异常观感的，是 Qt6 平台主题选择错误与 qt6ct、Kvantum、KDE 颜色方案分层不一致共同叠加的结果。

### 4.3 正确修复：明确让 Qt6 走 `qt6ct`

稳定修复的关键是不要再依赖系统在非 KDE 会话里的默认推断，而是主动在 niri 与 `environment.d` 中设置：

```ini
QT_QPA_PLATFORMTHEME=qt6ct
```

这样可以保证 Dolphin 这类 Qt6 应用在会话启动时就走对平台主题接管链。

### 4.4 `qt6ct` 的稳定组合

本案例最终稳定配置集中在 `~/.config/qt6ct/qt6ct.conf`：

```ini
style=Kvantum
color_scheme_path=/usr/share/qt6ct/colors/Kali-Light.conf
standard_dialogs=kde
```

含义分别是：

- `style=Kvantum`：Qt6 控件绘制交给 Kvantum；
- `color_scheme_path=...Kali-Light.conf`：给 Qt 提供亮色基准配色；
- `standard_dialogs=kde`：优先使用 KDE 对话框实现，避免文件选择器等组件行为再分叉。

### 4.5 安装并启用 Kvantum，主题定为 `KvMojaveLight`

确认安装并启用 Kvantum 后，`~/.config/Kvantum/kvantum.kvconfig` 中应为：

```ini
theme=KvMojaveLight
```

这一步决定的是控件样式引擎层的外观。

### 4.6 `dolphinrc` 里的颜色方案也要对齐

本案例中，`~/.config/dolphinrc` 的 `[UiSettings] ColorScheme` 一度仍指向 `KvCurves`，而 Kvantum 当前启用的却是 `KvMojaveLight`。这就导致：

- 控件外观层按 `KvMojaveLight` 绘制；
- Dolphin 自身/KDE 颜色方案层却继续按 `KvCurves` 取色；
- 结果就是主窗口、侧栏、局部面板颜色不一致。

因此需要把：

```ini
[UiSettings]
ColorScheme=KvCurves
```

调整为：

```ini
[UiSettings]
ColorScheme=KvMojaveLight
```

这里也要避免写成绝对规则。更准确的说法是：

> Dolphin 颜色方案并不是“必须永远与 Kvantum 主题同名”，但如果某个 Kvantum 主题本身附带视觉匹配的 KDE `.colors` 方案，优先对齐同名项通常最协调。本案例里 `KvMojaveLight` 与 `KvCurves` 的不一致，确实直接造成了视觉割裂。

### 4.7 为什么 Dolphin 内置“窗口配色方案”不是完整主题切换入口

Dolphin 自带的“窗口配色方案”菜单，能改的是 Dolphin 自身所使用的 KDE 颜色方案层；它不会同步切换：

- Qt6 平台主题；
- qt6ct 的全局接管方式；
- Kvantum 当前启用的主题引擎。

因此，在 `qt6ct + Kvantum + KDE 颜色方案` 这种组合里，Dolphin 内置菜单只能算局部调色入口，不能当成完整主题切换入口。只改这里而不改 Kvantum 或 qt6ct，就很容易出现“看起来改了，但重启后还是不对”“只有一部分变了”的现象。

### 4.8 如果目标只是稳定亮色，先用 `qt6ct + Fusion` 验证

如果后续想继续排查某个主题是否有兼容性问题，建议先简化变量：

1. 先保持 `QT_QPA_PLATFORMTHEME=qt6ct`；
2. 在 `qt6ct` 里把样式临时切到 `Fusion`；
3. 先验证 Dolphin 是否能稳定保持正常亮色和对比度；
4. 确认基础链路没问题后，再叠加 Kvantum。

这样可以快速区分：

- 是 Qt6 平台主题接管链本身还有问题；
- 还是 Kvantum 与具体颜色方案组合出了问题。

---

## 5. PDF 默认打开程序问题：诊断过程与修复结论

### 5.1 早期确实存在 MIME 关联污染

最开始检查 `mimeapps.list` 时，`application/pdf` 一度被伪本地 desktop 项污染，出现过：

- `atril-2.desktop`
- `atril-3.desktop`
- `atril-4.desktop`
- `atril-5.desktop`
- `atril-6.desktop`
- `atril-7.desktop`

这些并不是系统标准应用桌面文件，而是后续排查中确认的“本地伪 desktop 文件”。

### 5.2 `~/.local/share/applications/atril-N.desktop` 是怎么来的

这类 `atril-N.desktop` 文件通常来自命令路径式选择器或历史记录生成的本地条目。也就是说，当用户不是从标准 `.desktop` 应用项里选程序，而是手工输入类似 `/usr/bin/atril` 这种命令路径时，系统可能生成一个本地伪 desktop 文件用于记忆这次选择。

一旦生成：

- `xdg-mime query default application/pdf`
- `gio mime application/pdf`

都有可能指向这些伪对象，而不是系统真正的 `atril.desktop`。

这会让“看起来已经设了默认打开器”，但实际上指向的是一组不稳定、可能重复、可能失真的本地记录。

### 5.3 第一步修复：清理伪 desktop，恢复真正的 `atril.desktop`

这一层的修复动作包括：

1. 删除 `~/.local/share/applications/atril-N.desktop` 这类伪本地 desktop 文件；
2. 清理 `mimeapps.list` 中 `application/pdf` 对这些伪条目的引用；
3. 将 `~/.config/mimeapps.list` 中 `application/pdf` 恢复为真正的：

```ini
application/pdf=atril.desktop;
```

这一步是必要的，因为如果默认值继续指向伪 desktop，那么 `xdg-mime` 和 `gio` 看到的默认程序本身就是错的。

### 5.4 但仅修 MIME 还不够：关键根因之一是 `XDG_MENU_PREFIX` 缺失

继续排查后发现，即便 MIME 关联已经恢复到 `atril.desktop`，Dolphin 某些情况下仍会重新弹选择器，甚至弹出空白 chooser。

关键原因之一在于：当前会话不是 Plasma，而是 niri；此时 `XDG_MENU_PREFIX` 未设置，导致 `kbuildsycoca6` 在构建 KDE 菜单数据库时查找 `applications.menu` 失败，而系统实际存在的是：

- `gnome-applications.menu`
- `xfce-applications.menu`

也就是说，MIME 层修好了，但 KDE/KF6 用来列出“可选应用”的菜单索引层仍然不完整，于是 Dolphin 侧的应用选择器依然可能行为异常。

这就是为什么不能把问题简单归因成“只是 MIME 关联错了”。

### 5.5 用临时环境变量验证根因

本次排查里，通过临时方式启动：

```bash
XDG_MENU_PREFIX=gnome- dolphin
```

可以验证 chooser 行为恢复正常。这个验证结果很关键，因为它把问题从“怀疑是 MIME 没修干净”进一步收敛到“缺菜单前缀，导致 KF6/KService 菜单数据库异常”。

### 5.6 最终修复：在 niri 配置里补上 `XDG_MENU_PREFIX`

既然问题是会话级环境缺失，最终应在 `~/.config/niri/config.kdl` 的 `environment` 块中加入：

```kdl
environment {
    XDG_MENU_PREFIX "gnome-"
}
```

这样 niri 会话每次启动时都能带上正确前缀，`kbuildsycoca6` 也就能基于现有菜单定义正确构建应用索引。

需要额外注意的是：修改时不要在文件末尾重复追加一个新的 `environment` 块，避免出现解析失败。应当把变量合并到现有结构中。

### 5.7 为什么不要再手工输入 `/usr/bin/atril`

这一点必须单独强调：

> 不能再在 Dolphin 的“选择应用程序”窗口里手工输入 `/usr/bin/atril` 作为默认应用。

原因是这样做很可能再次生成新的伪 desktop 条目，例如 `atril-8.desktop`、`atril-9.desktop`，从而把已经修好的 MIME 关联重新污染。

后续如果需要重新指定默认应用，应当：

- 直接从标准 `.desktop` 应用列表中选择；
- 或用 `xdg-mime` / `gio mime` 指向真实的 desktop ID。

---

## 6. 最终稳定配置摘要

下面是本案例最终需要稳定下来的关键文件与关键值。

### 6.1 `~/.config/niri/config.kdl`

应保证 `environment` 块中至少包含与本问题相关的两项：

```kdl
environment {
    QT_QPA_PLATFORMTHEME "qt6ct"
    XDG_MENU_PREFIX "gnome-"
}
```

### 6.2 `~/.config/environment.d/80-qt6ct.conf`

```ini
QT_QPA_PLATFORMTHEME=qt6ct
```

### 6.3 `~/.config/qt6ct/qt6ct.conf`

```ini
style=Kvantum
color_scheme_path=/usr/share/qt6ct/colors/Kali-Light.conf
standard_dialogs=kde
```

### 6.4 `~/.config/Kvantum/kvantum.kvconfig`

```ini
theme=KvMojaveLight
```

### 6.5 `~/.config/dolphinrc`

```ini
[UiSettings]
ColorScheme=KvMojaveLight
```

### 6.6 `~/.config/mimeapps.list`

```ini
[Default Applications]
application/pdf=atril.desktop;
```

同时应确认：

- 不再残留 `atril-N.desktop` 伪条目引用；
- `~/.local/share/applications/` 中没有新的 `atril-*.desktop` 污染文件。

---

## 7. 后续使用规则与避免复发

### 7.1 改 Kvantum 主题时，要同步检查 KDE 颜色方案是否视觉匹配

不是所有场景都要求名称完全一致，但如果某个 Kvantum 主题附带同名 KDE `.colors` 方案，优先使用同名项通常最协调。至少要避免像本案例这样：

- Kvantum 用 `KvMojaveLight`
- Dolphin/KDE 颜色方案仍停留在 `KvCurves`

这种明显不匹配的组合。

### 7.2 Dolphin 内置“窗口配色方案”只改 Dolphin 自身颜色方案层

它不会替你同步切换 Kvantum 主题，也不会修改 Qt6 平台主题接管方式。因此它适合做局部颜色调整，不适合当作整套主题切换入口。

### 7.3 默认打开器应通过 `.desktop` 应用项或 `xdg-mime` 设置

不要在应用选择器里手工输入 `/usr/bin/...`。这类做法容易生成伪 desktop 条目，后续又会把 `mimeapps.list` 和默认值链路污染掉。

### 7.4 如果再次出现空白 chooser，优先检查这三项

按优先级建议先查：

1. `XDG_MENU_PREFIX` 是否仍在当前会话中生效；
2. `kbuildsycoca6 --noincremental` 输出是否仍提示找不到合适的 `applications.menu`；
3. `~/.local/share/applications/` 是否又出现新的伪 desktop 文件。

### 7.5 外观再异常时，先缩回最小变量集

优先退回到：

- `QT_QPA_PLATFORMTHEME=qt6ct`
- `qt6ct + Fusion`

先验证 Qt6 基础接管链是否稳定，再逐步恢复 Kvantum 和特定颜色方案。这样最容易判断问题到底出在哪一层。

---

## 8. 建议的复查命令

```bash
echo "$XDG_SESSION_TYPE $XDG_CURRENT_DESKTOP"
echo "$QT_QPA_PLATFORMTHEME"
echo "$XDG_MENU_PREFIX"

grep -n "application/pdf" ~/.config/mimeapps.list
find ~/.local/share/applications -maxdepth 1 -type f | grep -E 'atril-[0-9]+\.desktop$'

kbuildsycoca6 --noincremental
xdg-mime query default application/pdf
gio mime application/pdf
```

如果这几项同时满足：

- `QT_QPA_PLATFORMTHEME=qt6ct`
- `XDG_MENU_PREFIX=gnome-`
- `application/pdf=atril.desktop;`
- 本地无 `atril-N.desktop`

那么本案例中的两大问题通常都能稳定消失。

---

## 9. 最终归纳

这次问题表面上像是“Dolphin 很怪”，实际上是两条配置链路同时失衡：

1. 外观链：`qt5ct` 误注入 + Qt6 / qt6ct / Kvantum / KDE 颜色方案分层不一致；
2. 应用关联链：MIME 伪 desktop 污染 + 非 Plasma 会话缺失 `XDG_MENU_PREFIX`。

真正稳定的解决方式，不是只改某一个界面选项，而是把这几层配置各自归位：

- 让 Qt6 明确走 `qt6ct`；
- 让 qt6ct、Kvantum、Dolphin 颜色方案彼此对齐；
- 让 PDF 默认程序回到真实的 `.desktop` 项；
- 让 niri 会话带上 `XDG_MENU_PREFIX=gnome-`，使 KDE 应用选择器能正确建立菜单索引。

做到这几步后，Dolphin 在 Kali Linux + niri（Wayland）下可以稳定保持正常外观，也能正确记住 PDF 默认打开程序。
