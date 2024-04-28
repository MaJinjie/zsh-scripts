1. find-files.zsh 查找文件
2. search-string.zsh 搜索文本

# feature

1. find-files.zsh 能够同时传入文件和目录，并使用lscolors颜色输出到fzf面板。
2. search-string.zsh 能够根据匹配的条目数动态地调整fzf预览窗口的大小, 意味着你不需要关心何时打开预览窗口。（暂时没有添加切换键，因为这效果很好）
3. search-string.zsh 支持fzf和rg两种模式的切换，并且可以在fzf搜索过程中动态地修改`max-depth hidden no-ignore iglob type type-not`等所有可以影响到匹配文件的参数，非常便捷。
4. search-string.zsh 自主选择是否在开始时启用搜索，只有在传入正则匹配时才会启用初始查找。
5. find-files.zsh search-string.zsh 支持大多数常用的参数。
6. find-files.zsh search-string.zsh 定义了一种规则来替代命令行中选项参数的传入，这使得它使用起来更加方便。
7. find-files.zsh search-string.zsh 都能够正确解析带有特殊字符的文件。

# install

`git clone https://github.com/MaJinjie/fzf-fd-rg.git`

# usage

## 1 aliases

```bash
ff "fzf-fd -H -d6 --split"
ss "fzf-rg -d6 --window full --split --args \"-j 2\""
```

# introduce

## find-files

### 1 help

```bash
# 1. 能够解析常用的参数
# 2. 接受目录或文件
# 3. 尽可能地解释用户传入的参数(很完美，几乎完全避免了和文件名或模式冲突)
#   1. ,10xf, max-depth=10 --type=x --type=f
#   2. 1d,1h [1d,1h] 2024-04-20,2024-04-25
#   3. cc,py, extensions=cc ...
#   4. h,, exclude=*.h
#   5. ,0 去除depth标志
#   6. ,hi 切换--hidden --no-ignore
usage: ff [OPTIONS] [DIRECTORIES or Files]

OPTIONS:
    -g glob-based search
    -p full-path
    -t char set, file types dfxlspebc (-t t...)
    -T string, changed after time ( 1min 1h 1d(default) 2weeks "2018-10-27 10:00:00" 2018-10-27)
    -d int, max-depth
    -H bool, --hidden
    -I bool, --no-ignore
    -e extensions
    -E exclude glob pattern
    -o select + -O
    -q Cancel the first n matching file names (Optional default 1)

    --help
    --window full none
    --split Explain the parameters passed in by the user as much as possible
    --args pass -> fd

KEYBINDINGS:
    ctrl-s horizontal direction splitw
    ctrl-v vertical direction splitw
    alt-e subshell editor
    alt-enter open dirname file
    enter current open file
```

### 2 split

```bash
# 下面是经过多次修改精简后的过程, 它包含了：
# 1. --changed-after --changed-before
# 2. --max-depth
# 3. --hidden --no-ignore
# 4. --extension --exclude
__split() {
    local item
    for item in "$@"; do
        { [[ -e $item ]] && ((o_q-- < 1)) } && {
            [[ -d $item ]] && Directories+=$item || Files+=$item
            continue
        }
        ((o_split)) && [[ $item =~ , ]] && {
            if [[ $item =~ ^(\\.|[[:digit:]]+[mhdwMy]|[[:digit:]]{4}-[[:digit:]]{2}-[[:digit:]]{2}|[[:digit:]]{4}-[[:digit:]]{2}-[[:digit:]]{2}[[:space:]][[:digit:]]{2}:[[:digit:]]{2}:[[:digit:]]{2}),(\\.|[[:digit:]]+[mhdwMy]|[[:digit:]]{4}-[[:digit:]]{2}-[[:digit:]]{2}|[[:digit:]]{4}-[[:digit:]]{2}-[[:digit:]]{2}[[:space:]][[:digit:]]{2}:[[:digit:]]{2}:[[:digit:]]{2})?$ ]]; then
                [[ $item =~ ^[^.] ]] && o_args+="--changed-after=${item%,*}"
                [[ $item =~ [^.,]$ ]] && o_args+="--changed-before=${item#*,}"
            else
                while (($#item)); do
                    case $item in
                        ,#) break ;;
                        [[:lower:]]##,,*) o_excludes+="*.${item[(ws/,,/)1]}"; item=${item#*,,} ;;
                        [[:lower:]]##,*) o_extensions+=${item[(ws/,/)1]}; item=${item#*,} ;;
                        *)
                            [[ $item =~ h ]] && { ((idx=$o_args[(I)--hidden])) && unset "o_args[idx]" || o_args+=--hidden }
                            [[ $item =~ i ]] && { ((idx=$o_args[(I)--no-ignore])) && unset "o_args[idx]" || o_args+=--no-ignore }
                            [[ $item =~ [[:digit:]] ]] && { unset "o_args[${o_args[(i)--max-depth*]}]"; ((idx=${$(grep -oP '\d+' <<<$item)[(w)-1]})) && o_args+="--max-depth=$idx" } # 获取数字序列的最后一个
                            o_types+=( ${(u)${(s//)item}:#[[:digit:]hi,]} )
                            break
                            ;;
                    esac
                done
            fi
            continue
        }
        [[ -z $Pattern ]] || report_error "$item error, pattern is exists"
        Pattern=$item
    done
}

# 解释： 以sh或zsh作为拓展名，不包含*.cc(也就是不以cc作为拓展名)，深度为10，切换--hidden --no-ignore参数，可执行文件 修改时间为[1d,1h](一天前到1小时前), 正则模式为bash
ff sh,zsh,cc,,10hix 1d,1h bash
```

**需要注意，下同**：

1. `h` 是切换`--hidden`参数，也就是说你之前传入`--hidden`，它就取消，否则就加上。`i` 同理
2. `0` 0深度不可能吧，所以它被解释为取消之前传入的`--max-depth`
3. `<time>,[.] .,<time>` 这是时间范围的单边界区间的正确传入方式

## search-string

> 交互式查找字符串

### 1 help

```bash
# 1. 能够解析常用的参数
# 2. 接受目录或文件
# 3. 能够在fzf过滤时，解释--iglob --hidden --no-ignore --max-depth --type --type-not
# 4. 尽可能解释用户传入的参数
#   1. ,10 max-depth=10
#   2. cpp,c, --type=cpp
#   3. cc,, --type-not=c
#   4. ,0 去除depth标志
#   5. ,hi 切换--hidden --no-ignore
# 5. 解决了\b的转义问题
usage: ss [OPTIONS] [pattern] [DIRECTORIES or Files]
    OPTIONS:
        -t file types,  Comma-separated
        -T file types(not),  Comma-separated
        -d int max-depth
        -H bool --hidden
        -I bool --no-ignore
        -q Cancel the first n matching file names (Optional, default 1)
        -w world regex
        -u[uu] (Optional default -u)

        --help
        --window full none
        --split Explain the parameters passed in by the user as much as possible
        --args pass -> rg

    KEYBINDINGS:
        ctrl-s horizontal direction splitw
        ctrl-v vertical direction splitw
        alt-e subshell editor
        alt-enter open dirname file
        enter current open file
```

### 2 split

```bash
# 下面是经过多次修改精简后的过程, 它包含了：
# 1. --max-depth
# 2. --hidden --no-ignore
# 3. --type --type-not
__split() {
    local item
    for item in "$@"; do
        { [[ -e $item ]] && ((o_q-- < 1)) } && {
            [[ -d $item ]] && Directories+=$item || Files+=$item
            continue
        }
        ((o_split)) && [[ $item =~ , ]] && {
            while (($#item)); do
                case $item in
                    ,#) break ;;
                    [[:lower:]]##,,*) o_types+=--type-not=${item[(ws/,,/)1]}; item=${item#*,,} ;;
                    [[:lower:]]##,*) o_types+=--type=${item[(ws/,/)1]}; item=${item#*,} ;;
                    *)
                        [[ $item =~ h ]] && { ((idx=$o_dynamic[(I)--hidden])) && unset "o_dynamic[idx]" || o_dynamic+=--hidden }
                        [[ $item =~ i ]] && { ((idx=$o_dynamic[(I)--no-ignore])) && unset "o_dynamic[idx]" || o_dynamic+=--no-ignore }
                        [[ $item =~ [[:digit:]] ]] && { unset "o_dynamic[${o_dynamic[(i)--max-depth*]}]"; ((idx=${$(grep -oP '\d+' <<<$item)[(w)-1]})) && o_dynamic+="--max-depth=$idx" } # 获取数字序列的最后一个
                        break
                        ;;
                esac
            done
            continue
        }
        [[ -z $Pattern ]] || report_error "$item error, pattern is exists"
        Pattern=$item
    done
}
```

### 3 Oneline filtering

```bash
change_reload="
setopt extended_glob
local dynamic=($o_dynamic) types=($o_types) globs=()
[[ \$FZF_QUERY == *--* ]] && for item in \${(s/ /)\${FZF_QUERY##*--}}; do
    while ((\$#item)); do
        case \$item in
            ,#) break ;;
            h) ((idx=\$dynamic[(I)--hidden])) && unset \\\"dynamic[idx]\\\" || dynamic+=--hidden; item= ;;
            i) ((idx=\$dynamic[(I)--no-ignore])) && unset \\\"dynamic[idx]\\\" || dynamic+=--no-ignore; item= ;;
            <->) unset \\\"dynamic[\$dynamic[(i)--max-depth*]]\\\"; ((\$item)) && dynamic+=--max-depth=\$item; item= ;;
            [[:lower:]]##,,*) types+=--type-not=\${item[(ws/,,/)1]}; item=\${item#*,,} ;;
            [[:lower:]]##,*) types+=--type=\${item[(ws/,/)1]}; item=\${item#*,} ;;
            *) globs+=\\\"'--iglob=\$item'\\\"; item= ;;
        esac
    done
done
print -r \\\"reload($cmd \${(u)dynamic} \${(u)types} \${(u)globs} -- '\${\${FZF_QUERY%--*}%% #}' ${Directories} ${Files} || true)\\\"
"
```

### 4 preview

我目前定义预览窗口的调整机制是将预览窗口的变化分为以下几个阶段：

1. 行数大于`b1`或匹配条目为0,不显示
2. 行数`b1(10000) - b2(1000)`，预览窗口比例在`per1(0.0) - per2(0.3)`之间，随匹配的行数线性变化
3. 行数`b2(1000) - b3(100)`，预览窗口比例在`per2(0.3) - per3(0.6)`之间，随匹配的行数线性变化
4. 行数小于b3(100), 匹配的行数 + 预览窗口大小 < 总高度，预览窗口增大做补充

**通过修改`b[1-3] per[1-3]`,可以重新调整比例。**

```bash
# 由于在我的系统中，FZF_PREVIEW_LINES始终为空，所有我使用文件代替
change_preview="
typeset lines=\$((FZF_LINES - 3))  match_count=\$FZF_MATCH_COUNT preview_lines=\${FZF_PREVIEW_LINES:-\${\$(<$file_preview_size):-0}}
typeset b1=10000 b2=1000 b3=100 per1=0 per2=30 per3=60 result
if ((match_count == 0 || match_count > b1)); then
    result=0
elif ((match_count > b2)); then
    result=\$(( ((b1 - match_count) * (per2 - per1) / (b1 - b2)  + per1) * lines / 100 ))
elif ((match_count > b3)); then
    result=\$(( ((b2 - match_count) * (per3 - per2) / (b2 - b3)  + per2) * lines / 100 ))
elif ((match_count > (lines - preview_lines))); then
    result=\$preview_lines
else
    result=$\((lines - match_count))
fi
[[ -z \$FZF_PREVIEW_LINES ]] && print \$result > $file_preview_size
print \\\"change-preview-window(\$result)\\\"
"
```

# supplement

## 1 fzf

```bash
FZF_COLORS="--color=hl:yellow:bold,hl+:yellow:reverse,pointer:032,marker:010,bg+:-1,border:#808080"
FZF_HISTFILE="$XDG_CACHE_HOME/fzf/history"
FZF_FILE_PREVIEW="([[ -f {} ]] && (bkt --ttl 1m -- bat --style=numbers --color=always -- {}))"
FZF_DIR_PREVIEW="([[ -d {} ]] && (bkt --ttl 1m -- eza --color=always -TL4  {} | bat --color=always))"
FZF_BIN_PREVIEW="([[ \$(file --mime-type -b {}) = *binary* ]] && (echo {} is a binary file))"

export FZF_COLORS FZF_HISTFILE FZF_FILE_PREVIEW FZF_DIR_PREVIEW FZF_BIN_PREVIEW


# return
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
export FZF_GREP_TMUX_OPTS="-p100%,80%"
```

## others

1. 如果没有`lscolors`, 请使用对应的包管理工具下载或访问https://github.com/sharkdp/lscolors下载。
2. 如果不想要`report`颜色输出，就直接修改源脚本改为`print`, 注意脚本中的report路径
