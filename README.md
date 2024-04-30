1. find-files.zsh 查找文件
2. search-string.zsh 搜索文本

# feature

1. find-files.zsh 能够同时传入文件和目录，并使用lscolors颜色输出到fzf面板。
2. search-string.zsh 能够根据匹配的条目数动态地调整fzf预览窗口的大小, 意味着你不需要关心何时打开预览窗口。（暂时没有添加切换键，因为这效果很好）
3. search-string.zsh 支持fzf和rg两种模式的切换，并且可以在fzf搜索过程中动态地修改`max-depth hidden no-ignore iglob type type-not`等所有可以影响到匹配文件的参数，非常便捷。
4. search-string.zsh 初始时，如果未传入查询字符串，fzf面板为空，但会在后台执行一次该命令，即形式热缓存。
5. find-files.zsh search-string.zsh 支持大多数常用的选项参数。
6. find-files.zsh search-string.zsh 定义了一种规则来替代命令行中常用选项参数的传入，这使得它使用起来更加方便(精髓)。
7. find-files.zsh search-string.zsh 都能够正确解析带有特殊字符的文件。

# install

1. `git clone https://github.com/MaJinjie/fzf-fd-rg.git`
2. 把`fzf-fd fzf-rg tools-report` 加入到环境变量中

# usage

## 1 aliases

> 我的别名

```bash
ff "fzf-fd -H -d6 --split"
ss "fzf-rg -d6 --sort none:accessed --window full --split --args \"-j 2\""
```

# introduce

## find-files

### 1 help

```bash
# 1. 能够解析常用的参数
# 2. 接受目录或文件
# 3. 尽可能地解释用户传入的参数(很完美，几乎完全避免了和文件名或模式冲突)
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

    --sort -> rg --sortr (format default_val:other_val | val(default and other))
    --Sort -> rg --sort (same as up)
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

形式如：`ff <class>/<context>/[<class>/<context>]...[flags]`

**介绍**：

`class` 标志类，目前有ftex(E)。

f 单参数标志。内容可以为`higa<number>dfxlebcsp`，不解析flags,传入也没错
t 一个时间段。形式可以为`1d`、`,1d`、`1d,1h`，接受`r`标志，反转的意思，参数前后交换
e 拓展名集合。可以指定多个，接受`r`标志，一个是正向限定，一个是反向排除（通过`--exclude=*.exe`）来实现
E|x 和`--exclude=...`相同，可以指定多个

**注意**：

1. `/`可以替换为`[[:punct:]]`中的任何符号,例如`#,.:` 。
2. 多个class和context组合可以连续传递，而不需要使用空格分开。
3. 如果`context`中包含多个内容，请以`,`分割。在此情况下， 分隔符不可以为`,`，否则，它只会解析你的第一项。例如:`e,sh,zsh (error)`,正确的应该是 `e/sh,zsh/`
4. `h` 是切换`--hidden`参数，也就是说你之前传入`--hidden`，它就取消，否则就加上。`i` 同理
5. `0` 0深度不可能吧，所以它被解释为取消之前传入的`--max-depth`

```bash
# 下面是经过多次修改精简后的过程, 它包含了：
# 1. --changed-after --changed-before
# 2. --max-depth --type --glob --full-path
# 3. --hidden --no-ignore
# 4. --extension --exclude
__split() {
    ((DEBUG)) && set -x
    local item idx
    local match class context flags
    for item in "$@"; do
        { [[ -e $item ]] && ((o_q-- < 1)) } && {
            [[ -d $item ]] && Directories+=$item || Files+=$item
            continue
        }
        ((o_split)) && [[ $item =~ '^([[:alpha:]]([[:punct:]]).*\2)+[[:alpha:]]*$'  ]] && {
            flags=( ${(s//)item##*[[:punct:]]} )
            while [[ $item =~ '^[[:alpha:]]([[:punct:]]).*?\1' ]]; do
                class=$MATCH[1] context=$MATCH[3,-2] match=$MATCH
                case $class in
                    f)
                        while (($#context)); do
                            if [[ $context =~ ^h ]]; then
                                ((idx=$o_args[(I)--hidden])) && unset "o_args[idx]" || o_args+=--hidden
                            elif [[ $context =~ ^i ]]; then
                                ((idx=$o_args[(I)--no-ignore])) && unset "o_args[idx]" || o_args+=--no-ignore
                            elif [[ $context =~ ^g ]]; then
                                ((idx=$o_args[(I)--glob])) && unset "o_args[idx]" || o_args+=--glob
                            elif [[ $context =~ ^a ]]; then
                                ((idx=$o_args[(I)--full-path])) && unset "o_args[idx]" || o_args+=--full-path
                            elif [[ $context =~ ^[dfxlebcsp] ]]; then
                                o_types+=$MATCH
                            elif [[ $context =~ ^[[:digit:]]+ ]]; then
                                unset "o_args[${o_args[(i)--max-depth*]}]"
                                ((MATCH)) && o_args+="--max-depth=$MATCH"
                            else
                                report_error "class:$class" "$context" "内容错误"
                            fi
                            context=${context#$MATCH}
                        done
                        ;;
                    t)
                        local after before
                        ((flags[(I)r])) && {after=before; before=after;} || {after=after; before=before;}
                        [[ $context =~ ^[^,]* && -n $MATCH ]] && o_args+="--changed-$after=$MATCH"
                        context=${context#$MATCH}
                        [[ $context =~ [^,]*$ && -n $MATCH ]] && o_args+="--changed-$before=$MATCH"
                        ;;
                    e)
                        ((flags[(I)r])) && o_excludes+=( $(print -- \*.${^${(s/,/)context}}) ) || o_extensions+=( ${(s/,/)context} )
                        ;;
                    x)
                        o_excludes+=( $(print -- ${^${(s/,/)context}}) )                        ;;
                    *)
                        report_error "class:$class" "类别错误"
                        ;;
                esac
                item=${item#$match}
            done
            continue
        }
        [[ -z $Pattern ]] || report_error "$item error, pattern is exists"
        Pattern=$item
    done
    ((DEBUG)) && set +x
}

```

## search-string

> 交互式查找字符串

### 1 help

```bash
# 1. 能够解析常用的参数
# 2. 接受目录或文件
# 3. 能够在fzf过滤时，解释--iglob --hidden --no-ignore --max-depth --type --type-not
# 4. 尽可能解释用户传入的参数
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
        -m --max-count
        -1 --max-count=1
        -L --follow

        --sort -> rg --sortr (format default_val:other_val | val(default and other))
        --Sort -> rg --sort (same as up)
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

形式如：`ss <class>/<context>/[<class>/<context>]...[flags]`

**介绍**：

`class` 标志类，目前有ftex(E)。

f 单参数标志。内容可以为`hi<number>os`，其中，s是排序，o是文件匹配唯一。其中，s接受flags中的r，即反序
t --type。接受`r`标志，--type-not
e 拓展名集合。可以指定多个，接受`r`标志，一个是正向限定，一个是反向排除（通过`--exclude=*.exe`）来实现
g --iglob相同，可以指定多个，接受r标志

```bash
# 下面是经过多次修改精简后的过程, 它包含了：
# 1. --max-depth --sort --max-count=1
# 2. --hidden --no-ignore
# 3. --type --type-not
# 4. --iglob

__split() {
    local item idx opt
    local match class context flags
    for item in "$@"; do
        { [[ -e $item ]] && ((o_q-- < 1)) } && {
            [[ -d $item ]] && Directories+=$item || Files+=$item
            continue
        }
        ((o_split)) && [[ $item =~ '^([[:alpha:]]([[:punct:]]).*\2)+[[:alpha:]]*$'  ]] && {
            flags=( ${(s//)item##*[[:punct:]]} )
            while [[ $item =~ '^[[:alpha:]]([[:punct:]]).*?\1' ]]; do
                class=$MATCH[1] context=$MATCH[3,-2] match=$MATCH
                case $class in
                    f)
                        while (($#context)); do
                            if [[ $context =~ ^h ]]; then
                                ((idx=$o_dynamic[(I)--hidden])) && unset "o_dynamic[idx]" || o_dynamic+=--hidden
                            elif [[ $context =~ ^i ]]; then
                                ((idx=$o_dynamic[(I)--no-ignore])) && unset "o_dynamic[idx]" || o_dynamic+=--no-ignore
                            elif [[ $context =~ ^o ]]; then
                                 o_dynamic[$o_dynamic[(i)--max-count*]]=--max-count=1
                            elif [[ $context =~ ^s ]]; then
                                ((flags[(I)r])) && opt=--sort= || opt=--sortr=
                                ((idx=$o_dynamic[(I)$opt*])) && if [[ $o_dynamic[idx] == *$o_default_map[${opt%=}] ]]; then
                                    o_dynamic[idx]=$opt$o_normal_map[${opt%=}]
                                else
                                    o_dynamic[idx]=$opt$o_default_map[${opt%=}]
                                fi
                            elif [[ $context =~ ^[[:digit:]]+ ]]; then
                                unset "o_dynamic[${o_dynamic[(i)--max-depth*]}]"
                                ((MATCH)) && o_dynamic+="--max-depth=$MATCH"
                            else
                                report_error "class:$class" "$context" "内容错误"
                            fi
                            context=${context#$MATCH}
                        done
                        ;;
                    t)
                        ((flags[(I)r])) && opt=--type-not= || opt=--type=
                        o_types+=( $(print -- $opt${^${(s/,/)context}}) )
                        ;;
                    e)
                        ((flags[(I)r])) && opt="--iglob=!" || opt="--iglob="
                        o_iglobs+=( $(print -- \'$opt\*.${^${(@s/,/)context}}\') )
                        ;;
                    g)
                        ((flags[(I)r])) && opt="--iglob=!" || opt="--iglob="
                        o_iglobs+=( $(print -- \'$opt${^${(@s/,/)context}}\') )
                        ;;
                    *)
                        report_error "class:$class" "类别错误"
                        ;;
                esac
                item=${item#$match}

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
            o) dynamic[\$dynamic[(i)--max-count*]]=--max-count=1; item=${item#,##} ;;
            h) ((idx=\$dynamic[(I)--hidden])) && unset \\\"dynamic[idx]\\\" || dynamic+=--hidden; item= ;;
            i) ((idx=\$dynamic[(I)--no-ignore])) && unset \\\"dynamic[idx]\\\" || dynamic+=--no-ignore; item= ;;
            s) ((idx=\$dynamic[(I)--sortr[[:space:]=]*])) && { [[ \$dynamic[idx] == *$o_default_map[--sortr] ]] && dynamic[idx]=--sortr=$o_normal_map[--sortr] || dynamic[idx]=--sortr=$o_default_map[--sortr] }; item= ;;
            S) ((idx=\$dynamic[(I)--sort[[:space:]=]*])) && { [[ \$dynamic[idx] == *$o_default_map[--sort] ]] && dynamic[idx]=--sort=$o_normal_map[--sort] || dynamic[idx]=--sort=$o_default_map[--sort] }; item= ;;
            <->) unset \\\"dynamic[\$dynamic[(i)--max-depth*]]\\\"; ((\$item)) && dynamic+=--max-depth=\$item; item= ;;
            [[:lower:]]##,,*) types+=--type-not=\${item[(ws/,,/)1]}; item=\${item#*,,} ;;
            [[:lower:]]##,*) types+=--type=\${item[(ws/,/)1]}; item=\${item#*,} ;;
            *) iglobs+=\\\"'--iglob=\$item'\\\"; item= ;;
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
# 如果更改了fzf布局，需要改变line中的数字3
change_preview="
typeset lines=\$((FZF_LINES - 3))  match_count=\$FZF_MATCH_COUNT preview_lines=\${FZF_PREVIEW_LINES:-\${\$(<$file_preview_size):-0}}
typeset b1=10000 b2=1000 b3=100 per1=0 per2=30 per3=60 result
if ((match_count == 0 || match_count > b1)); then
    result=0
elif ((match_count > b2)); then
    result=\$(( ((b1 - match_count) * (per2 - per1) / (b1 - b2)  + per1) * lines / 100 ))
elif ((match_count > b3)); then
    result=\$(( ((b2 - match_count) * (per3 - per2) / (b2 - b3)  + per2) * lines / 100 ))
elif ((match_count > (100 - per3) * lines / 100)); then
    result=\$(( per3 * lines / 100 ))
else
    result=$\((lines - match_count))
fi
[[ -z \$FZF_PREVIEW_LINES ]] && print \$result > $file_preview_size
print \\\"change-preview-window(\$result)\\\"
"
```

由于在我目前版本，FZF_PREVIEW_LINES始终为空，所以我使用文件记录预览窗口的行数

### 5 inital-search

如果字符串为空，那么就在后台执行一遍命令。
目的：形成热缓存，加快未来的查找速度。

```bash
initial_search="
if [[ -z '\{q}' ]]; then
    if (($o_default_map[--no-warm])); then
        print \\\"ignore\\\"
    else
        print \\\"execute-silent:$cmd $o_dynamic $o_types -- \\\{q} ${Directories} ${Files} &> /dev/null &\\\"
    fi
else
    print \\\"reload:$cmd $o_dynamic $o_types -- \\\{q} ${Directories} ${Files}\\\"
fi
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

1. 接下来只对zsh版本做维护
2. 如果没有`lscolors`, 请使用对应的包管理工具下载或访问https://github.com/sharkdp/lscolors下载。
3. 如果不想要`report`颜色输出，就直接修改源脚本改为`print`, 注意脚本中的report路径

# 总结

编写zsh脚本的过程，也是学习zsh与bash不同的过程。
相较于bash，zsh对变量的解析更加精细，对变量可用的操作也更多。
最典型的也是最常用的字符串和数组之间的变换，zsh只需要简单使用变量标志即可，而bash却可能写一个循环。
zsh中，通配符拓展以及`zsh/pcre`引擎都很好。
只有用过的人，才能深有体会。
