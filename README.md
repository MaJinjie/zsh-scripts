1. fzf-fd 查找文件
2. fzf-rg 搜索文本
3. fzf-of 搜索最近nvim打开的文件
4. fzf-man 查看man页面

# feature

1. fzf-fd 能够同时传入文件和目录，并使用lscolors颜色输出到fzf面板。
2. fzf-rg 能够根据匹配的条目数动态地调整fzf预览窗口的大小, 意味着你不需要关心何时打开预览窗口。
3. fzf-rg 支持fzf和rg两种模式的切换，并且可以在fzf搜索过程中动态地修改`max-depth hidden no-ignore iglob type `等所有可以影响到匹配文件的参数。
4. fzf-rg 初始时，如果未传入查询字符串，fzf面板为空，但会在后台执行一次该命令，直到正式开启搜索，即进行热缓存,加快搜索速度。
5. fzf-rg 为了减少刷新（reload）的次数，只有当`--`后字符串以空格结尾时，才会执行重新搜索。
6. fzf-rg fzf-fd 支持大多数常用的选项参数。
7. fzf-rg fzf-fd 定义了一种规则来替代命令行中常用选项参数的传入，这使得它使用起来更加方便。
8. fzf-rg fzf-fd 能够正确解析带有特殊字符的文件。

# install

## 1 添加到path

1. `git clone https://github.com/MaJinjie/zsh-scripts.git`
2. 把目录加入到环境变量中

## 2 zint插件管理

```bash
# 需要使用NICHOLAS85/z-a-linkbin ice
zinit ice lbin'fzf-*'
zinit light MaJinjie/zsh-scripts

```

# usage

## 1 aliases

> 我的别名

```bash
 ff "fzf-fd --alias --split --popup -H -d6"
 ss "FZF_TMUX_OPTS='-p100%,100%' fzf-rg --alias --split --popup --args \"--threads=2\" --warm -d6"
 fo "fzf-of --popup"
 fm "fzf-man --popup"
```

# introduce

## find-files

### 1 help

```bash
usage: ff [OPTIONS] [DIRECTORIES or Files] [-- fd options]

OPTIONS:
    fd:
    -g glob-based search
    -p full-path
    -t char set, file types dfxlspebc (-t t...)
    -d int, max-depth
    -H bool, --hidden
    -I bool, --no-ignore
    -e extensions, Comma-separated
    --args pass -> fd

    others:
    -q | --query force initial query string
    --popup start fzf-tmux
    --root-dir relative path -> abs dir
    --split Explain the parameters passed in by the user as much as possible, HhIig,exts
    --alias use directory alias

    --help

KEYBINDINGS:
    alt-e subshell editor
    alt-enter open dirname file
    enter current open file
```

### 2 split

形式如：`ff [duration | [dfxlspebchHiIg\d],[extension[,extension]]]`

**介绍**：

1. `duration` 一个时间段，例如`1d,1h => 1d-1h` `1d,, => 1d-`
2. `dfxlspebc` 分别对应要搜索的文件类型
3. `hHiI<digit>` 分别代表切换隐藏文件、忽略文件以及指定搜索深度
4. `g` 指定当前git仓库的`--git-dir`为`root-dir`
5. `extension` 传递给`fd --extension`

## search-string

> 交互式查找字符串

### 1 help

```bash
usage: ss [OPTIONS] [pattern] [DIRECTORIES or Files] [-- rg options]
    OPTIONS:
        rg:
        -t file types,  Comma-separated
        -g iglob pattern,  Comma-separated
        -H bool --hidden
        -I bool --no-ignorehas
        -d int max-depth
        -m --max-count
        --sort format (sort|sortr):default:toggle_val
        --args pass -> rg

        others
        -q force initial query string
        --warm warm-cache when initial querystring is empty
        --popup start fzf-tmux
        --split Explain the parameters passed in by the user as much as possible
        --alias use directory alias

        --root-dir | -r filter base dir
        --help
    KEYBINDINGS:
        alt-e subshell editor
        alt-enter open dirname file
        enter current open file
```

### 2 split

形式如：`rg [hHiIsSoOg\d],[type[,type]]]`

**介绍**：

1. `Ss` 是否排序搜索条目
2. `Oo` 是否搜索条目文件唯一
3. `hHiI<digit>` 分别代表切换隐藏文件、忽略文件以及指定搜索深度
4. `g` 指定当前git仓库的`--git-dir`为`root-dir`
5. `type` 传递给rg的`--type`

### 3 Oneline filtering

```bash
# 使用该正则实现搜索字符串和其他项的分离
[[ {q} =~ ^(?:[[:blank:]]*(.*?)[[:blank:]]*)(?:-|--[[:blank:]]*(.*?)[^[:blank:]]*)?$ ]]

# example1 搜索--root-dir目录的linting/目录下，cpp sh类型文件，与*sh | *h 匹配的文件
bash -- ,cpp,sh *.sh *h linting/
```

### 4 preview

我目前定义预览窗口的调整机制是将预览窗口的变化分为以下几个阶段：

1. 行数大于`b1`或匹配条目为0,不显示
2. 行数`b1(10000) - b2(1000)`，预览窗口比例在`per1(0.0) - per2(0.3)`之间，随匹配的行数线性变化
3. 行数`b2(1000) - b3(100)`，预览窗口比例在`per2(0.3) - per3(0.6)`之间，随匹配的行数线性变化
4. 行数小于b3(100), 匹配的行数 + 预览窗口大小 < 总高度，预览窗口增大做补充

**通过修改`b[1-3] per[1-3]`,可以重新调整比例。**

### 5 inital-search

如果字符串为空，那么就在后台执行一遍命令。
目的：形成热缓存，加快之后的查找速度。

# supplement

## 1 fzf

```bash
FZF_COLORS="--color=hl:yellow:bold,hl+:yellow:reverse,pointer:032,marker:010,bg+:-1,border:#808080"
FZF_HISTFILE="$XDG_CACHE_HOME/fzf/history"
FZF_FILE_PREVIEW="([[ -f {} ]] && (bkt --ttl 1m -- bat --style=numbers --color=always -- {}))"
FZF_DIR_PREVIEW="([[ -d {} ]] && (bkt --ttl 1m -- eza --color=always -TL4  {} | bat --color=always))"
FZF_BIN_PREVIEW="([[ \$(file --mime-type -b {}) = *binary* ]] && (echo {} is a binary file))"

export FZF_COLORS FZF_HISTFILE FZF_FILE_PREVIEW FZF_DIR_PREVIEW FZF_BIN_PREVIEW

export FZF_DEFAULT_OPTS=" \
--marker='▍' \
--scrollbar='█' \
--ellipsis='' \
--cycle \
$FZF_COLORS \
--reverse \
--info=inline \
--ansi \
--multi \
--height=80% \
--tabstop=4 \
--scroll-off=2 \
--history=$FZF_HISTFILE \
--jump-labels='abcdefghijklmnopqrstuvwxyz' \
--preview-window :hidden \
--preview=\"($FZF_FILE_PREVIEW || $FZF_DIR_PREVIEW) 2>/dev/null | head -300\" \
--bind='home:beginning-of-line' \
--bind='end:end-of-line' \
--bind='tab:toggle+down' \
--bind='btab:up+toggle' \
--bind='esc:abort' \
--bind='ctrl-u:unix-line-discard' \
--bind='ctrl-w:backward-kill-word' \
--bind='ctrl-y:execute-silent(wl-copy -n {+})' \
--bind='ctrl-/:change-preview-window(up,60%,border-down|right,60%,border-left)' \
--bind='ctrl-\:toggle-preview' \
--bind='ctrl-q:abort' \
--bind='ctrl-l:clear-selection+first' \
--bind='ctrl-j:down' \
--bind='ctrl-k:up' \
--bind='ctrl-p:prev-history' \
--bind='ctrl-n:next-history' \
--bind='ctrl-b:beginning-of-line' \
--bind='ctrl-e:end-of-line' \
--bind='ctrl-s:toggle-all' \
--bind='alt-a:toggle-sort' \
--bind='alt-j:preview-down' \
--bind='alt-k:preview-up' \
--bind='alt-w:toggle-preview-wrap'
--bind='alt-b:preview-page-up' \
--bind='alt-f:preview-page-down' \
--bind='enter:accept' \
--bind='?:jump' \
"

export FZF_TMUX_OPTS="-p70%,80%"
```

## others

1. 接下来只对zsh版本做维护
2. 如果没有`lscolors`, 请使用对应的包管理工具下载或访问https://github.com/sharkdp/lscolors。(如果最新版本有问题，就使用`v0.15.0` )
