# Neovim 从一行报错到全套开发环境 —— Kali + niri + Noctalia5 下的配置重建与踩坑档案

> **场景**：Kali Rolling + niri（Wayland）+ Noctalia v5 桌面壳。nvim 每次启动都弹 `E5113: module 'base16-colorscheme' not found`，按回车才能进去。原本只想修掉这个报错，最后连带把整套开发环境（LSP、文件树、模糊搜索、格式化、LaTeX、Markdown）都搭起来了。
> **结论先行**：最初那个报错的根因很单纯 —— **nvim 里没有任何插件管理器，`base16-nvim` 插件从来没装过**，而 Noctalia 侧的动态配色链路一直是通的。真正麻烦的是修的过程中新引入和新暴露出的 **13 个坑**，包括我自己用 lazy.nvim 带进来的一个 regression。
> **状态**：全部修复并逐项实测通过。25 个插件、12 个 mason 包，启动耗时 28ms。

写于**2026年8月31日-星期一-北京时间05：09：15**

---

## 1. 环境信息（复现此问题的系统配置）

| 项目 | 值 | 说明 |
| --- | --- | --- |
| 发行版 | Kali GNU/Linux Rolling `2026.3` | apt 系，`ID_LIKE=debian` |
| 会话类型 | `wayland`（`XDG_SESSION_TYPE=wayland`） | |
| 桌面环境 | **niri**（`XDG_CURRENT_DESKTOP=niri`） | wlroots 系平铺合成器 |
| 桌面壳 | **Noctalia v5.0.0~beta.8**（`/usr/bin/noctalia`，apt 装的） | 自带动态配色引擎 |
| Neovim | **v0.12.3**（LuaJIT 2.1.1761786044） | 关键前提，很多结论跟这个版本号直接相关 |
| 终端字体 | JetBrainsMono Nerd Font（kitty / alacritty 都在用） | 图标能正常显示的前提 |
| 编译工具 | gcc 15.3.0、clang 21.1.8、cmake 4.3.4、make | fzf-native 和 clangd 要用 |
| 语言运行时 | node v24.20.0、python 3.14.6、rustup 1.29.0 / cargo 1.95.0、openjdk 21、dotnet **6.0.400** | dotnet 版本后面成了一个决策点 |
| TeX | latexmk、pdflatex、xelatex、lualatex、biber、chktex 全齐；ctex 宏包已装，48 个中文字体 | 中文排版靠 xelatex + ctex |
| PDF 阅读器 | zathura 2026.07.18 + `libpdf-poppler.so` 后端 | 支持 SyncTeX；过程中补装的 |
| sudo | **需要密码**（无免密） | 所以全程只往用户目录装东西 |

改动前的 nvim 配置只有 4 个文件：`init.lua` + `lua/config/{options,keymaps,autocmds}.lua`，外加一个 `lua/matugen.lua`。**没有插件管理器，`~/.local/share/nvim/` 是空目录。**

---

## 2. 起始问题现象

打开 nvim 立刻报错，必须按回车才能继续：

```text
Error in /home/bassttelsevic/.config/nvim/init.lua:
E5113: Lua chunk: /home/bassttelsevic/.config/nvim/lua/matugen.lua:4: module 'base16-colorscheme' not found:
        no field package.preload['base16-colorscheme']
        no file './base16-colorscheme.lua'
        no file '/usr/share/luajit-2.1/base16-colorscheme.lua'
        ... （后略十余行搜索路径）
stack traceback:
        [C]: in function 'require'
        /home/bassttelsevic/.config/nvim/lua/matugen.lua:4: in function 'setup'
        /home/bassttelsevic/.config/nvim/init.lua:15: in main chunk
Press ENTER or type command to continue
```

---

## 3. 根因分析

### 3.1 缺的是插件，不是配置

`lua/matugen.lua:4` 要 `require('base16-colorscheme')`，这个模块来自插件 **RRethy/base16-nvim**，不是 Neovim 内置模块。

```bash
ls -la ~/.local/share/nvim/
# 输出: 总计 8，只有 . 和 ..     ← 空目录，一个插件都没有

find ~/.local/share/nvim ~/.config/nvim -iname "*base16*"
# 输出: 空                       ← 全盘没有 base16-nvim
```

`lua/config/plugins.lua` 里只有一行注释「插件管理（暂不使用，占位保留）」，而且 `init.lua` **压根没 require 它**。

### 3.2 pcall 保护错了位置

`init.lua` 原本是这样：

```lua
local ok, matugen = pcall(require, 'matugen')
if ok then matugen.setup() end
```

`pcall` 只保护了 **require（加载 matugen.lua 这个文件）** —— 这一步是成功的，因为文件确实存在，所以 `ok = true`。真正抛错的是紧跟其后**完全没有被保护的** `matugen.setup()`，错误于是原样冒泡到顶层。

这是个很典型的错误：**保护了「拿到模块」，没保护「调用模块」**。

---

## 4. 我的三次误判（必须交代）

这次排查我下过三个错误结论，都是靠后续实测推翻的。记下来，因为每一个都可能把人带偏。

### 4.1 误判一：以为 nvim 没接到 Noctalia 上

我一开始看到 `~/.config/noctalia/config.toml` 是 0 字节、又没有 `user-templates.toml`，就断言「Noctalia 没有在给 nvim 生成配色，你得手动接上」。

**错了。** 证据是文件时间和内容：

```bash
stat -c "%y  %n" ~/.config/kitty/themes/noctalia.conf ~/.config/nvim/lua/matugen.lua
# 2026-08-31 03:24:42  ~/.config/kitty/themes/noctalia.conf
# 2026-08-31 03:24:44  ~/.config/nvim/lua/matugen.lua   ← 比 kitty 还晚 2 秒

ls ~/.local/state/noctalia/community-templates/ | grep -i neovim
# 输出: neovim                   ← Noctalia 自带的 neovim 模板一直存在且已启用
```

Noctalia 的模板配置就在那里：

```toml
[templates.nvim-base16]
input_path  = "matugen-template.lua"
output_path = "$XDG_CONFIG_HOME/nvim/lua/matugen.lua"
post_hook   = "bash '{{ config_dir }}/apply.sh'"
```

也就是说 **Noctalia 侧从头到尾都是通的**，`lua/matugen.lua` 一直由它生成和更新。真正缺的只有 nvim 这边的 base16-nvim 插件。

**教训**：判断「A 有没有接上 B」，不要只看 A 的配置文件，要看 B 的产物有没有被更新。文件 mtime 是最直接的证据。

### 4.2 误判二：以为 signal handler 会指数级泄漏

Noctalia 生成的 `lua/matugen.lua` 末尾有这么一段热重载逻辑：

```lua
local signal = vim.uv.new_signal()
signal:start('sigusr1', vim.schedule_wrap(function()
  package.loaded['matugen'] = nil
  require('matugen').setup()
end))
```

我的推理是：handler 触发时清掉模块缓存并重新 `require`，而模块顶层又会**无条件**注册一个新的 signal handle；下次信号来时两个 handler 都触发、各自再注册一个 —— 于是 1→2→4→8，换十次壁纸就是 1024 个 handler。听起来完全合理。

**实测直接推翻了。** 起一个真实 nvim 实例，连发 5 次 SIGUSR1，用 `vim.uv.walk` 数 signal 类型的 handle：

```text
初始 signal handle 数: 1
第 1 次 SIGUSR1 后: 1
第 2 次 SIGUSR1 后: 1
第 3 次 SIGUSR1 后: 1
第 4 次 SIGUSR1 后: 1
第 5 次 SIGUSR1 后: 1
```

原因：旧模块表被 `package.loaded['matugen'] = nil` 丢弃后变成不可达对象，**Lua GC 回收它时会自动 close 那个 uv handle**。所以根本不会累积。

我原本已经准备好一个「monkeypatch `vim.uv.new_signal` 在 require 期间屏蔽注册」的修法，测完直接作废，**一行都没改**。

**教训**：涉及 GC、异步、资源生命周期的推理，链条再顺也要跑数据验证。这次要是没测，就会为了一个不存在的 bug 引入一段很脏的 hack。

### 4.3 误判三：装了一套多余的 matugen，还去抢同一个输出文件

因为误判一，我按官方文档独立装了一套 matugen：`~/.local/bin/matugen` 二进制 + `~/.config/matugen/config.toml` + `~/.config/nvim/lua/matugen-template.lua`，输出目标写的是 `~/.config/nvim/lua/matugen.lua`。

**这正好和 Noctalia 的 neovim 模板抢同一个文件。** 表现就是我生成的紫色调（`base00 = '#14121c'`）在 03:24:44 被 Noctalia 改回了黄绿色调（`base00 = '#23231a'`），缩进风格也从我的 2 空格 + 中文注释变成了它的 4 空格 + 硬编码 hex。

确认 Noctalia 不依赖 matugen 二进制之后全部清理：

```bash
dpkg -s noctalia | grep -i "^Depends" | tr ',' '\n' | grep -ci matugen
# 输出: 0                              ← 依赖里没有 matugen
strings /usr/bin/noctalia | grep -iw matugen
# 输出: 空                              ← 二进制里无引用
```

而且开工前 `which matugen` 就是空的，Noctalia 却已经在正常给 kitty/niri/gtk/qt 生成主题 —— 这本身就是它不需要 matugen CLI 的铁证。

```bash
rm ~/.config/nvim/lua/matugen-template.lua
rm -rf ~/.config/matugen
rm ~/.local/bin/matugen
```

---

## 5. 我自己引入的 regression：lazy.nvim 重置 runtimepath

这个坑值得单独一节，因为它是**修 bug 的过程中新造出来的**，而且症状和原始问题长得很像（同样是 E5113）。

### 5.1 现象

装完 lazy.nvim 之后，**打开任何 `.lua` 文件都会报错**：

```text
Error in BufReadPost Autocommands for "*":
... script /usr/share/nvim/runtime/ftplugin/lua.lua: Vim(runtime):E5113: Lua chunk:
/usr/share/nvim/runtime/lua/vim/treesitter.lua:460: Parser could not be created for buffer 1 and language "lua"
```

### 5.2 对照实验定位

```bash
nvim --clean --headless /tmp/t.lua -c 'qa!'
# 输出: 空                    ← 不加载我的配置：无报错

nvim --headless /tmp/t.lua -c 'qa!'
# 输出: 上面那段 E5113        ← 加载我的配置：报错
```

于是问题一定在配置里。查 runtimepath：

```bash
nvim --clean --headless -c 'lua print(vim.o.rtp:find("/usr/lib/nvim") and "有" or "没有")' -c 'qa!'
# 输出: 有

nvim --headless -c 'lua print(vim.o.rtp:find("/usr/lib/nvim") and "有" or "没有")' -c 'qa!'
# 输出: 没有                  ← 根因锁定

nvim --headless -c 'lua print(vim.inspect(vim.api.nvim_get_runtime_file("parser/lua.so", true)))' -c 'qa!'
# 输出: {}                    ← 找不到任何 parser
nvim --clean --headless -c 'lua print(vim.inspect(vim.api.nvim_get_runtime_file("parser/lua.so", true)))' -c 'qa!'
# 输出: { "/usr/lib/nvim/parser/lua.so" }
```

### 5.3 根因

lazy.nvim 的默认配置里有这一项（`lua/lazy/core/config.lua`）：

```lua
rtp = {
  reset = true, -- reset the runtime path to $VIMRUNTIME and your config directory
  paths = {},
  ...
}
```

**Debian/Kali 的 nvim 包把 tree-sitter parser 装在 `/usr/lib/nvim/parser/`**，而这个路径既不是 `$VIMRUNTIME` 也不是配置目录，于是被 lazy 的 rtp 重置直接丢掉。Neovim 0.12 的 `runtime/ftplugin/lua.lua` 第 2 行会调 `vim.treesitter.start()`，找不到 parser 就抛错。

这是个发行版打包路径 + 插件管理器默认行为的组合坑，在 Arch 之类把 parser 放在 `$VIMRUNTIME` 下的发行版上不会出现。

### 5.4 修法（不写死路径）

在 lazy 重置 rtp **之前**，先扫出当前 rtp 里所有带 `parser/` 的目录，再交给 lazy 保留：

```lua
-- lua/config/lazy.lua
local parser_rtp = {}
for _, dir in ipairs(vim.api.nvim_list_runtime_paths()) do
    if (vim.uv or vim.loop).fs_stat(dir .. "/parser") then
        table.insert(parser_rtp, dir)
    end
end

require("lazy").setup({
    -- ...
    performance = { rtp = { paths = parser_rtp } },
})
```

不写死 `/usr/lib/nvim`，换发行版或上游改路径都不会失效。

### 5.5 验证

```bash
nvim --headless -c 'lua print(vim.o.rtp:find("/usr/lib/nvim") and "有" or "没有")' -c 'qa!'
# 输出: 有
nvim --headless /tmp/t.lua -c 'qa!'
# 输出: 空                    ← 报错消失
# treesitter 高亮激活: 是   ← 顺带白拿到内置 treesitter 高亮
```

---

## 6. 实测中发现的其余坑（逐个记录）

下面每一条都是「按官方文档或常见教程写完，实测发现不工作」的情况。

### 6.1 clangd 22 的 `--function-arg-placeholders` 必须带值

**现象**：C/C++ 文件打开后 LSP 立刻退出，`vim.lsp.get_clients()` 返回空。

```text
Client clangd quit with exit code 1 and signal 0.
```

查 `~/.local/state/nvim/lsp.log`：

```text
"rpc" "clangd" "stderr" "clangd: for the --function-arg-placeholders option: requires a value!"
```

**根因**：网上流传的 clangd 配置片段里是裸写这个 flag，但 **clangd 22.1.6 要求它必须带值**（老版本可以裸写）。

**修法**：

```lua
-- 错误
"--function-arg-placeholders",
-- 正确
"--function-arg-placeholders=1",
```

**验证**：`t.c` 报出 2 条诊断（`Incompatible pointer to integer conversion`），`t.cpp` 报出 `No matching member function for call to 'push_back'`。

### 6.2 nvim-lint 内置的 zsh linter 在 nvim 里根本跑不起来

**现象**：zsh 文件明显有语法错误，`lint.try_lint()` 调用成功、job 也结束了，但诊断数是 0。而命令行里 `zsh -n` 明明能报错。

**定位过程**：先确认不是解析的问题 ——

```bash
# errorformat 能否解析 zsh 的输出
nvim --clean --headless -c 'lua
local items = vim.fn.getqflist({ lines = { "/dev/stdin:2: parse error near `)'"'"'" }, efm = "%s:%l:%m" }).items
print(items[1].valid, items[1].bufnr, items[1].lnum)' -c 'qa!'
# 输出: 1  0  2               ← valid=1 bufnr=0，能通过 nvim-lint 的过滤条件
```

于是把 parser 包一层，直接打印原始输出：

```text
cmd=zsh  stdin=true  args={ "--no-exec", "--no-rcs", "--no-globalrcs", "/dev/stdin" }
RAW_OUTPUT_BEGIN[zsh: can't open input file: /dev/stdin
]RAW_OUTPUT_END
PARSED_COUNT=0
```

**根因**：nvim-lint 内置的 zsh linter 把 buffer 内容写进子进程 stdin，再让 zsh 去读 `/dev/stdin`。但 nvim 子进程的 stdin 是 **libuv 创建的管道**，zsh 无法通过 `/dev/stdin`（即 `/proc/self/fd/0`）重新打开它。同样是内置的 `bash` linter 用的是 `-n -s`（从 stdin 直接读脚本），所以不受影响。

**修法**：改成检查磁盘上的文件。

```lua
lint.linters.zsh = vim.tbl_extend("force", lint.linters.zsh, {
    stdin = false,
    append_fname = true,   -- 命令变成 zsh --no-exec --no-rcs --no-globalrcs <文件>
    args = { "--no-exec", "--no-rcs", "--no-globalrcs" },
})
```

顺带一个连带调整：既然检查的是磁盘文件，触发时机里的 `InsertLeave` 就没意义了（那时改动还没落盘，查的仍是旧内容），所以只保留 `BufReadPost` 和 `BufWritePost`。

**验证**：错误文件报出「第 2 行 `parse error near ')'`」，语法正确的文件 0 诊断。

### 6.3 shellcheck 官方不支持 zsh（很多博客写错了）

这个不是 bug，是事实核查。搜到的几篇中文和英文博客都写「shellcheck 支持 sh/bash/zsh」，但 shellcheck 官方 wiki 的 **SC1071** 条目写得很清楚：

> ShellCheck only supports a limited number of Bourne-based Unix shells: bash, ksh, dash and POSIX sh. **It does not support scripts written for other shells like Zsh**, Csh, Tcsh or PowerShell.

对应的 issue（koalaman/shellcheck#809）挂了多年没有进展。

**影响到配置决策**：`bashls` 会自动调用 PATH 里的 shellcheck。如果把 `zsh` 加进 bashls 的 `filetypes`，结果就是**每个 zsh 文件都被报「不支持的 shell」**。所以刻意保持它默认的 `{ "bash", "sh" }`，zsh 单独交给 nvim-lint。

分工最终是：

| filetype | 谁负责 | 能查什么 |
| --- | --- | --- |
| `bash` / `sh` | bashls（自动调 shellcheck） | 语法 + 语义（变量没引号、无用的 cat 等） |
| `zsh` | nvim-lint 跑 `zsh --no-exec` | **只有语法**（括号不配对、缺 `fi`/`done`） |

### 6.4 vimtex 最新版要求 nvim 0.12.4，本机 0.12.3（还连带把 texlab 一起搞挂）

**现象**：打开 `.tex` 文件报错，而且 `VimtexCompile` 命令不存在、`\ll` 映射不存在、**texlab 也没挂上**。

```text
Error in BufReadPost Autocommands for "*":
... script ~/.local/share/nvim/lazy/vimtex/ftplugin/tex.vim, line 19:
Vim(echoerr):Error: VimTeX does not support your version of Vim
```

看它的检测代码（`ftplugin/tex.vim`）：

```vim
if !(!get(g:, 'vimtex_version_check', 1)
      \ || has('nvim-0.12.4')
      \ || has('patch-9.2.0'))
  echoerr 'Error: VimTeX does not support your version of Vim'
```

**这个报错的连带伤害比看起来大**：`echoerr` 在 ftplugin 里抛出，中断了整条 FileType/ftplugin 链，所以 mason-lspconfig 那边的 texlab 也跟着挂不上。表面看是「LaTeX 的 LSP 坏了」，实际是 vimtex 的版本检测。

**查证是不是真需要 0.12.4**：

```bash
cd ~/.local/share/nvim/lazy/vimtex
git log -5 --oneline -L 15,21:ftplugin/tex.vim
# 1d27d952 chore: bump version requirements    ← 2026-07-22
#   -  \|| has('nvim-0.10')
#   +  \|| has('nvim-0.12.4')
```

是一条例行的 `chore` 提交（作者的策略是只支持最新稳定版），不是因为用了 0.12.4 独有的 API。

**两种修法及取舍**：

| 方案 | 问题 |
| --- | --- |
| 设 `g:vimtex_version_check = 0` 跳过检测，继续用 master | 关掉了安全检查，master 后续提交可能真的用上 0.12.4 的 API |
| **锁到抬高要求之前的 tag** | 没有未知风险，可复现 |

查各 tag 的要求：

```bash
for t in $(git tag --sort=-creatordate | head -6); do
  echo "$t → nvim-$(git show $t:ftplugin/tex.vim | grep -oP "has\('nvim-\K[0-9.]+" | head -1)"
done
# v2.18 → 需要 nvim-0.10  (2026-07-21)     ← 抬高要求前一天发布，正合适
# v2.17 → 需要 nvim-0.10  (2025-09-22)
# v2.16 → 需要 nvim-0.9.5 (2025-01-18)
```

**修法**：lazy 的 spec 里加 `tag = "v2.18"`。

一个操作细节：改完 spec 后跑 `:Lazy! restore` **没用**，它是按 lockfile 恢复，不会应用新的 tag 约束（版本仍是 `v2.18-38-gcb81778a`）。要用 `:Lazy! update vimtex` 才会真正 checkout 到 tag。

**验证**：

```text
VimtexCompile 命令=存在   VimtexView 命令=存在
\ll 映射=<Plug>(vimtex-compile)   \lv 映射=存在   dse 文本对象=存在
LSP=harper_ls+texlab               ← texlab 恢复挂载
```

以后 nvim 升到 0.12.4+，删掉那行 `tag` 就能回 master。

### 6.5 nvim-colorizer 抢在 termguicolors 异步探测之前加载

**现象**：`Colorizer: Error: &termguicolors must be set`。

这个坑的隐蔽之处在于：**headless 下测会误判成「只是 headless 的正常现象」**。用伪终端测才暴露：

```bash
TERM=xterm-256color COLORTERM=truecolor script -qec "nvim a.lua -c '...'" /dev/null
# termguicolors=true              ← 最终值是 true
# 有Colorizer错误: true            ← 但报错确实发生了
```

**根因**：Neovim 0.10+ 会自动探测终端真彩色能力并打开 `termguicolors`，但那是**异步**的（要等终端响应 DSR 查询）。colorizer 在 `BufReadPre` 就加载，跑在探测完成之前，于是读到 `termguicolors=false` 直接报错退出 —— 结果是**颜色块功能实际不工作**，尽管之后 `termguicolors` 变成了 true。

我之前还专门测过一次「真终端里 `termguicolors` 会自动变 true，所以不用手动设」，那个结论本身没错，但**漏掉了时序**。

**修法**：`lua/config/options.lua` 里显式设置，消掉竞争。

```lua
vim.opt.termguicolors = true
```

**验证**：`有Colorizer错误: false`。

### 6.6 mason-lspconfig 把 stylua 当成语言服务器启用了

**现象**：打开 `.lua` 文件，`vim.lsp.get_clients()` 返回的客户端名是 **`stylua`**。stylua 是格式化器，不该作为 LSP 挂载。

**根因**：两件事凑一起 ——

1. **nvim-lspconfig 里确实有 `lsp/stylua.lua`**，因为 stylua 带 `--lsp` 模式：

   ```lua
   return { cmd = { 'stylua', '--lsp' }, filetypes = { 'lua' }, ... }
   ```

2. mason-lspconfig 的 `automatic_enable = true` 会把「所有已装的 mason 包」里能对上 lspconfig 条目的**全部** `vim.lsp.enable()`。我为 conform 装了 stylua，于是它被当成 LSP 启用了。

**为什么必须处理**：不只是多一个进程。走 LSP 那条路的 stylua **不带 conform 里配的 `--indent-width 4`**，会用它自己的默认值 —— 正好是我之前提醒过的「格式化器悄悄改缩进」那类问题。

**顺便查了所有装的格式化器谁会被误启用**：

| mason 包 | lspconfig 里有条目吗 | 处理 |
| --- | --- | --- |
| stylua | 有 `lsp/stylua.lua` | **排除** |
| ruff | 有 `lsp/ruff.lua` | **保留**（见下） |
| shfmt | 无 | 不受影响 |
| clang-format | 无 | 不受影响 |
| shellcheck | 无 | 不受影响 |

**修法**：

```lua
automatic_enable = {
    exclude = { "stylua" },
},
```

**ruff 为什么不排除**：它作为 LSP 提供的是 basedpyright 没有的**快速 lint 规则**（未使用的 import、未定义变量等），basedpyright 干的是类型检查，两者互补不重复。而格式化仍然由 conform 走 CLI（conform 的 python 项配了 `ruff_organize_imports` + `ruff_format`），`lsp_format = "fallback"` 在有 CLI 格式化器时不会触发，所以不冲突。

**验证**：

```text
a.lua   → 无                      ← stylua 不再挂载
a.py    → basedpyright,ruff       ← ruff 保留
conform 对 lua 格式化 → 缩进=4 空格 ← 仍然按我配的来
```

### 6.7 clang-format 的 `--fallback-style` 不接受内联 YAML

**背景**：clang-format 默认是 LLVM 风格（2 空格），和这台机器全局的 `shiftwidth=4` 不一致。我想让它在**项目没有 `.clang-format` 时**用 4 空格，于是加了：

```lua
prepend_args = { "--fallback-style={BasedOnStyle: LLVM, IndentWidth: 4}" },
```

**现象**：格式化静默失效（文件完全没变）。用 conform 的回调拿到错误：

```text
ERR=Formatter 'clang_format' error: Invalid fallback style: {BasedOnStyle: LLVM, IndentWidth: 4}
```

直接测也一样：

```bash
printf 'int main(){int x=1;return x;}\n' | clang-format --fallback-style="{BasedOnStyle: LLVM, IndentWidth: 4}" --assume-filename=x.c
# Invalid fallback style: {BasedOnStyle: LLVM, IndentWidth: 4}    exit=1

printf 'int main(){int x=1;return x;}\n' | clang-format -style="{BasedOnStyle: LLVM, IndentWidth: 4}" --assume-filename=x.c
# int main() {
#     int x = 1;      ← -style 能接受内联 YAML
# }
```

**根因**：`--fallback-style` **只接受预设名字**（LLVM、Google、Chromium…），内联 YAML 只有 `-style=` 支持。

**但不能改用 `-style=`**：那会**无条件覆盖项目自带的 `.clang-format`**，克别人仓库时会按自己的风格重排代码，弊远大于利。

**最终决定：不动它。** 让 clang-format 走 C/C++ 生态的标准规矩 —— 向上搜索项目里的 `.clang-format` 并遵守，搜不到才用 LLVM。想让自己的项目用 4 空格，就在项目根目录（或 `~`，clang-format 会一直往上找）放一个文件：

```bash
printf 'BasedOnStyle: LLVM\nIndentWidth: 4\n' > .clang-format
```

**验证**：无 `.clang-format` 时 2 空格；放上之后同一份代码格式化成 4 空格。

### 6.8 `<leader>ll` 和 vimtex 的 `<localleader>ll` 撞车

**根因**：原 `init.lua` 里两个 leader 是同一个键：

```lua
vim.g.mapleader = " "
vim.g.maplocalleader = " "       -- 和 mapleader 相同
```

vimtex 大量使用 `<localleader>`（`\ll` 编译、`\lv` 看 PDF、`\lc` 清理…），而我给 nvim-lint 的手动检查绑的是 `<leader>ll`。两个 leader 都是空格的话，这两个会变成同一个按键 `<空格>ll`。

buffer 局部映射优先级高于全局，所以在 `.tex` 文件里 vimtex 赢、别处 nvim-lint 赢 —— 能用，但行为随文件类型漂移，属于埋雷。

**修法**：把 localleader 错开成反斜杠（lazy.nvim 官方文档的示例也是这么写的）：

```lua
vim.g.maplocalleader = "\\"
```

### 6.9 flash.nvim 的 `s` 和 mini.surround 的 `s` 前缀撞车

flash.nvim 默认用 `s`/`S` 做跳转，mini.surround 默认也用 `s` 作为操作前缀（`sa`/`sd`/`sr`）。两个都装就必然冲突。

**修法**：mini.surround 整体挪到 `gs` 前缀（`gsa`/`gsd`/`gsr`/`gsf`/`gsh`），`s`/`S` 留给 flash。`gs` 原本是 vim 的 sleep 命令，基本没人用，这也是社区常见做法。

顺带一提，`s` 原本是「替换单个字符」，被 flash 占掉后可以用 `cl` 替代。

### 6.10 matugen 模板的注释里不能出现双花括号

（这条属于误判三期间的产物，虽然那套配置最后删掉了，但坑本身值得记。）

我在模板文件的注释里写了一句说明：

```lua
-- 占位符语法参考 matugen v4：{{colors.<role>.default.hex}}
```

结果渲染直接失败：

```text
Error: found '<' expected identifier, or '_'
 3 │ -- 占位符语法参考 matugen v4：{{colors.<role>.default.hex}}
   │                                        ┬
   │                                        ╰── found '<' expected identifier, or '_'
```

**根因**：模板引擎不理解「注释」这个概念，它只认双花括号。文件里**任何位置**出现的 `{{ }}` 都会被当表达式解析，包括注释里的示意写法。

### 6.11 lazy 的 keys 与 config 之别：写错地方按键就不存在

**现象**：验收时发现 `<leader>ll` 没注册。

**根因**：我一开始把这个 keymap 写在 nvim-lint 的 `config = function()` 里。而 nvim-lint 是 `event = { "BufReadPre", "BufNewFile" }` 懒加载的 —— **插件没加载，config 就没执行，keymap 也就不存在**。

**修法**：挪到 `keys = { ... }`。lazy 会为 `keys` 里声明的按键先注册一个**占位映射**，按下时才触发插件加载。

这也解释了为什么验收时 `Gitsigns`、`RenderMarkdown`、`WhichKey` 三个命令在 headless 下显示为「不存在」—— 它们分别等 `BufReadPre`、`ft=markdown`、`VeryLazy`，headless 没打开文件、没触发事件，属于懒加载的正常表现，不是错误。区分「懒加载没触发」和「真的坏了」，得开对应类型的真实文件再测。

### 6.12 vimtex 默认走 pdflatex，编译不了中文

**现象**：装好 zathura 之后测编译，vimtex 报 `Compilation failed`，但 PDF **居然生成了**（有效的 PDF 1.7）。

看日志才知道是 LaTeX 自己的错误：

```text
./doc.tex:4: LaTeX Error: Unicode character 测 (U+6D4B)
./doc.tex:5: LaTeX Error: Unicode character 这 (U+8FD9)
...
```

**根因**：latexmk 默认调 **pdflatex**，而 pdflatex 处理不了 UTF-8 中文。它对每个汉字报错但继续往下走，所以最后仍吐出一个缺字的 PDF —— 「失败」和「有产物」同时成立，容易误判成 vimtex 的问题。

**本机的前提是齐的**（查过才动手）：

```bash
kpsewhich ctex.sty
# /usr/share/texlive/texmf-dist/tex/latex/ctex/ctex.sty   ← ctex 已装
fc-list :lang=zh | wc -l
# 48                                                       ← 48 个中文字体
xelatex -interaction=nonstopmode cjk.tex; echo $?
# 0                                                        ← xelatex 编译中文没问题
```

**vimtex 怎么选引擎**（读源码得到的优先级，从高到低）：

1. 文件开头 20 行内的魔法注释 `% !TEX program = xelatex`（`autoload/vimtex/state/class.vim` 的 `get_tex_program()`）
2. 项目里 `.latexmkrc` 的 `$pdf_mode`
3. `g:vimtex_compiler_latexmk_engines` 里 `'_'` 这个默认键 —— 默认值是 `-pdf`，也就是 pdflatex

**修法**：把默认引擎改成 XeLaTeX。

```lua
vim.g.vimtex_compiler_latexmk_engines = {
    ["_"] = "-xelatex",      -- 默认
    pdflatex = "-pdf",
    lualatex = "-lualatex",
    luatex = "-lualatex",
    xelatex = "-xelatex",
}
```

注意这个字典是**整体替换**默认值的，所以要把还想保留的键一并写上。

单个文档想用回 pdflatex，在文件头写一行就行，优先级高于这个默认值：

```latex
% !TEX program = pdflatex
```

**验证**：

```text
VimTeX: Compilation completed
doc.log 首行: This is XeTeX, Version 3.141592653-2.6-0.999998 (TeX Live 2026/Debian)
pdftotext doc.pdf -  → 中文测试        ← 中文正确排进 PDF
doc.synctex.gz 948 字节                 ← SyncTeX 也在
```

### 6.13 Wayland 下 vimtex 的 zathura viewer 一直报找不到窗口 ID

**现象**：每次 `\lv` 都提示 `VimTeX: Viewer cannot find Zathura window ID!`。

**根因**：vimtex 的 `zathura` viewer 用 `_template.vim` 里的 `xdo_exists()` / `xdo_get_id()` 判断窗口是否存在、以及聚焦窗口，而这两个函数依赖 **xdotool** —— X11 工具。zathura 是 GTK3 应用，同时链接了 libX11 和 libwayland-client：

```bash
ldd /usr/bin/zathura | grep -iE "wayland|X11"
# libX11.so.6 ...
# libwayland-client.so.0 ...
```

在 niri（Wayland）会话下它跑成**原生 Wayland 客户端**，xdotool 看不到，所以窗口 ID 永远拿不到。

**先确认这个警告到底有没有实际影响**。`_template.vim` 里的逻辑是：

```vim
if self._exists()
  call self._forward_search(l:outfile)
else
  call self._start(l:outfile)
endif
```

`_exists()` 恒为假，意味着每次都走 `_start`（重新启动 viewer）。按这个逻辑推理，连按两次 `\lv` 应该开出两个窗口。**实测不是**：

```text
连续两次 VimtexView → zathura 进程数: 1
```

原因是 **zathura 自带 D-Bus 单实例**：同一个文件的第二次调用会转发给已有实例，自己退出。所以正向搜索仍然正确落到已开的窗口上。这个警告只影响「把窗口提到前台」这一件事，功能没坏。

**修法**：换成 `zathura_simple`。它不做任何窗口管理（不碰 xdotool），噪音消失，功能靠 zathura 自己的 D-Bus 兜住。

```lua
vim.g.vimtex_view_method = "zathura_simple"
```

**验证**：

```text
无警告输出
zathura 进程数: 1
进程命令行: zathura -x /usr/bin/nvim --headless -c "VimtexInverseSearch %{line}:%{column} '%{input}'" \
            --synctex-forward 1:1:/tmp/textest/doc.tex
```

正向搜索参数（`--synctex-forward`）和反向搜索的编辑器回调（`-x`）都在，两个方向都是通的。

---

## 7. 一个测试方法上的坑：pkill -f 把自己杀了

测 SIGUSR1 热重载时我这么写：

```bash
timeout 60 nvim --headless --listen /tmp/s.sock &
sleep 3
for i in 1 2 3 4; do
  pkill -SIGUSR1 -f "nvim --headless --listen /tmp/s.sock"   # ← 问题在这
  ...
done
```

脚本只打印了第一行就整个中断。原因是 **`pkill -f` 匹配的是完整命令行，而运行这段脚本的 bash 进程的命令行里正好包含那串模式字符串**，于是 SIGUSR1 打到了 bash 自己身上 —— bash 对 SIGUSR1 的默认动作是终止。

改成先取 PID 再精确发信号：

```bash
PID=$(pgrep -f "listen /tmp/s.sock" | head -1)
kill -USR1 $PID
```

这和之前 pi 那篇里「排查进程别在超大正则里混宽泛字符」是同一类教训的变体：**`pkill -f` 在脚本里用要格外小心自我匹配。**

**而且同一个坑我踩了两次。** 后面测 zathura 时想清场，又写了：

```bash
pkill -f "zathura.*textest"   # 同样的错，又把自己的 shell 干掉了
```

这次的表现是整个命令没有任何输出（连后面的 `echo` 都没执行）。正确写法始终是先取 PID：

```bash
pids=$(pgrep -x zathura)
[ -n "$pids" ] && kill $pids
```

用 `pgrep -x`（精确匹配进程名）而不是 `-f`（匹配完整命令行）能从根上避开这个问题。

---

## 8. 分阶段实施记录

### 8.1 第一阶段：修原始报错（插件管理器 + 配色）

按 lazy.nvim 官方的 Structured Setup 做：

| 文件 | 作用 |
| --- | --- |
| `init.lua` | 末尾改成 `require("config.lazy")`，删掉裸奔的 `matugen.setup()` |
| `lua/config/lazy.lua` | lazy 引导 + `{ import = "plugins" }` |
| `lua/plugins/colorscheme.lua` | base16-nvim 声明（`lazy = false`, `priority = 1000`） |
| `lua/config/theme.lua` | 配色名 / 兜底名的单一事实来源 |
| `colors/matugen.lua` | 让 `:colorscheme matugen` 原生可用 |
| ~~`lua/config/plugins.lua`~~ | 删除（空占位，且从未被 require） |

其中 `colors/matugen.lua` 这个设计值得说一下。Neovim 的规范是 `:colorscheme {name}` 会在 runtimepath 里搜 `colors/{name}.lua`（`:help :colorscheme`），所以把配色入口放这里，`:colorscheme matugen` 就能直接用、`:colo` 补全也认。

另外查了 base16-nvim 的源码，确认它**不会设 `vim.g.colors_name`**：

```bash
grep -n "colors_name" ~/.local/share/nvim/lazy/base16-nvim/lua/base16-colorscheme.lua
# 输出: 空
```

而 Neovim 自己的 `runtime/colors/vim.lua` 是显式设的（`vim.g.colors_name = 'vim'`），说明这是配色文件的责任。不设的后果是 `colors_name` 会停留在之前的值（原来一直是 `darkblue`），状态栏之类读它的插件会显示错误信息。所以在 `colors/matugen.lua` 里从文件名推导出来设上：

```lua
vim.g.colors_name = vim.fn.fnamemodify(debug.getinfo(1, "S").source:sub(2), ":t:r")
```

同时把 `options.lua` 里硬编码的 `vim.cmd.colorscheme("darkblue")` 去掉 —— 它会造成启动闪色，并让 `colors_name` 和实际配色不一致。兜底逻辑挪到插件的 `config` 里。

**一个兼容性确认**：Noctalia 的 `apply.sh` 里有这段逻辑 ——

```bash
if [ ! -f "$plugin_file" ] && ! grep -rl "base16-nvim" "${plugins_dir}" 2>/dev/null | grep -q .; then
    # 生成 lua/plugins/base16.lua
fi
```

它会检查 `lua/plugins/` 下有没有文件提到 `base16-nvim`。我的 `plugins/colorscheme.lua` 里有这个字符串，所以它**不会**再重复生成一份插件声明，两边天然兼容，不需要额外处理。

顺带也解释了原始报错的来历：`apply.sh` 的 else 分支（检测到没装 lazy 时）会往 `init.lua` 追加 `local ok, matugen = pcall(require, 'matugen'); if ok then matugen.setup() end` —— 那两行就是这么来的。

### 8.2 第二阶段：lazygit

- 二进制：GitHub release v0.64.1 装到 `~/.local/bin/`（apt 只有 0.57.0；sha256 已校验）
- 插件：`kdheepak/lazygit.nvim` + plenary

**为什么不用 snacks.nvim 的 lazygit**：snacks 会**按 nvim 配色自动生成 lazygit 主题**，而 Noctalia 的社区模板里已经有 lazygit 一项，两者会抢 lazygit 的配置。kdheepak 这个只负责在浮窗里调起命令，不碰 lazygit 配置，配色继续归 Noctalia。

顺手把 `MasonUninstall`、`MasonLog` 补进了 `cmd` 列表 —— 一开始只列了 `Mason`/`MasonInstall`/`MasonUpdate`，导致要卸 jdtls 时 `MasonUninstall` 命令不存在（`E492: Not an editor command`）。

### 8.3 第三阶段：文件树 + 模糊搜索

- `neo-tree.nvim`（v3.x）+ nui + web-devicons
- `telescope.nvim` + `telescope-fzf-native`（现场用 cc 编译出 `libfzf.so`）

两个决策：

**telescope 不锁 `0.1.x` 分支**。官方 README 让你锁，但它自己的 issue #3524 说明 `0.1.x` 已经一年多没更新、master 反而更稳定，所以用 lazy 默认的 master。

**不依赖 `fd`**。`command -v fd` 虽然有输出，但指向的是 `~/.pi/agent/bin/fd`（pi 自带的，别的进程未必看得到），所以让 telescope 走系统级的 `rg`（`/usr/bin/rg`）。

neo-tree 的配置里有两点是实测后加的：`never_show = { ".git" }`（`hide_dotfiles = false` 会把 `.git` 也列出来，很吵），以及 `["<space>"] = "none"`（空格是 leader，不解绑会被树的按键吃掉）。

### 8.4 第四阶段：LSP

架构上和网上多数教程不一样，因为这台机器是 **Neovim 0.12.3**：

| 组件 | 角色 |
| --- | --- |
| `mason.nvim` | 只负责把 server 下载到 `~/.local/share/nvim/mason/`，**不碰 apt** |
| `nvim-lspconfig` | 只当「各 server 默认配置的集合」用 |
| `vim.lsp.config()` / `vim.lsp.enable()` | Neovim 原生接口，写覆盖配置 |
| `mason-lspconfig` | 桥接，装好的自动 enable |

nvim-lspconfig 官方文档自己写着：**"It has no API or framework. It is not required for Nvim LSP support."**

**省掉的插件**（都实测确认过内置可用）：

```text
vim.lsp.completion.enable: true       → 不用 nvim-cmp / blink.cmp
vim.o.autocomplete 可读:   true       → 0.12 的原生自动触发
vim.snippet.expand:        true       → 不用 LuaSnip 展开 LSP 片段
gc / gcc:                  已内置    → 不用 Comment.nvim
```

**内置 LSP 快捷键**（实测 `--clean` 下就存在，不重复定义）：

```text
grn  gra  grr  gri  grt  gO  ]d  [d   全部已内置
```

只补了内置没有的：`gd`、`gD`、`<leader>ld`。

**语言取舍过程**：一开始按需求装了 C/C++、Rust、Python、Java、C# 五种，中途调整两次 ——

- Java 去掉（用 IntelliJ IDEA 写）。jdtls 已经装上了（55MB），用 `:MasonUninstall jdtls` 卸掉。
- C# 去掉（用 Godot 写）。这条顺带绕开了一个卡点：C# 的正路是 `roslyn_ls`（VS Code C# 扩展用的那个官方开源实现，老的 omnisharp 已停止维护），但它要求 .NET 8/9，而本机是 **dotnet 6.0.400**，且 apt 源里只有 6.0：

  ```bash
  dotnet --list-sdks
  # 6.0.400 [/usr/share/dotnet/sdk]
  apt-cache policy dotnet-sdk-9.0
  # 无此包
  ```

最终 6 个 server：`clangd`、`rust_analyzer`、`basedpyright`、`texlab`、`marksman`、`harper_ls`，后来加了 `bashls`。

**harper_ls 的 filetype 必须限制**。它默认会附加到 `c`/`cpp`/`cs`/`python` 等一大堆类型去检查**代码注释**的语法：

```bash
curl -s .../lsp/harper_ls.lua | grep -A5 filetypes
# filetypes = { 'asciidoc', 'c', 'cpp', 'cs', 'gitcommit', ...
```

改成只管写作类文件：

```lua
filetypes = { "markdown", "tex", "plaintex", "gitcommit", "text" },
```

**basedpyright 设 `standard`**。它默认是 `recommended`（比 strict 还严），刚上手会满屏警告。

**rust_analyzer 单文件不报诊断是正常的**。一开始测 `t.rs` 里明显的类型错误却 0 诊断，以为坏了。实际是 rust-analyzer 需要 `Cargo.toml` 才会跑 cargo check。在真实 cargo 项目里测：

```text
诊断数=3
  行2: mismatched types  expected `i32`, found `&str`
  行2: expected due to this
```

**texlab 的 chktex 要保存后才出诊断**。打开时 0 诊断，`:w` 之后出现：

```text
保存后诊断数=3
  行3 [Harper] Use the Unicode ellipsis character (…).
  行3 [ChkTeX] You should use \ldots to achieve an ellipsis.
  行4 [ChkTeX] Double space found.
```

设置键名和官方文档一致（`texlab.chktex.onOpenAndSave`，默认 `false`，这里显式开了）。另外 chktex 会往 stderr 打一句 `Could not find global resource file`（本机没有 `/etc/chktexrc` 也没有 `~/.chktexrc`），属于提示性质，不影响工作。

### 8.5 第五阶段：其余 11 个插件

| 插件 | 用途 | 备注 |
| --- | --- | --- |
| `gitsigns.nvim` | 行内改动标记、单块暂存/回滚、行内 blame | 和 lazygit 互补 |
| `which-key.nvim` | 按前缀键弹出可选项 | leader 映射已三十多个 |
| `conform.nvim` | 格式化框架 | 见下 |
| `vimtex` | LaTeX 编译 / PDF 跳转 / 文本对象 | 锁 v2.18 |
| `render-markdown.nvim` | buffer 内渲染标题、表格、复选框 | 用内置 markdown parser |
| `mini.surround` | 加/删/换包围符号 | 挪到 `gs` 前缀 |
| `mini.pairs` | 自动配对 | |
| `flash.nvim` | 两字符跳转 | 占 `s`/`S` |
| `nvim-colorizer.lua` | 色值显示成颜色块 | 用 catgoose 的维护版本 |
| `trouble.nvim` | 诊断/引用列表 UI | |
| `persistence.nvim` | 按项目存/恢复会话 | |
| `toggleterm.nvim` | 在 nvim 里开终端 | |

**conform 为什么必要**：这台机器全局 `shiftwidth=4`，但 stylua 默认 Tab、clang-format 默认 2 空格。直接用 LSP 的格式化会把文件悄悄改掉。conform 能按语言逐个指定格式化器和参数：

```lua
formatters = {
    shfmt  = { prepend_args = { "-i", "4", "-ci" } },
    stylua = { prepend_args = { "--indent-type", "Spaces", "--indent-width", "4" } },
}
```

并且**刻意不开保存时自动格式化**（`format_on_save` 那段留成注释），格式化是会改文件内容的操作，让它只在按 `<leader>cf` 时发生。

格式化器实测结果：

```text
d.sh  → if [ -z "$x" ]; then / 4空格 echo hi / fi          （shfmt -i 4 生效）
e.py  → import os / import sys 拆开排序，def f(a): 4空格   （ruff 整理 import + 格式化）
a.lua → local function f(a, b) / 4空格 return a + b        （stylua 4空格）
f.c   → int main() { / 2空格 int x = 1;                    （clang-format LLVM 默认）
```

---

## 9. 最终配置总览

### 9.1 文件结构

```text
~/.config/nvim/
├── init.lua                       全局前置（leader、禁用 netrw）→ require config.*
├── colors/
│   └── matugen.lua                配色入口，:colorscheme matugen 可用
└── lua/
    ├── matugen.lua                Noctalia 生成的调色板（勿手改）
    ├── config/
    │   ├── options.lua            行号、缩进、termguicolors
    │   ├── keymaps.lua            （目前为空）
    │   ├── autocmds.lua           （目前为空）
    │   ├── theme.lua              配色名 / 兜底名的单一来源
    │   └── lazy.lua               lazy 引导 + parser rtp 修复
    └── plugins/
        ├── colorscheme.lua        base16-nvim
        ├── neo-tree.lua           文件树
        ├── telescope.lua          模糊搜索
        ├── lazygit.lua            git 操作台
        ├── git.lua                gitsigns
        ├── lsp.lua                mason + lspconfig + 原生 vim.lsp.config
        ├── lint.lua               nvim-lint（zsh）
        ├── format.lua             conform
        ├── latex.lua              vimtex
        ├── markdown.lua           render-markdown
        ├── editing.lua            mini.surround / mini.pairs / flash
        ├── misc.lua               colorizer / trouble / persistence
        ├── terminal.lua           toggleterm
        └── which-key.lua          按键提示
```

### 9.2 配色链路

```text
Noctalia（换壁纸/换主题）
  → ~/.local/state/noctalia/community-templates/neovim/   它自带的模板
  → ~/.config/nvim/lua/matugen.lua                        生成物，勿手改
  → apply.sh 执行 pkill -SIGUSR1 nvim                     热重载信号
  → lua/matugen.lua 里的 handler 重新 setup
  → colors/matugen.lua → base16-nvim                      落成高亮组
```

这条链路在整个过程里被动验证了多次：`Normal bg` 依次出现过 `#23231a`（黄绿）、`#1d1e21`、`#052738`（蓝），每次都和当时的桌面配色一致。

**白拿的两项**：base16-nvim 定义了 **128 组 treesitter 高亮**（`@function`、`@variable`、`@comment.error`…）和 **23 组 Diagnostic 高亮**，所以内置 treesitter 的语法高亮和 LSP 的报错波浪线都自动跟着壁纸走，不用配一行。

### 9.3 插件与外部程序清单

25 个插件（`lazy-lock.json`）：base16-nvim、conform.nvim、flash.nvim、gitsigns.nvim、lazy.nvim、lazygit.nvim、mason-lspconfig.nvim、mason.nvim、mini.pairs、mini.surround、neo-tree.nvim、nui.nvim、nvim-colorizer.lua、nvim-lint、nvim-lspconfig、nvim-web-devicons、persistence.nvim、plenary.nvim、render-markdown.nvim、telescope-fzf-native.nvim、telescope.nvim、toggleterm.nvim、trouble.nvim、vimtex(v2.18)、which-key.nvim

12 个 mason 包：basedpyright、bash-language-server、clangd、clang-format、harper-ls、marksman、ruff、rust-analyzer、shellcheck、shfmt、stylua、texlab

`~/.local/bin/`：lazygit v0.64.1

磁盘：mason 740MB（basedpyright 295MB + clangd 220MB 是大头）、插件 69MB

---

## 10. 验收结果

### 10.1 各语言 LSP 与诊断实测

| 测试文件 | 挂载的 server | 实际诊断 |
| --- | --- | --- |
| `t.c` | clangd | 2 条，`Incompatible pointer to integer conversion` |
| `t.cpp` | clangd | 1 条，`No matching member function for call to 'push_back'` |
| `src/main.rs`（cargo 项目） | rust_analyzer | 3 条，`mismatched types: expected i32, found &str` |
| `t.py` | basedpyright + ruff | 1 条 |
| `t.tex` | texlab + harper_ls | ChkTeX 的 `\ldots`、`Double space` |
| `t.md` | marksman + harper_ls | Harper 语法建议 |
| `t.sh` | bashls | shellcheck 的 `Double quote to prevent globbing` |
| `t.zsh` | 无 LSP（预期） | nvim-lint，第 2 行 `parse error near ')'` |
| `a.lua` | 无（未装 lua_ls） | — |

### 10.2 其他

```text
启动报错:        无
启动耗时:        28.076 ms
插件:            共 25 个，启动只加载 3 个（其余懒加载）
配色:            colors_name=matugen，Normal bg 与桌面一致
leader 按键:     36 个全部注册，无冲突
buffer 局部键:   gitsigns 8 个全部注册；LSP 的 gd/gD/<leader>ld 在附加后存在
:checkhealth lazy: 无 ERROR / 无 WARNING
内置 treesitter: lua 文件高亮已激活
toggleterm:      成功拉起 /usr/bin/zsh
lazygit:         成功拉起 ~/.local/bin/lazygit
```

---

## 11. 避坑清单

分享给别人时的重点，按重要性排。

### 11.1 Debian/Kali + lazy.nvim 一定要保住 parser 路径

这是本次最值得单独拎出来的一条。发行版把 tree-sitter parser 放 `/usr/lib/nvim/parser/`，而 lazy 默认 `performance.rtp.reset = true` 会把它踢出 runtimepath，结果**打开 .lua 就报 E5113**。在 Arch 之类的发行版上不会出现，所以网上搜不到太多资料。

### 11.2 pcall 要包住真正会抛错的那一句

`pcall(require, 'x')` 只保护加载，不保护 `x.setup()`。

### 11.3 判断集成有没有生效，看产物的 mtime

不要只读配置文件就下结论。文件时间戳和内容特征（缩进风格、注释语言）能直接指出是谁写的。

### 11.4 两个工具写同一个文件，迟早出事

我的 matugen 和 Noctalia 的 neovim 模板都输出 `lua/matugen.lua`，结果就是互相覆盖。确定唯一写手，另一个删掉。

### 11.5 mason 装的格式化器可能被当成 LSP 启用

`automatic_enable = true` 会把所有能对上 lspconfig 条目的已装包都 enable。stylua、ruff 这类「带 `--lsp` 模式的格式化器」就会被误挂。装完 LSP 后务必检查一次实际挂载的客户端名：

```lua
:lua =vim.tbl_map(function(c) return c.name end, vim.lsp.get_clients({bufnr=0}))
```

### 11.6 涉及 GC / 异步 / 时序的推理，必须实测

误判二（signal handle 泄漏）和坑 6.5（termguicolors 竞争）是同一类问题的两面：一个是我以为有问题、实际 GC 兜住了；一个是我以为没问题、实际有时序竞争。**推理链再顺都不能替代跑一次。**

### 11.7 headless 测不出全部问题

`termguicolors` 那个坑在 headless 下会被误判成「headless 的正常现象」。要用伪终端：

```bash
TERM=xterm-256color COLORTERM=truecolor script -qec "nvim ..." /dev/null
```

而且检查要**延迟**做（异步探测要时间）：

```lua
vim.defer_fn(function() ... end, 800)
```

### 11.8 懒加载插件的按键写 `keys` 不写 `config`

写在 `config` 里的 keymap，插件没加载就不存在。

### 11.9 mapleader 和 maplocalleader 不要用同一个键

filetype 插件（vimtex、nvim-jdtls 等）大量使用 `<localleader>`，重合会导致按键行为随文件类型漂移。

### 11.10 改了 lazy spec 的 tag/version，要用 update 不是 restore

`:Lazy! restore` 按 lockfile 恢复，不应用新的版本约束。

### 11.11 shellcheck 不支持 zsh，别硬塞

塞进 bashls 的 filetypes 只会让每个 zsh 文件报 SC1071。

### 11.12 脚本里用 pkill -f 小心自我匹配

bash 进程的命令行里包含你写的模式串，信号会打到自己身上。先 `pgrep` 取 PID 再 `kill`，能用 `pgrep -x`（只匹配进程名）就不用 `-f`。**这个坑我在同一次排查里踩了两次。**

### 11.13 中文 LaTeX 必须用 xelatex 或 lualatex

latexmk 默认调 pdflatex，而 pdflatex 处理不了 UTF-8 中文。更阴的是它会**一边报错一边产出 PDF**，看到有 PDF 就以为成功就错了。vimtex 里改 `vimtex_compiler_latexmk_engines` 的 `'_'` 默认值，单文档用 `% !TEX program = ...` 覆盖。

### 11.14 Wayland 下用 zathura_simple 而不是 zathura

vimtex 的 `zathura` viewer 靠 xdotool（X11 工具）管窗口，Wayland 下必然报 `cannot find Zathura window ID`。`zathura_simple` 不做窗口管理，而 zathura 自带的 D-Bus 单实例会把第二次调用转发给已有窗口，所以功能不受影响。

### 11.15 clang-format 别用 `-style=` 强设风格

会覆盖别人仓库的 `.clang-format`。要自定义就放 `.clang-format` 文件。

### 11.16 装之前先查 0.12 有没有内置

这一轮省掉了 nvim-cmp、LuaSnip、Comment.nvim 三个常见插件，还确认了 0.12 的默认 statusline 已经内置诊断计数和 LSP 进度显示（`vim.diagnostic.status()`、`vim.ui.progress_status()`），lualine 之类属于审美需求而非功能缺失。

---

## 12. 遗留事项

### 12.1 已完成：zathura

过程中补装了（`sudo apt install zathura zathura-pdf-poppler`）。实测结果：

- zathura 2026.07.18，`libpdf-poppler.so` 后端就位
- 能正确打开 vimtex 编译出的中文 PDF
- **Noctalia 的 zathura 主题已生效**，标题高亮颜色跟当前壁纸调色一致
- `--synctex-forward` 和 inverse search 的 `-x` 回调都正确传入

配合改动：viewer 从 `zathura` 换成 `zathura_simple`（见 6.13），默认引擎换成 xelatex（见 6.12）。

`zathura-pdf-mupdf` 在 Kali 源里没有，用 poppler 后端就行。

### 12.2 Rust 标准库跳转

不装的话跳到 `Vec`、`String` 的定义会失败：

```bash
rustup component add rust-src
```

### 12.3 lua_ls 没装

现在编辑 nvim 配置本身是没有 LSP 的（`a.lua → 无`）。要的话：

```vim
:MasonInstall lua-language-server
```

装完记得配 `diagnostics.globals = { "vim" }`，否则会把 `vim` 报成未定义全局变量。

### 12.4 vimtex 锁在 v2.18

等 nvim 升到 0.12.4+，把 `lua/plugins/latex.lua` 里的 `tag = "v2.18"` 删掉即可回 master。

### 12.5 备份

改动前的配置备份在 `~/.config/nvim.bak.20260831-030907`，确认无问题后可删。

---

## 13. 排查方法复盘

这次做对的几件事：

1. **先做对照实验再猜原因**。`nvim --clean` 对比自己的配置，一步就把「系统问题」和「配置问题」分开了。
2. **看产物而不是看配置**。误判一是靠 mtime 和文件内容特征纠正的。
3. **可疑推理一律跑数据**。误判二直接被 `vim.uv.walk` 的计数推翻，省掉一段没必要的 hack。
4. **读源码确认行为**，不靠推测。base16-nvim 不设 `colors_name`、nvim-lint 的 `append_fname` 逻辑、lspconfig 里真的有 `lsp/stylua.lua`、vimtex 抬高版本要求的那条 commit —— 都是直接翻代码得到的。
5. **把中间产物打出来**。zsh linter 那个坑，靠把 parser 包一层打印 `RAW_OUTPUT` 一步定位。诊断为 0 时，要区分「命令没跑」「跑了没输出」「有输出没解析出来」「解析了被过滤掉」，不打印中间值就只能瞎猜。
6. **官方文档也要交叉验证**。telescope 的 README 让锁 `0.1.x`，它自己的 issue 却说 master 更稳；网上多篇博客说 shellcheck 支持 zsh，官方 SC1071 明确否认。

做得不好的：

1. **抄配置片段没核对版本**。clangd 的 `--function-arg-placeholders` 就是照抄旧教程，clangd 22 已经改了。
2. **一开始下结论太快**。误判一如果先看一眼 `~/.local/state/noctalia/community-templates/`，整个误判三（多余的 matugen）都不会发生。
3. **只在 headless 里验证**。termguicolors 那个坑差点被放过。

---

## 14. 参考链接

### 官方文档

- Neovim LSP：<https://neovim.io/doc/user/lsp/>
- Neovim `:colorscheme` / syntax：<https://neovim.io/doc/user/syntax.html>
- lazy.nvim 安装（Structured Setup）：<https://lazy.folke.io/installation>
- lazy.nvim 配置项：<https://lazy.folke.io/configuration>
- lazy.nvim 插件 spec：<https://lazy.folke.io/spec>
- texlab 配置项：<https://github.com/latex-lsp/texlab/wiki/Configuration>
- shellcheck SC1071（不支持 zsh）：<https://www.shellcheck.net/wiki/SC1071>
- Noctalia 的 Neovim 模板说明：<https://docs.noctalia.dev/noctalia/templates/community/neovim/>

### GitHub 仓库

- base16-nvim：<https://github.com/RRethy/base16-nvim>
- lazy.nvim：<https://github.com/folke/lazy.nvim>
- mason.nvim：<https://github.com/mason-org/mason.nvim>
- mason-lspconfig.nvim：<https://github.com/mason-org/mason-lspconfig.nvim>
- nvim-lspconfig：<https://github.com/neovim/nvim-lspconfig>
- nvim-lint：<https://github.com/mfussenegger/nvim-lint>
- conform.nvim：<https://github.com/stevearc/conform.nvim>
- vimtex：<https://github.com/lervag/vimtex>
- neo-tree.nvim：<https://github.com/nvim-neo-tree/neo-tree.nvim>
- telescope.nvim：<https://github.com/nvim-telescope/telescope.nvim>
- telescope issue #3524（0.1.x 与 master 之争）：<https://github.com/nvim-telescope/telescope.nvim/issues/3524>
- lazygit：<https://github.com/jesseduffield/lazygit>
- lazygit.nvim：<https://github.com/kdheepak/lazygit.nvim>
- gitsigns.nvim：<https://github.com/lewis6991/gitsigns.nvim>
- render-markdown.nvim：<https://github.com/MeanderingProgrammer/render-markdown.nvim>
- mini.nvim（surround / pairs）：<https://github.com/nvim-mini>
- flash.nvim：<https://github.com/folke/flash.nvim>
- trouble.nvim：<https://github.com/folke/trouble.nvim>
- which-key.nvim：<https://github.com/folke/which-key.nvim>
- persistence.nvim：<https://github.com/folke/persistence.nvim>
- toggleterm.nvim：<https://github.com/akinsho/toggleterm.nvim>
- nvim-colorizer.lua（catgoose 维护版）：<https://github.com/catgoose/nvim-colorizer.lua>
- harper（harper-ls）：<https://github.com/Automattic/harper>
- marksman：<https://github.com/artempyanykh/marksman>
