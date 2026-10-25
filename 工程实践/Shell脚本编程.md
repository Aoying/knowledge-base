---
知识点: Shell脚本编程
层级: 基础
适用版本: Bash 4+
更新日期: 2026-10-25
关联:
  - Linux与Shell
标签:
  - 运维
  - 实践
---

# Shell 脚本编程

> P0：Shell 编程基础。基础命令、权限、SSH、systemd、cron 见 [工程实践/Linux与Shell.md](工程实践/Linux与Shell.md)，本篇只写脚本编程本身。

## 1. 写脚本的规矩

```bash
#!/bin/bash
# 第一行叫 shebang，指定解释器，必须是文件第一行第一列
chmod +x deploy.sh
./deploy.sh        # 执行要带路径；直接写 deploy.sh 会去 PATH 里找，找不到
bash deploy.sh     # 不加执行权限时这样跑也行
```

- 文件保存为 LF 换行（Windows 的 CRLF 到 Linux 会报 `bad interpreter`）。
- 调试先过语法关：`bash -n deploy.sh` 只检查语法不执行。

## 2. 变量

```bash
name="app"         # 赋值：等号两边不能有空格，否则会被当成命令
echo "$name"       # 引用变量加双引号，见第 9 节
readonly VER="1.0" # 只读变量，之后再赋值会报错
unset name         # 删除变量
export JAVA_HOME=/usr/lib/jvm/java-17   # export 后子进程才能继承
```

- 变量名：字母、数字、下划线，不能以数字开头。
- 命令行里 `VAR=val ./app.sh` 只对这一次执行生效，不污染当前 shell。

## 3. 字符串处理

`${}` 是字符串处理的主力，不用为了截取去调 awk：

```bash
f="/data/myapp/app-20261019.tar.gz"
echo "${#f}"            # 长度：31
echo "${f##*/}"         # 删最长前缀到最后一个 /：app-20261019.tar.gz
echo "${f#*/}"          # 删最短前缀到第一个 /：data/myapp/app-20261019.tar.gz
echo "${f%.*}"          # 删最短后缀（到最后一个点）：/data/myapp/app-20261019.tar
echo "${f%%.*}"         # 删最长后缀（到第一个点）：/data/myapp/app-20261019
echo "${f:6:5}"         # 从下标 6 起取 5 个字符：myapp
echo "${f/app/web}"     # 首次替换：/data/myweb/app-20261019.tar.gz
echo "${f//a/A}"        # 全部替换：/dAtA/myApp/App-20261019.tAr.gz
```

默认值与容错（脚本里最常用的三招）：

```bash
echo "${PORT:-8080}"    # PORT 未定义或为空时取 8080（:- 空也触发）
echo "${PORT-8080}"     # 只有未定义时取 8080（- 空不触发）
echo "${PORT:?PORT 未设置}"  # PORT 未定义或为空时直接报错退出
```

大小写转换（Bash 4+）：

```bash
s="helloWorld"
echo "${s^}"    # 首字母大写：HelloWorld
echo "${s^^}"   # 全大写：HELLOWORLD
echo "${s,}"    # 首字母小写（已是小写则不变）
echo "${s,,}"   # 全小写：helloworld
```

- 记忆口诀：`#` 从左删（键盘上 # 在左）、`%` 从右删；一个符号最短删，两个符号最长删。

## 4. 数组

```bash
hosts=(web01 web02 "web 03")   # 定义；带空格的元素必须加引号
echo "${hosts[0]}"             # 取下标：web01
echo "${hosts[@]}"             # 取全部元素
echo "${#hosts[@]}"            # 元素个数：3
echo "${!hosts[@]}"            # 全部下标：0 1 2
hosts+=(web04)                 # 追加
for h in "${hosts[@]}"; do echo "$h"; done   # 遍历必须加引号，否则 web 03 会被拆成两个
```

关联数组（键值对，Bash 4+，macOS 自带 bash 3.2 不支持）：

```bash
declare -A port=( [web]=8080 [api]=8081 )
echo "${port[web]}"     # 8080
echo "${!port[@]}"      # 全部键：web api（顺序不保证）
```

## 5. 位置参数与特殊变量

```bash
./deploy.sh prod v1.2
# 脚本内：
#   $0        脚本名本身：./deploy.sh
#   $1 $2     第 1、2 个参数：prod、v1.2
#   $#        参数个数：2
#   $@        全部参数（加引号 "$@" 时每个参数独立成词）
#   $*        全部参数（加引号 "$*" 时拼成一个词）
#   $?        上一个命令的退出码
#   $$        当前脚本进程 PID
#   $!        最近一个后台进程 PID
```

`"$@"` 与 `"$*"` 的区别是高频坑：

```bash
# 假设参数是 "web 01" web02（第一个参数本身带空格）
for a in "$@"; do echo "[$a]"; done   # 两行：[web 01] [web02]，正确
for a in "$*"; do echo "[$a]"; done   # 一行：[web 01 web02]，被拼成一个
for a in $@; do echo "[$a]"; done     # 不加引号会按空格拆词：三行，全错
```

移位与兜底：

```bash
env="$1"; shift          # 取出第 1 个参数，剩下的前移一位
version="${1:-latest}"   # 没传第 2 个参数时默认 latest
[ $# -lt 1 ] && { echo "用法: $0 <env> [version]"; exit 1; }
```

## 6. 函数与退出码

```bash
health_check() {
  local url="$1"              # local 只在函数内有效，不写 local 会污染全局
  curl -fsS -m 5 "$url" >/dev/null
  return $?                   # return 的值就是函数的退出码
}

if health_check "http://127.0.0.1:8080/health"; then
  echo "服务正常"
fi
```

返回值与退出码的区别（新手最容易混的一对）：

- 退出码：函数或命令跑完留下的状态，`return` 或最后一个命令决定，取值只能是 0–255（`return 300` 会被截成 44，即 300−256）。
- 返回值（数据）：函数想往外传数据，用 `echo` 输出、调用方用命令替换接住，而不是 `return`：

```bash
get_version() { echo "v1.2"; }   # 传数据靠输出
ver="$(get_version)"              # ver=v1.2
```

- `return` 后面不写值时，默认返回函数内最后一条命令的退出码。

## 7. 条件：if 与 case

```bash
if [ -f /data/myapp/app.jar ]; then
  echo "jar 存在"
elif [ -d /data/myapp ]; then
  echo "目录存在但 jar 不在"
else
  echo "都不存在"
fi
```

常用 test 判断：`-f` 文件存在、`-d` 目录存在、`-s` 文件非空、`-z` 字符串为空、`-n` 字符串非空、`-eq/-ne/-lt/-gt` 数字比较、`=`/`!=` 字符串比较。

`[[ ]]` 比 `[ ]` 好用，脚本里优先用它：

```bash
name="app-prod"
if [[ "$name" == app-* && -n "$name" ]]; then echo "命中"; fi   # 支持通配与 &&，不用转义
if [[ "$ver" =~ ^v[0-9]+\.[0-9]+$ ]]; then echo "版本号合法"; fi  # =~ 直接写正则
```

- `[ ]` 是命令：变量必须加引号（空变量会报 `unary operator expected`），`>` `<` 要转义，`&&` 不能直接写里面。
- `[[ ]]` 是关键字：变量不用加引号也能防空值，支持 `==` 通配和 `=~` 正则。

多分支用 case，比一串 if-elif 清晰：

```bash
case "$env" in
  prod)  port=8080 ;;
  test)  port=8081 ;;
  dev|local) port=8082 ;;   # 一个分支可写多个值，用 | 分隔
  *)     echo "未知环境: $env"; exit 1 ;;
esac
```

## 8. 循环

```bash
for h in web01 web02 web03; do echo "部署 $h"; done
for ((i = 0; i < 3; i++)); do echo "第 $i 次"; done     # C 风格循环
for f in /data/logs/*.log; do echo "处理 $f"; done      # 通配展开成文件列表
retry=0
while [ "$retry" -lt 3 ]; do
  curl -fsS http://127.0.0.1:8080/health && break
  retry=$((retry + 1)); sleep 2
done
```

按行读文件（`IFS=` 保住行首行尾空格，`-r` 不让反斜杠转义）：

```bash
while IFS= read -r line; do echo "行: $line"; done < list.txt
```

- 管道进 while 的坑：`cat list.txt | while read -r line; do ...; done` 里循环跑在子 shell，循环内改的变量循环外看不到；要保留变量改用重定向（上例）或进程替换（第 10 节）。

`break` 跳出循环，`continue` 跳过本次；`break 2` 可一次跳出两层循环。

## 9. 引号与命令替换

```bash
name="world"
echo "hello $name"     # 双引号：变量、命令替换、算术展开都生效
echo 'hello $name'     # 单引号：原样输出 hello $name，什么都不展开
echo hello\ $name      # 反斜杠转义单个字符
echo $name             # 无引号：会按空格拆词 + 通配展开，能加引号就加引号
```

命令替换：`$()` 优先于反引号（可嵌套、可读性好）：

```bash
today="$(date +%F)"              # 2026-10-25
count="$(grep -c ERROR app.log)" # 注意：grep 没匹配时退出码是 1，见第 11 节
nested="$(echo "内层 $(date +%Y)")"  # 嵌套只有 $() 能直观写出来
```

- 命令替换会删掉结果末尾的换行符；要保留多行结构时用数组或重定向到文件。
- `$(...)` 里的命令失败时，赋值语句本身的退出码仍是 0（除非开 pipefail 组合判断，见第 11 节）。

## 10. 管道与重定向（脚本视角）

```bash
./app.sh > out.log 2>&1      # stdout 与 stderr 都进文件（顺序不能反）
./app.sh 2>&1 > out.log      # 错了：stderr 去了终端，文件里只有 stdout
./app.sh > /dev/null 2>&1    # 全部丢弃
./app.sh >> out.log 2>&1     # 追加，不覆盖
cmd < input.txt              # 文件当标准输入
```

`2>&1` 的含义是“把 stderr 并到 stdout 当前去的地方”，所以必须写在 `>` 之后。

HereDoc（多行文本喂给命令）：

```bash
cat > /data/myapp/config.properties <<EOF
port=${PORT:-8080}
env=$env
EOF
# 不加引号的 EOF：变量会展开；写成 <<'EOF' 则原样输出，适合写模板
# <<-EOF 会删掉每行开头的 Tab（注意是 Tab，不是空格），适合缩进代码
```

HereString 与进程替换：

```bash
grep ERROR <<< "$log_text"              # <<< 把字符串当一行输入
diff <(sort a.txt) <(sort b.txt)        # <(...) 把命令输出伪装成文件
while read -r line; do echo "$line"; done < <(grep ERROR app.log)  # 不进子 shell，变量保留
```

## 11. trap、调试与错误处理

先写 cleanup 再写逻辑，临时文件与子进程才不会泄漏：

```bash
TMP="$(mktemp -d)"
cleanup() { rm -rf "$TMP"; }
trap cleanup EXIT      # 脚本退出时（无论成功失败）都清掉临时目录
trap 'echo "被中断"; exit 130' INT TERM   # Ctrl+C 与 kill 默认发这两个信号
```

- `trap '' INT` 是忽略信号（一般别用，脚本关不掉）；`trap - INT` 是恢复默认行为。
- EXIT trap 里不要再 exit，否则会覆盖脚本原本的退出码。

调试三板斧：

```bash
bash -n deploy.sh      # 只查语法，不执行
bash -x deploy.sh      # 执行时逐行打印（变量展开后的样子）
set -x                 # 在脚本内从这行起开追踪；set +x 关掉
```

错误处理用 `set -euo pipefail`，但要懂它的三处含义与坑：

```bash
set -e                 # 任一命令失败（退出码非 0）立即退出
set -u                 # 用到未定义变量立即报错退出
set -o pipefail        # 管道以最后一个非 0 的退出码为准（默认只看最后一个命令）
```

```bash
set -euo pipefail
grep ERROR app.log | wc -l   # 没匹配时 grep 退出码 1，pipefail 让整条管道失败，脚本直接退出
```

- `set -e` 不触发的场景：if/while 的条件里、`&&`/`||` 列表里（最后一个除外）、函数返回值被判断时。需要“失败也继续”的命令，显式写 `cmd || true`。
- `set -u` 下 `${VAR:-default}` 不报错（它本身就是判空写法），但裸用 `$VAR` 会直接退出。
- `set -e` + 命令替换的坑：`out="$(可能失败的命令)"` 赋值语句本身退出码是 0，失败会被吞掉；关键命令失败必须立即判断时，拆开写或用 `if ! out="$(cmd)"; then` 判断。

ERR trap 补最后一块拼图（失败时定位行号）：

```bash
trap 'echo "失败于行号 $LINENO，退出码 $?"' ERR
```

- ERR trap 同样不进 if 条件与 `||` 右侧；它只帮定位，不替代 `set -e`。

## 12. 综合示例：带清理与重试的部署检查

```bash
#!/bin/bash
set -euo pipefail

ENV="${1:-test}"
RETRY="${2:-3}"
TMP="$(mktemp -d)"
trap 'rm -rf "$TMP"' EXIT

log() { echo "[$(date +%F_%T)] $*"; }

check() {
  local url="$1" i=0
  while [ "$i" -lt "$RETRY" ]; do
    if curl -fsS -m 5 "$url" >/dev/null; then
      log "健康检查通过: $url"
      return 0
    fi
    i=$((i + 1)); sleep 2
  done
  log "健康检查失败: $url"
  return 1
}

case "$ENV" in
  prod) URL="http://127.0.0.1:8080/health" ;;
  test) URL="http://127.0.0.1:8081/health" ;;
  *)    log "未知环境: $ENV"; exit 1 ;;
esac

check "$URL"
log "完成"
```

## 13. 常见坑

- 赋值等号两边有空格：`a = 1` 会被当成执行命令 `a`。
- `"$@"` 不加引号：带空格的参数会被拆词，见第 5 节。
- `[ ]` 里变量不加引号：空变量报 `unary operator expected`，优先用 `[[ ]]`。
- 函数 `return` 传数据：只能传 0–255 的状态，数据靠 `echo` + 命令替换。
- `cmd > file 2>&1` 顺序写反：stderr 去了终端，文件里只有 stdout。
- `set -e` 当万能药：条件判断与管道场景下它不触发，关键失败要显式判断。
- 通配展开无匹配时原样保留：`for f in *.log` 在没有 log 时会循环一次字面量 `*.log`，先 `shopt -s nullglob` 可避免。

## 面试与自测

1. `"$@"`、`"$*"`、`$@` 三者有什么区别？遍历带空格的参数时该用哪个？
2. 函数想把算出来的字符串传给调用方，为什么不能用 `return`？该怎么做？
3. `cmd > file 2>&1` 和 `cmd 2>&1 > file` 的结果有什么不同？为什么？
4. `set -euo pipefail` 里三个选项各起什么作用？`set -e` 在哪些场景下不触发？
5. 脚本里建了临时目录，怎么保证脚本中途失败或被 Ctrl+C 中断时也能清掉？

### 参考答案

1. `"$@"` 把每个参数展开成独立的词，带空格的参数不会被拆散；`"$*"` 把全部参数拼成一个词（用 IFS 首字符连接）；`$@` 不加引号会按空格拆词，带空格的参数会被拆错。遍历带空格的参数必须用 `"$@"`。
2. `return` 只能传 0–255 的退出状态，传数据会丢信息（还会对 256 取模）。正确做法是函数内 `echo` 输出数据，调用方用命令替换接住，如 `ver="$(get_version)"`。
3. 前者 stdout 与 stderr 都进文件；后者 stderr 去了终端，文件里只有 stdout。因为 `2>&1` 的含义是“把 stderr 并到 stdout 当前去的地方”，写在 `>` 之前时 stdout 还指向终端，写在之后才指向文件。
4. `-e` 任一命令失败立即退出；`-u` 用到未定义变量立即报错；`pipefail` 让管道以最后一个非 0 的退出码为准。`set -e` 不触发的场景：if/while 条件里、`&&`/`||` 列表里（最后一个除外）、返回值正被判断时；需要失败也继续的命令写 `cmd || true`。
5. 用 trap：先定义 cleanup 函数删临时目录，再 `trap cleanup EXIT`（无论成功失败退出时都执行），并 `trap '...' INT TERM` 处理中断信号；注意 EXIT trap 里不要再 exit，以免覆盖原本的退出码。
