# Vim

> 每晚 20:30 自动补充完善。先记住：Vim 是模式编辑器，`Esc` 回到普通模式是所有操作的前提。

## 模式

- **普通模式**（Normal）：移动、删除、复制，打开即是
- **插入模式**（Insert）：`i`（光标前）/`a`（光标后）/`o`（下方新行）/`O`（上方新行）进入，`Esc` 退出
- **可视模式**（Visual）：`v` 字符选择 / `V` 行选择 / `Ctrl-v` 块选择
- **命令模式**：`:` 输入命令，`/` 搜索

## 移动（普通模式，肌肉记忆优先级最高）

```
h j k l        左 下 上 右
w / b          下一个词头 / 上一个词头
0 / $          行首 / 行末
gg / G         文件头 / 文件尾
:n / nG        跳到第 n 行
%              跳到匹配的括号
Ctrl-f / Ctrl-b   翻页下 / 上
*              搜索光标处单词
```

## 编辑

```
x        删一个字符
dd       删整行；5dd 删 5 行
dw / d$  删到词尾 / 行尾
yy       复制整行；p 粘贴到下方，P 上方
u / Ctrl-r   撤销 / 重做
.        重复上一次修改（神器）
>> / <<  行缩进 / 反缩进
```

## 搜索替换

```
/关键字    向下搜索，n 下一个，N 上一个
:%s/旧/新/g        全文替换
:%s/旧/新/gc       逐个确认替换
:s/旧/新/g         仅当前行
```

## 分屏与标签

```
:sp 文件名    横分屏；:vsp 文件名  竖分屏
Ctrl-w w      切分屏；Ctrl-w h/j/k/l  方向切
:tabnew       新标签页；gt / gT  切换
```

## 退出保存

```
:w      保存；:w! 强制保存
:q      退出；:q! 不保存退出
:wq / :x   保存退出
ZZ      保存退出（普通模式快捷）
```

## 常用配置（~/.vimrc）

```vim
set number relativenumber   " 行号 + 相对行号
set tabstop=4 shiftwidth=4 expandtab  " 4 空格缩进
set ignorecase smartcase    " 搜索忽略大小写（有大写时敏感）
set hlsearch incsearch      " 高亮 + 增量搜索
syntax on                   " 语法高亮
set clipboard=unnamedplus   " 与系统剪贴板互通（需 +clipboard）
```

## 5 个常见坑

1. **卡在插入模式出不来**：狂按 `Esc` 回到普通模式再操作；`:q!` 强制退出不保存。
2. **方向键乱码**：终端类型不对，先 `Esc` 再 `:set term=ansi` 或换个终端。
3. **粘贴代码缩进错乱**：粘贴前 `:set paste`，粘完 `:set nopaste`。
4. **`u` 撤销不够用**：Vim 撤销是树状的，`g-` / `g+` 按时间回退分支。
5. **想用鼠标**：`:set mouse=a` 开启鼠标支持（终端 Vim）。

## 疑难杂症

### 1. 远程粘贴代码缩进全乱（bracketed paste）
- **现象**：从本地复制大段代码粘到远程 vim，每行缩进越来越深。
- **原因**：开了 `autoindent`，粘贴时 vim 把每行都当手打的又缩进一次。
- **解决**：粘贴前 `:set paste`，粘完 `:set nopaste`；或用 `"+p` 走系统剪贴板（需 +clipboard）；新版 vim/neovim 一般自动处理 bracketed paste。

### 2. 打开文件提示 swap 文件已存在
- **现象**：`Found a swap file by the name ...`，问你要恢复还是删除。
- **原因**：上次 vim 非正常退出（断网、kill），swap 残留；或另一个 vim 正开着这文件。
- **解决**：确认没别的 vim 在编辑，选 `D` 删除 swap；经常断网的话把 `directory` 指到本地 tmp，避免残留。

## 最小可用 vimrc

```vim
" ~/.vimrc（Windows 下是 ~/_vimrc）
set nocompatible
syntax on
set number relativenumber
set tabstop=4 shiftwidth=4 expandtab
set autoindent smartindent
set ignorecase smartcase        " 搜索忽略大小写，有大写时区分
set incsearch hlsearch
set mouse=a
set clipboard=unnamedplus       " 和系统剪贴板互通（需 +clipboard）
set encoding=utf-8
set nobackup noswapfile         " 不想要 .swp 残留就关掉
" jk 快速退出插入模式
inoremap jk <Esc>
" 空格当 leader 键
let mapleader=" "
nnoremap <leader>w :w<CR>
nnoremap <leader>q :q<CR>
```

- 改完 `:source ~/.vimrc` 即时生效，不用重启。
- 想再进一步直接上 Neovim + lazy.nvim，但先把原生 vim 玩熟。

## 多文件：buffer / window / tab

```vim
:e file2.py        " 再开一个文件（当前窗口切换，旧文件进 buffer）
:ls               " 列出所有 buffer
:b2               " 切到 2 号 buffer
:bn / :bp         " 下一个 / 上一个 buffer
:bd               " 关掉当前 buffer

:sp file3.py      " 横向分屏
:vsp file4.py     " 纵向分屏
Ctrl+w h/j/k/l    " 分屏间跳转
Ctrl+w =          " 分屏等宽

:tabnew file5.py  " 新标签页
gt / gT           " 标签页左右切
```

- 一句话区分：buffer 是"打开的文件"，window 是"看 buffer 的格子"，tab 是"格子的布局"。
- 日常够用组合：`buffer + 分屏`，tab 很少需要。

## 宏录制（重复劳动终结者）

```vim
qa          " 开始录制到寄存器 a
I"<Esc>A",<Esc>j   " 示例操作：行首加引号、行尾加逗号、下一行
q           " 结束录制
@a          " 执行一次
10@a        " 执行 10 次
@@          " 再来一次上次的宏
```

- 批量改 100 行格式：录 3 秒，`99@a`，喝咖啡。
- 宏存在寄存器里，`:reg a` 查看；复杂宏可以粘出来存进 vimrc 长期用。
