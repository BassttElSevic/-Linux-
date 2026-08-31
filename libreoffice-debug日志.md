# LibreOffice 在 Kali + niri 下的命令缺失、中文化与 Noctalia 主题不生效调试日志

> **场景**：Kali Linux + niri（Wayland）+ Noctalia v5 会话下使用 LibreOffice 26.2.4.2（**官网下载的 deb 包，不是发行版仓库版**）。连续遇到四个看起来互不相关的问题：终端里 `libreoffice` 命令不存在、AI 助手反复误判系统没装 LibreOffice、缺中文语言包、Noctalia 的 LibreOffice 主题模板勾选了却完全不生效。
>
> **结论先行**：前三个问题是**同一个原因**——官网 deb 包的命令名、包名、安装路径全都和发行版仓库版不一样，所有基于"标准布局"的检测手段一律失效。第四个问题是**两个独立缺陷叠加**：
>
> 1. Noctalia 的模板分发管道**不递归子目录**，导致 LibreOffice 模板必需的 `META-INF/manifest.xml` 从未下发到任何用户机器上（上游仓库里这个文件是存在的，是分发环节丢的）；
> 2. 模板的 `apply.sh` 用 `command -v unopkg` 检测安装，而官网 deb 版的 `unopkg` 在 `/opt/libreoffice26.2/program/` 下，**不在 PATH 里**。
>
> **状态**：四个问题全部修复并验证。第四个问题的缺陷 1 属于上游服务端问题，已单独写 issue 反馈。

---

## 1. 环境信息

| 项目 | 值 | 说明 |
| --- | --- | --- |
| 发行版 | Kali GNU/Linux Rolling 2026.3 | 基于 Debian sid |
| 会话类型 | `wayland`（`XDG_SESSION_TYPE=wayland`） | |
| 合成器 | niri 26.04 (v26.04-85-gdd75865f) | `XDG_CURRENT_DESKTOP=niri` |
| Shell 环境 | Noctalia v5.0.0 | 负责主题模板分发与应用 |
| LibreOffice | 26.2.4.2 | **官网 deb 包**，装在 `/opt/libreoffice26.2` |
| 系统 locale | `zh_CN.UTF-8` | |
| Python | 3.14.6 | `apply.sh` 的 hex→dec 转换依赖它 |
| zip | 3.0 | `apply.sh` 打包 `.oxt` 依赖它 |

---

## 2. 四个问题的现象

### 2.1 终端里 `libreoffice` 不存在

图形界面能正常打开、能用，但终端敲什么都报错：

```console
$ libreoffice
bash: libreoffice: command not found
$ lowriter
bash: lowriter: command not found
```

### 2.2 AI 助手反复误判"系统没装 LibreOffice"

多次让 AI 处理文档任务，它每次都先说"你系统里没有 LibreOffice"，然后建议先安装。而实际上装了，图形界面正在运行。

### 2.3 界面是英文，没有中文语言包

菜单全英文，按 F1 出来的帮助也是英文。

### 2.4 Noctalia 主题模板勾选了但完全不生效

Noctalia 设置 → Templates 里 `libreoffice` 已勾选，提示输出路径是：

```text
~/.local/state/noctalia/libreoffice-theme-staging/Theme_Colors.xcu
```

该文件确实生成了，但 LibreOffice 打开后配色**毫无变化**，仍是默认灰白配色。切换主题、重启 LibreOffice 都没用。

---

## 3. 贯穿前三个问题的主线：官网 deb 包的布局和仓库版完全不同

这一节是理解 2.1、2.2、2.4-B 的基础。

LibreOffice 有两种主流安装方式，布局差异很大：

| 对比项 | 发行版仓库版（`apt install libreoffice`） | 官网 deb 版（本机） |
| --- | --- | --- |
| 主命令 | `libreoffice` | **`libreoffice26.2`**（带版本号） |
| 命令位置 | `/usr/bin/libreoffice` | `/usr/local/bin/libreoffice26.2` |
| 安装目录 | `/usr/lib/libreoffice` | `/opt/libreoffice26.2` |
| 包名 | `libreoffice-core`、`libreoffice-writer`… | **`libobasis26.2-core`**、`libreoffice26.2-*` |
| `lowriter` / `localc` 等简写 | 提供 | **不提供** |
| `unopkg` | `/usr/bin/unopkg`（在 PATH） | `/opt/libreoffice26.2/program/unopkg`（**不在 PATH**） |

三处都不一样：**命令名、包名、路径**。网上绝大多数教程和脚本默认你用仓库版，所以照抄一律失败。

验证本机实际情况：

```bash
ls -d /opt/libreoffice*
# 输出: /opt/libreoffice26.2

command -v libreoffice26.2
# 输出: /usr/local/bin/libreoffice26.2

ls -l /usr/local/bin/libreoffice26.2
# 输出: ... -> /opt/libreoffice26.2/program/soffice

dpkg -l | grep -c "^ii  libobasis26.2"
# 输出: 20 个以上（core/calc/writer/impress 等模块）
```

`.desktop` 文件里的 `Exec` 也印证了这一点：

```bash
grep -h "^Exec" /usr/share/applications/libreoffice*.desktop | head -3
# 输出:
# Exec=libreoffice26.2 --base %U
# Exec=libreoffice26.2 --calc %U
# Exec=libreoffice26.2 --draw %U
```

> **这解释了 2.1 的现象**：图形界面能开，是因为 `.desktop` 里写的是 `libreoffice26.2`，而不是 `libreoffice`。图标能用和终端能用走的是两条不同的名字。

---

## 4. 问题一：终端命令缺失

### 4.1 修复：在 `~/.local/bin` 补齐命令名

`~/.local/bin` 本来就在 PATH 里，不需要 root，也不动系统文件：

```bash
cd ~/.local/bin

# 主命令
ln -sf /opt/libreoffice26.2/program/soffice libreoffice

# 各组件简写（官网包不提供，需要自己写 wrapper）
for pair in "lowriter:writer" "localc:calc" "loimpress:impress" \
            "lodraw:draw" "lobase:base" "lomath:math"; do
  name="${pair%%:*}"; mode="${pair##*:}"
  printf '#!/bin/sh\nexec /opt/libreoffice26.2/program/soffice --%s "$@"\n' "$mode" > "$name"
  chmod +x "$name"
done

hash -r   # 刷新当前 shell 的命令缓存
```

### 4.2 为什么 `lowriter` 要写 wrapper 而不是软链接

`soffice` 这个启动脚本**不根据 `argv[0]` 判断启动模式**。软链接成 `lowriter` 再执行，它不会知道你想开 Writer，只会开默认启动界面。必须显式传 `--writer` 参数，所以只能用 wrapper 脚本。

### 4.3 验证

```bash
libreoffice --version
# 输出: LibreOffice 26.2.4.2 0229ac93fcf0d7cbc6376066c6f35021cef002dc

localc --headless --convert-to xlsx test.csv --outdir .
# 输出: convert ... -> /tmp/test.xlsx using filter : Calc Office Open XML
```

---

## 5. 问题二：为什么 AI 反复误判"没装 LibreOffice"

这个问题值得单独记录，因为它不是 AI 瞎猜，而是**所有常规检测手段在这套布局下全部失效**。

AI 或者检测脚本通常用这五种方式判断 LibreOffice 是否存在。修复前的实际结果：

| 检测方式 | 仓库版 | 本机（官网版） |
| --- | --- | --- |
| `which libreoffice` | 有 | **无**（命令叫 `libreoffice26.2`） |
| `ls /usr/bin/libreoffice` | 有 | **无**（在 `/usr/local/bin`） |
| `dpkg -l libreoffice-core` | 有 | **无**（包名 `libobasis26.2-core`） |
| `ls /usr/lib/libreoffice` | 有 | **无**（在 `/opt/libreoffice26.2`） |
| `snap list` / `flatpak list` | — | **无**（都没装） |

五种全部返回"没有"。AI 拿到这个结果，报告"未安装"是合理推论，问题在于**没有一种通用检测能覆盖官网 deb 版**。

### 5.1 副作用：4.1 的修复顺带治好了这个问题

建完软链接后再测：

```bash
which libreoffice
# 输出: /home/bassttelsevic/.local/bin/libreoffice   ← 现在能查到

apt list --installed 2>/dev/null | grep -c "^libreoffice"
# 输出: 非 0
```

### 5.2 顺带发现一个残包

检测过程中发现一个异常组合：

```bash
dpkg -l libreoffice-writer >/dev/null 2>&1 && echo 有 || echo 无
# 输出: 有

dpkg -l libreoffice-core >/dev/null 2>&1 && echo 有 || echo 无
# 输出: 无
```

装了仓库版的 `libreoffice-writer`，却没有 `libreoffice-core`。这是个孤立残包，可能是历史上装了一半或被依赖带进来的。目前不影响使用，但如果以后 Writer 行为异常，可以清掉避免两个版本互相干扰：

```bash
sudo apt purge libreoffice-writer
```

---

## 6. 问题三：中文字体与中文语言包

### 6.1 先查清缺什么

```bash
fc-list :lang=zh-cn | wc -l
# 输出: 45   ← 基础中文字体是有的

dpkg -l | grep -iE "libobasis.*(zh|langpack)" | awk '{print $2}'
# 输出: 空   ← 确认没有任何中文语言模块

grep AllLanguages /opt/libreoffice26.2/program/versionrc
# 输出: AllLanguages=en-US   ← 只装了英文
```

结论：中文**字体**基础可用（Noto CJK 全套在），缺的是文泉驿等常用字体，以及 Windows 字体名的映射；中文**语言包**完全没有。

### 6.2 装字体

```bash
sudo apt install --no-install-recommends \
  fonts-wqy-zenhei fonts-wqy-microhei fonts-noto-cjk-extra \
  fonts-lxgw-wenkai fonts-smiley-sans fonts-arphic-ukai
```

各包用途：

| 包 | 用途 |
| --- | --- |
| `fonts-wqy-zenhei` / `-microhei` | 文泉驿正黑/微米黑，老牌开源中文黑体，兼容性好 |
| `fonts-noto-cjk-extra` | 思源全字重（原本只有 Regular 和 Bold） |
| `fonts-lxgw-wenkai` | 霞鹜文楷，用来接楷体 |
| `fonts-smiley-sans` | 得意黑，标题用 |
| `fonts-arphic-ukai` | 楷体兜底 |

装完中文字体从 45 个到 111 个。

### 6.3 关键一步：配置 Windows 字体名映射

只装字体不够。别人发来的 `.docx` 里写的字体名是 `宋体`、`SimSun`、`微软雅黑` 这些 Windows 字体，Linux 上没有，会出现豆腐块或整段替换成西文字体导致排版错乱。

需要用 fontconfig 建立映射。写 `~/.config/fontconfig/conf.d/99-zh-win-aliases.conf`：

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<fontconfig>
  <!-- 宋体 -> 思源宋体 -->
  <match target="pattern"><test name="family"><string>SimSun</string></test>
    <edit name="family" mode="prepend" binding="strong"><string>Noto Serif CJK SC</string></edit></match>
  <match target="pattern"><test name="family"><string>宋体</string></test>
    <edit name="family" mode="prepend" binding="strong"><string>Noto Serif CJK SC</string></edit></match>

  <!-- 微软雅黑 / 黑体 -> 思源黑体 -->
  <match target="pattern"><test name="family"><string>Microsoft YaHei</string></test>
    <edit name="family" mode="prepend" binding="strong"><string>Noto Sans CJK SC</string></edit></match>
  <match target="pattern"><test name="family"><string>微软雅黑</string></test>
    <edit name="family" mode="prepend" binding="strong"><string>Noto Sans CJK SC</string></edit></match>
  <match target="pattern"><test name="family"><string>SimHei</string></test>
    <edit name="family" mode="prepend" binding="strong"><string>Noto Sans CJK SC</string></edit></match>

  <!-- 楷体 -> 霞鹜文楷 -->
  <match target="pattern"><test name="family"><string>KaiTi</string></test>
    <edit name="family" mode="prepend" binding="strong"><string>LXGW WenKai</string></edit></match>
  <match target="pattern"><test name="family"><string>楷体</string></test>
    <edit name="family" mode="prepend" binding="strong"><string>LXGW WenKai</string></edit></match>

  <!-- 默认族兜底，避免 sans-serif 落到无中文的西文字体 -->
  <match target="pattern"><test name="family"><string>sans-serif</string></test>
    <edit name="family" mode="append" binding="weak"><string>Noto Sans CJK SC</string></edit></match>
  <match target="pattern"><test name="family"><string>serif</string></test>
    <edit name="family" mode="append" binding="weak"><string>Noto Serif CJK SC</string></edit></match>
</fontconfig>
```

刷新并验证：

```bash
fc-cache -f
fc-match SimSun
# 输出: NotoSerifCJK-Regular.ttc: "Noto Serif CJK SC" "Regular"
fc-match 楷体
# 输出: LXGWWenKai-Regular.ttf: "霞鹜文楷" "Regular"
```

### 6.4 装中文语言包（官网版必须版本号精确匹配）

官网 deb 版的语言包要单独下载，而且**版本必须和主程序完全一致**，否则装不上或行为异常。

本机主程序是 `26.2.4.2-2`，对应官网发布目录是 `26.2.4`：

```bash
# 官网 download.documentfoundation.org 在国内基本连不上，用清华镜像
BASE="https://mirrors.tuna.tsinghua.edu.cn/libreoffice/libreoffice/stable/26.2.4/deb/x86_64"

mkdir -p /tmp/lo_zh && cd /tmp/lo_zh
curl -fLO "$BASE/LibreOffice_26.2.4_Linux_x86-64_deb_langpack_zh-CN.tar.gz"
curl -fLO "$BASE/LibreOffice_26.2.4_Linux_x86-64_deb_helppack_zh-CN.tar.gz"

for f in *.tar.gz; do tar xzf "$f"; done

# 解出来的 deb 版本号应当与已装主程序一致
sudo dpkg -i ./LibreOffice_26.2.4.2_Linux_x86-64_deb_langpack_zh-CN/DEBS/*.deb \
             ./LibreOffice_26.2.4.2_Linux_x86-64_deb_helppack_zh-CN/DEBS/*.deb
```

装完确认：

```bash
dpkg -l | grep -E "zh-cn" | awk '{print $2, $3}'
# 输出:
# libobasis26.2-zh-cn       26.2.4.2-2
# libobasis26.2-zh-cn-help  26.2.4.2-2
# libreoffice26.2-zh-cn     26.2.4.2-2

ls /opt/libreoffice26.2/program/resource/ | grep zh
# 输出: zh_CN
```

系统 locale 是 `zh_CN.UTF-8`，所以界面会自动切中文。如果没切，手动设：`工具 → 选项 → 语言设置 → 语言 → 用户界面`。

### 6.5 端到端验证

构造一个引用了三种 Windows 字体名的文档转 PDF，再提取文字和渲染成图检查：

```bash
libreoffice --headless --convert-to pdf t.fodt --outdir .
pdftotext t.pdf -
# 输出: 中文全部正常提取，没有乱码

pdffonts t.pdf
# 输出: BAAAAA+SourceHanSerifSC-Regular  ← 宋体映射到思源宋体，成功嵌入
```

再 `pdftoppm` 转图目视确认：中文全部正常显示，**无豆腐块**。

> **一个需要知道的局限**：渲染结果里三行的字形都是宋体，也就是说 LibreOffice 没有按 fontconfig 把"微软雅黑"渲染成黑体、"楷体"渲染成楷体。原因是 **LibreOffice 有自己的字体替换逻辑，优先级高于 fontconfig**。
>
> 影响：中文能正常显示（这是最关键的，不会有豆腐块），但 Windows 专有字体的**字形区分**不会自动生效。如果确实需要区分，在 `工具 → 选项 → LibreOffice → 字体` 里勾选"应用替换表"并手动添加规则。日常阅读文档不影响。

---

## 7. 问题四：Noctalia 的 LibreOffice 主题模板不生效

这个问题最复杂，是**两个独立缺陷叠加**。

### 7.1 先搞清模板的工作流

读 `template.toml`：

```toml
[templates.libreoffice]
input_path = "Theme_Colors.xcu"
output_path = "$XDG_STATE_HOME/noctalia/libreoffice-theme-staging/Theme_Colors.xcu"
post_hook = "bash '{{ config_dir }}/apply.sh'"
```

所以完整链路是：

```text
Noctalia 渲染 Theme_Colors.xcu（颜色为 hex 占位符）
→ 写到 libreoffice-theme-staging/（注意是 staging，暂存）
→ post_hook 调 apply.sh
→ apply.sh 把 hex 转成十进制（LibreOffice 的 ColorScheme schema 要求十进制整数）
→ 连同静态文件打包成 noctalia-theme.oxt
→ 用 unopkg 安装成 LibreOffice 扩展
→ LibreOffice 里出现名为 "Noctalia" 的配色方案
```

关键点：**Noctalia 只负责渲染出半成品**。staging 目录里那个文件里的颜色还是 hex（如 `e5bb66`），LibreOffice 根本不认。真正干活的是 `apply.sh`。文件开头的注释也写明了这点。

所以"staging 文件已生成"完全不代表主题已应用。

### 7.2 定位：apply.sh 根本没跑完

检查 `apply.sh` 的产物目录：

```bash
D=~/.local/state/noctalia/community-templates/libreoffice
find "$D/build"
# 输出:
# .../build/pkg/Paths.xcu
# .../build/pkg/description.xml
# .../build/pkg/META-INF          ← 空目录
# .../build/pkg/Theme_Colors.xcu
# .../build/pkg/pkg-description.en
```

两个决定性证据：

1. `build/pkg/META-INF/` 是**空目录**；
2. **整个 `build/` 里没有 `.oxt` 文件**。

对照 `apply.sh` 的源码行号：

```bash
19:  mkdir -p "$BUILD_DIR/pkg/META-INF"                          ← 执行了（目录存在）
44:  cp "$CONFIG_DIR/META-INF/manifest.xml" ".../META-INF/..."   ← 在这里失败
47:  (cd "$BUILD_DIR/pkg" && zip -qr "$OXT_PATH" .)              ← 从未执行（无 .oxt）
```

而脚本第 6 行是：

```bash
set -euo pipefail
```

`set -e` 的含义是任何命令失败立刻退出。所以第 44 行的 `cp` 一失败，脚本当场中止，后面的打包和安装全都没跑。

再验证扩展确实没装上：

```bash
/opt/libreoffice26.2/program/unopkg list
# 输出: 空（没有任何 noctalia 扩展）
```

### 7.3 缺陷 A：`META-INF/manifest.xml` 在分发环节被丢弃

第 44 行为什么失败？因为本地模板目录里没有这个文件：

```bash
ls ~/.local/state/noctalia/community-templates/libreoffice/
# 输出里没有 META-INF 目录

ls -ld ~/.local/state/noctalia/community-templates/libreoffice/META-INF
# 输出: 没有那个文件或目录
```

**但上游仓库里这个文件是存在的。** 克隆 `noctalia-dev/community-templates` 后：

```bash
find community-templates/libreoffice -type f
# 输出包含: libreoffice/META-INF/manifest.xml
```

也就是说，文件不是上游漏写的，而是**分发过程中丢的**。继续查本地缓存的文件清单：

```bash
# 本地 catalog 里 libreoffice 声明了几个文件
python3 -c "
import json
d=json.load(open('$HOME/.local/state/noctalia/community-templates/catalog.json'))
items=[t for t in (d if isinstance(d,list) else d.get('templates',d).values() if isinstance(d.get('templates',d),dict) else d) ]
" 2>/dev/null

# 更直接地看模板自己的缓存元数据
cat ~/.local/state/noctalia/community-templates/libreoffice/.noctalia-cache.json
# files 里只有 9 项：Paths.xcu / README.md / Theme_Colors.xcu / apply.sh /
# description.xml / light-screenshot.png / pkg-description.en /
# screenshot.png / template.toml
# ← 没有 META-INF/manifest.xml（上游是 10 个文件）
```

再查服务端返回的数据，确认不是本地缓存的问题：

```bash
curl -s https://api.noctalia.dev/templates | python3 -c "...筛选 libreoffice..."
# 输出: 同样只有 9 个文件，也没有 META-INF/manifest.xml
```

最后一条决定性证据——**整个 catalog 里，带子目录路径（含斜杠）的文件数是 0**：

```bash
# 遍历 catalog.json 所有模板的所有 files，统计 name 里含 "/" 的
# 输出: 含子目录的文件数: 0
```

而上游仓库里**唯一**带子目录的模板正是 libreoffice：

```bash
find community-templates -mindepth 3 -type f -not -path "*/.git*" -not -path "*/.github/*"
# 输出: libreoffice/META-INF/manifest.xml   ← 只有这一个
```

结论：**Noctalia 的模板 catalog 生成/分发只收集模板目录顶层文件，不递归子目录。** LibreOffice 是唯一依赖子目录文件的模板，所以它是唯一受影响的，而且**受影响的是所有启用该模板的用户，不是本机特有问题**。

补充一点：仓库的 CI 校验脚本 `validate-templates.py` 第 237 行用的是 `rglob("*")`，是递归的，所以 CI 能看到这个文件、不会报错。递归能力的缺失只在 catalog/分发这一侧，这也解释了为什么这个问题能一直没被发现。

### 7.4 缺陷 B：`command -v unopkg` 检测不到官网 deb 版

`apply.sh` 第 80 行和第 89 行是这样判断 LibreOffice 是否存在的：

```bash
80:  if command -v unopkg >/dev/null 2>&1; then          # 原生安装
89:  if flatpak info org.libreoffice.LibreOffice ...     # Flatpak 安装
99:  echo "... no LibreOffice installation found ..." >&2
```

本机的实际情况：

```bash
command -v unopkg
# 输出: 空   ← 不在 PATH

ls /opt/libreoffice26.2/program/unopkg
# 输出: 存在

flatpak info org.libreoffice.LibreOffice
# 输出: 失败（没装 Flatpak 版）
```

两个分支都不成立，走到第 99 行报 "no LibreOffice installation found"。

也就是说，**即使缺陷 A 修好了，脚本也会在安装环节失败**。两个缺陷是串联的，必须都修。

模板自己的 README 第 85 行其实承认了这一点：

> "Not verified live on this machine, only Flatpak LibreOffice is installed here."

作者只在 Flatpak 版上测过，原生安装这条代码路径从未实测。

### 7.5 修复

**第一步：补上缺失的 manifest**

```bash
D=~/.local/state/noctalia/community-templates/libreoffice
mkdir -p "$D/META-INF"
cat > "$D/META-INF/manifest.xml" <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<manifest:manifest xmlns:manifest="http://openoffice.org/2001/manifest">
  <manifest:file-entry manifest:full-path="Theme_Colors.xcu" manifest:media-type="application/vnd.sun.star.configuration-data"/>
  <manifest:file-entry manifest:full-path="Paths.xcu" manifest:media-type="application/vnd.sun.star.configuration-data"/>
</manifest:manifest>
EOF
xmllint --noout "$D/META-INF/manifest.xml" && echo XML 合法
```

内容与上游仓库里的那份语义一致（声明两个 `.xcu` 为 `configuration-data` 类型）。

**第二步：把 `unopkg` 接进 PATH**

```bash
ln -sf /opt/libreoffice26.2/program/unopkg ~/.local/bin/unopkg
hash -r
command -v unopkg
# 输出: /home/bassttelsevic/.local/bin/unopkg
```

**第三步：关掉 LibreOffice 再跑 apply.sh**

这一步不能省。`apply.sh` 第 64-66 行有保护逻辑：检测到 `soffice.bin` 在运行就跳过安装。原因写在注释里——在 LibreOffice 运行时执行 `unopkg add --force` 被实测会破坏已有安装（先删掉旧注册、再安装失败，最后什么都不剩）。

```bash
pgrep -x soffice.bin && echo "还在跑，先关掉"

bash ~/.local/state/noctalia/community-templates/libreoffice/apply.sh
echo "退出码: $?"
# 输出: 退出码: 0

ls -lh ~/.local/state/noctalia/community-templates/libreoffice/build/*.oxt
# 输出: noctalia-theme.oxt  3.0K   ← 这次生成了

unzip -l .../noctalia-theme.oxt
# 输出包含: META-INF/manifest.xml  ← 关键文件在包里
```

**第四步：确认扩展注册成功**

```bash
unopkg list
# 输出:
# Identifier: dev.noctalia.libreoffice.theme
#   Version: 1.0.0
#   is registered: yes
#   bundled Packages: {
#       .../Theme_Colors.xcu   is registered: yes
#       .../Paths.xcu          is registered: yes
#   }
```

**第五步：把配色方案切到 Noctalia**

README 说扩展装完还要手动在 GUI 里选一次方案。也可以直接改配置（**必须在 LibreOffice 关闭时改**，否则会被运行中的进程覆盖）：

```bash
R=~/.config/libreoffice/4/user/registrymodifications.xcu
cp "$R" "$R.bak-$(date +%s)"      # 先备份

grep -o 'CurrentColorScheme[^/]*<value>[^<]*' "$R"
# 输出: CurrentColorScheme" oor:op="fuse"><value>COLOR_SCHEME_LIBREOFFICE_AUTOMATIC

sed -i 's|\(CurrentColorScheme" oor:op="fuse"><value>\)COLOR_SCHEME_LIBREOFFICE_AUTOMATIC|\1Noctalia|' "$R"
xmllint --noout "$R" && echo 配置仍合法
```

### 7.6 验证颜色转换正确性

`apply.sh` 要把 hex 转成十进制。拿渲染后的 staging 文件逐项核对转换结果：

```text
正确转换: 42/42
```

> **核对时容易搞错对照对象**：不要拿模板目录里的 `Theme_Colors.xcu` 去比。那是模板源文件，里面大部分颜色是 `{{ colors.xxx }}` 占位符，只有十几个是字面写死的 hex（文档背景、BASIC 编辑器配色那些），拿它比只能验证一小部分，还会误以为总共只有 15 个颜色。正确的对照对象是 `libreoffice-theme-staging/Theme_Colors.xcu`，即 Noctalia 渲染后的那份，里面 42 个颜色全是实际 hex 值。

有一个坑值得记录：用 `grep -cE "<value>[0-9a-fA-F]{6}</value>"` 检查"是否还有未转换的 hex"会得到 **1**，看起来像漏了一个。实际是**误报**：

```text
BASICKeyword 原值 0b57d0 → 转换后 743376
```

`743376` 是正确的十进制结果，但它恰好是 6 位纯数字，而纯数字也符合 hex 字符集，所以被正则当成未转换的 hex 匹配上了。检查这类问题不能只看字符集，要拿原值做换算比对。

### 7.7 最终验证

```bash
# 启动一次 LibreOffice，确认配置不被重置
libreoffice --headless --convert-to pdf t.fodt --outdir . >/dev/null
grep -o 'CurrentColorScheme[^/]*<value>[^<]*' ~/.config/libreoffice/4/user/registrymodifications.xcu
# 输出: ...<value>Noctalia   ← 保留住了

unopkg list | grep -c dev.noctalia
# 输出: 1
```

打开 LibreOffice，配色已跟随 Noctalia 当前调色板（背景 `#292214`、强调色 `#e5bb66`、菜单栏 `#453921`）。

---

## 8. 最终配置摘要

### 8.1 `~/.local/bin/`（补齐命令）

```text
libreoffice -> /opt/libreoffice26.2/program/soffice
unopkg      -> /opt/libreoffice26.2/program/unopkg
lowriter, localc, loimpress, lodraw, lobase, lomath   （wrapper 脚本，各带 --<模式> 参数）
```

### 8.2 `~/.config/fontconfig/conf.d/99-zh-win-aliases.conf`

Windows 中文字体名到本机字体的映射，内容见 6.3。

### 8.3 LibreOffice 语言模块

```text
libobasis26.2-zh-cn        26.2.4.2-2
libobasis26.2-zh-cn-help   26.2.4.2-2
libreoffice26.2-zh-cn      26.2.4.2-2
```

版本必须与 `libobasis26.2-core` 完全一致。

### 8.4 `~/.local/state/noctalia/community-templates/libreoffice/META-INF/manifest.xml`

手工补的文件，内容见 7.5。**注意这个文件会被模板更新覆盖掉**，见 9.3。

### 8.5 `~/.config/libreoffice/4/user/registrymodifications.xcu`

```ini
CurrentColorScheme = Noctalia
```

---

## 9. 避坑清单

### 9.1 装官网 deb 版就要接受命令名带版本号

要么记住 `libreoffice26.2` 这个名字，要么按 4.1 建软链接。如果不想折腾，可以卸掉官网版改用 `sudo apt install libreoffice`，仓库版自带全套标准命令名和标准包名，AI 和各种脚本也都能正确识别。

### 9.2 LibreOffice 升级后软链接会失效

升到 27.x 时安装目录会变成 `/opt/libreoffice27.x`，`~/.local/bin` 里所有指向 `/opt/libreoffice26.2` 的链接全部断掉。症状是命令报错找不到文件。重跑 4.1 和 7.5 第二步，把版本号换掉即可。

同理，语言包也要重新下载对应新版本的，旧版语言包不能跨版本用。

### 9.3 Noctalia 模板更新会覆盖手工补的 manifest

只要上游分发管道还没修，每次模板缓存刷新都可能把 `META-INF/` 目录清掉。症状是**改主题后 LibreOffice 配色不再跟着变**。

确诊和恢复：

```bash
ls ~/.local/state/noctalia/community-templates/libreoffice/META-INF/manifest.xml \
  || echo "manifest 又没了，重新补 7.5 第一步"

bash ~/.local/state/noctalia/community-templates/libreoffice/apply.sh
```

### 9.4 改主题前先关 LibreOffice

`apply.sh` 检测到 LibreOffice 在运行就跳过安装，而且**没有自动重试**——关掉 LibreOffice 也不会自动补装，必须重新触发一次主题应用或手动跑 `apply.sh`。这是模板自身的设计取舍（避免运行时安装破坏扩展），不是本机的问题。

习惯：先关 LibreOffice，再切主题。忘了就手动补跑一次。

### 9.5 "staging 文件已生成"不等于主题已生效

这是本次排查最初的误判点。看到 `libreoffice-theme-staging/Theme_Colors.xcu` 存在、时间戳还是新的，很容易以为 Noctalia 那边已经干完了。实际上那只是 hex 占位符的半成品，真正的转换、打包、安装全在 `post_hook` 里。判断是否真的生效应该看：

```bash
unopkg list | grep dev.noctalia    # 扩展是否注册
ls .../build/*.oxt                 # 包是否生成
```

### 9.6 `set -euo pipefail` 的脚本失败往往没有任何输出

`apply.sh` 在第 44 行中止时，`cp` 的报错被 post_hook 吞掉了，Noctalia 日志里也搜不到相关记录。表现就是"什么都没发生"。排查这类静默失败的方法是**看产物**：哪些中间文件生成了、哪些没有，据此定位脚本停在第几行。本例中"META-INF 是空目录 + 没有 .oxt"直接锁定了行号。

### 9.7 判断字体问题要看渲染结果，不要只看 `fc-match`

`fc-match` 显示映射成功，不代表应用真的按这个映射渲染。LibreOffice 有自己的替换逻辑会覆盖 fontconfig。可靠的验证方式是转 PDF 后 `pdffonts` 看实际嵌入的字体，再 `pdftoppm` 转图目视确认。

---

## 10. 复查命令

```bash
# 命令名是否齐全
for c in libreoffice soffice unopkg lowriter localc; do
  printf "%-12s " "$c"; command -v $c >/dev/null && echo OK || echo MISSING
done

# 中文语言包
dpkg -l | grep -E "libobasis.*zh-cn" | awk '{print $2, $3}'

# 字体与映射
fc-list :lang=zh-cn | wc -l
fc-match SimSun; fc-match 微软雅黑; fc-match 楷体

# Noctalia 主题链路
ls ~/.local/state/noctalia/community-templates/libreoffice/META-INF/manifest.xml
ls -lh ~/.local/state/noctalia/community-templates/libreoffice/build/*.oxt
unopkg list | grep -A2 dev.noctalia
grep -o 'CurrentColorScheme[^/]*<value>[^<]*' \
  ~/.config/libreoffice/4/user/registrymodifications.xcu
```

全部正常的话应当看到：五个命令都 OK、三个 zh-cn 包版本一致、中文字体 111 个、三个 `fc-match` 分别指向思源宋体/思源黑体/霞鹜文楷、manifest 存在、`.oxt` 存在、扩展 `is registered: yes`、配色方案是 `Noctalia`。

---

## 11. 归纳

四个问题，两条根因。

**根因一：官网 deb 包的布局与发行版仓库版完全不同**（命令名带版本号、包名是 `libobasis*`、装在 `/opt`、`unopkg` 不在 PATH）。它直接造成了终端命令缺失、AI 误判未安装，并且是 Noctalia 主题失效的第二个缺陷的成因。凡是基于"标准布局"假设的检测都会失败，而这类假设几乎存在于所有教程、脚本和第三方集成里。

**根因二：Noctalia 的模板分发管道不递归子目录**，导致 LibreOffice 模板必需的 `META-INF/manifest.xml` 从未下发。上游仓库文件是完整的，CI 校验也是递归的，只有 catalog 生成/分发这一环没有递归，所以问题长期没被发现。LibreOffice 是仓库里唯一依赖子目录的模板，也就是唯一受害者。

排查过程中有两个方法论上的收获：

- **静默失败要靠产物定位**。`set -e` 脚本中止时可能没有任何日志，但中间文件的生成情况能精确指出停在第几行。
- **"上游有这个文件"和"我这里有这个文件"是两件事**。最初我判断是上游漏了文件，克隆仓库后才发现文件在、是分发丢的。这个区别决定了 issue 该怎么写、该提给谁。

中文化部分没有意外，标准流程：装字体、配 fontconfig 映射、装版本号精确匹配的语言包。唯一需要注意的是 LibreOffice 的字体替换优先级高于 fontconfig，所以 Windows 字体的字形区分不会自动生效，但中文显示本身没问题。

---

*记录时间：2026-08-31 · 环境：Kali Rolling 2026.3 / niri 26.04 / Noctalia v5.0.0 / LibreOffice 26.2.4.2（官网 deb）*

*一句话结论：**官网 deb 版的 LibreOffice 命令叫 `libreoffice26.2`、`unopkg` 不在 PATH，这两点会连带坑掉终端使用、AI 识别和第三方主题集成；Noctalia 的 LibreOffice 模板则因为分发管道不递归子目录而缺 `META-INF/manifest.xml`，所有用户都受影响。***
