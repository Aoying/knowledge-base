---
知识点: 生产级Shell脚本
层级: 进阶
适用版本: Bash 4+
更新日期: 2026-10-25
关联:
  - Linux与Shell
  - Shell脚本编程
标签:
  - 运维
  - 实践
---

# 生产级 Shell 脚本

> 基础命令、权限、管道、systemd、SSH 见 [工程实践/Shell/Linux与Shell.md](工程实践/Shell/Linux与Shell.md)，本篇不重复。本篇只讲一件事：脚本从“能跑”到“敢放进生产定时任务”还差的写法——严格模式、参数与配置、日志、重试、清理、锁、幂等、安全删除、静态检查。

## 1. 严格模式与退出码

```bash
#!/usr/bin/env bash
set -euo pipefail
```

- `set -e`：任一命令失败立即退出；`set -u`：未定义变量即报错；`set -o pipefail`：管道中任一段失败，整个管道算失败。
- 例外要显式处理：`if ! cmd; then ... fi`、`cmd || true`、条件判断里的命令不受 `set -e` 影响。
- 退出码约定：0 成功，1 通用失败，2 用法错误，业务自定义从 10 起；`exit` 时带上码，不要裸 `exit`。

## 2. 参数解析与配置管理

短参数用 `getopts`，长参数手写 `case` 循环先转成短参数，再统一进 `getopts`：

```bash
usage() { echo "用法: $0 [-c 配置文件] [-e 环境] [-f]"; }

args=()
while [[ $# -gt 0 ]]; do
  case "$1" in
    --config)  args+=(-c "$2"); shift 2 ;;
    --config=*) args+=(-c "${1#*=}"); shift ;;
    --env)     args+=(-e "$2"); shift 2 ;;
    --force)   args+=(-f); shift ;;
    --help)    usage; exit 0 ;;
    --)        shift; args+=(-- "$@"); break ;;
    *)         args+=("$1"); shift ;;
  esac
done
set -- ${args[@]+"${args[@]}"}

CONFIG="" ENV="prod" FORCE=0
while getopts ":c:e:fh" opt; do
  case "$opt" in
    c) CONFIG="$OPTARG" ;;
    e) ENV="$OPTARG" ;;
    f) FORCE=1 ;;
    h) usage; exit 0 ;;
    *) usage; exit 2 ;;
  esac
done
```

配置加载优先级：命令行参数 > 环境变量 > 配置文件 > 默认值。实现顺序反过来写——先默认值，再 `source` 配置文件，再用环境变量覆盖，最后命令行参数覆盖：

```bash
ENV="prod"                       # 默认值
[[ -n "$CONFIG" && -f "$CONFIG" ]] && set -a && source "$CONFIG" && set +a
ENV="${APP_ENV:-$ENV}"           # 环境变量覆盖配置文件
```

- 配置文件只加载受信任的路径；`set -a` 让文件里的变量自动 export，加载完记得 `set +a`。
- 敏感值（密码、Token）只走环境变量或密钥管理，不写进配置文件、不进脚本正文。

## 3. 日志：分级、时间戳、双写

```bash
LOG_FILE="/var/log/myapp/deploy.log"

log() {
  local level="$1"; shift
  printf '%s [%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$level" "$*" | tee -a "$LOG_FILE"
}
log_info()  { log "INFO"  "$@"; }
log_warn()  { log "WARN"  "$@" >&2; }
log_error() { log "ERROR" "$@" >&2; }
```

- INFO 走 stdout，WARN/ERROR 走 stderr，文件里三级全留；`tee -a` 追加不覆盖。
- 日志里不打敏感值；需要脱敏的字段先替换再输出。
- 归档与轮转交给 logrotate 或定时任务（见第 10 节骨架外的运维动作），脚本自己不负责删日志。

## 4. 异常处理与重试：指数退避 + 上限

```bash
# retry <最大次数> <命令...>
retry() {
  local max_attempts="$1"; shift
  local delay=1 elapsed=0 max_elapsed=300 attempt=1 rc=0
  while true; do
    # 注意：不要写成 if "$@"; then return 0; fi 再取 $?——无 else 的 if 失败时返回 0，取不到命令的退出码
    "$@" && return 0
    rc=$?
    if (( attempt >= max_attempts )) || (( elapsed >= max_elapsed )); then
      log_error "重试耗尽（${attempt} 次，耗时 ${elapsed}s），最后退出码 ${rc}"
      return "$rc"
    fi
    log_warn "第 ${attempt} 次失败（退出码 ${rc}），${delay}s 后重试"
    sleep "$delay"
    elapsed=$((elapsed + delay))
    delay=$((delay * 2))
    (( delay > 30 )) && delay=30
    attempt=$((attempt + 1))
  done
}

retry 5 curl -fsS --max-time 30 "https://example.com/health"
```

- 两个上限缺一不可：次数上限防无限重试，总超时上限防单次雪崩时拖垮定时任务窗口。
- `if "$@"` 包住命令，`set -e` 下失败也不会直接退出脚本。
- 只对瞬时故障重试（网络抖动、服务重启中）；参数错误、配置错误、权限错误重试无用，直接失败告警。

## 5. 临时文件与资源清理：mktemp + trap

```bash
WORKDIR="$(mktemp -d)"
trap 'rm -rf "$WORKDIR"' EXIT
trap 'log_error "脚本在行 ${LINENO} 异常退出"' ERR
```

- 临时目录、临时文件一律 `mktemp` 生成，不自己拼 `/tmp/xxx.$$`（可预测路径有劫持风险）。
- 清理只挂 `EXIT`，正常退出和异常退出都会跑；`ERR` 只做记录，不做清理，避免重复清理。
- `trap` 里引用的变量要加引号，且清理命令本身要对空值安全（见第 8 节）。

## 6. 文件锁与并发执行控制：flock

```bash
LOCK_FILE="/var/lock/myapp-deploy.lock"
exec 200>"$LOCK_FILE"
if ! flock -w 60 200; then
  log_error "60s 内未拿到锁，可能有旧实例在跑，退出"
  exit 1
fi
```

- `flock` 保证同一时刻只有一个实例；锁随文件描述符持有，脚本退出自动释放，不用手动解锁。
- `-w 60` 是拿锁超时，不是任务超时；拿不到锁就退出，不要静默排队（定时任务下排队等于堆积）。
- 锁文件放 `/var/lock`，不要放进临时目录（临时目录被清理后锁就没了）。

## 7. 幂等性：重复执行结果一致

- 目录创建用 `mkdir -p`，软链用 `ln -sfn`，不要 `mkdir` 裸奔、不要先 `rm` 再 `ln`。
- 文件写入用“写临时文件 + `mv` 原子替换”，不要直接重定向覆盖正在被读的文件。
- 部署类脚本先判断目标状态：已是目标版本直接跳过并返回成功，不要重复发布。
- 幂等不是“失败了重跑能好”，而是“成功了重跑也没副作用”——先写这条，再写重试。

## 8. 安全删除与权限最小化

```bash
# 安全删除：空值拒绝、根目录拒绝、-- 防 - 开头文件名、默认进备份目录
safe_rm() {
  local target="${1:?target required}"
  if [[ -z "$target" || "$target" == "/" ]]; then
    log_error "拒绝删除：目标为空或根目录"
    return 1
  fi
  local trash="/var/backups/trash/$(date +%Y%m%d)"
  mkdir -p "$trash"
  mv -- "$target" "$trash/$(basename "$target").$(date +%s)"
}
```

- 变量为空保护：`${TARGET:?变量为空}`，空值时直接报错退出，不走到 `rm`。
- `rm` 必须带 `--`：`rm -rf -- "$dir"`，防止 `$dir` 以 `-` 开头被当成选项。
- 能 mv 进备份目录就不直接 `rm -rf`；真要删，先 `ls` 同一路径确认一遍（[工程实践/Shell/Linux与Shell.md](工程实践/Shell/Linux与Shell.md) 的补强节有同一条血泪规则）。
- 权限最小化：脚本用专用账号跑，只对需要的命令配 sudo（visudo 里写具体命令，不写 `ALL`）；不要 `chmod 777` 兜底，不要整脚本 `sudo bash`。

## 9. ShellCheck 静态检查与自动化测试

```bash
# 安装（二选一）
sudo apt install shellcheck   # Debian/Ubuntu
brew install shellcheck       # macOS

shellcheck -S warning deploy.sh
```

- SC2086：变量未加引号——`rm -rf $dir` 改成 `rm -rf "$dir"`。
- SC2046：`for f in $(ls)` 这类未引用的命令替换——改成 glob 或 `while read`。
- SC2162：`read` 没加 `-r`——`read -r line`，反斜杠才不会被吃掉。
- 把 `shellcheck` 放进提交检查或 CI 门禁，警告清零再合入；不要用 `# shellcheck disable=` 批量屏蔽。

bats 入门一例（bats-core，测试上面第 4 节的 `retry`）：

```bats
#!/usr/bin/env bats

setup() {
  source ./lib.sh   # retry 函数所在文件
}

@test "retry 成功时只跑一次" {
  run retry 3 true
  [ "$status" -eq 0 ]
}

@test "retry 耗尽后返回原退出码" {
  run retry 2 false
  [ "$status" -eq 1 ]
}
```

```bash
bats test_retry.bats
```

## 10. 生产级脚本骨架：把上面串起来

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_NAME="$(basename "$0")"
LOG_FILE="/var/log/myapp/${SCRIPT_NAME%.sh}.log"
LOCK_FILE="/var/lock/${SCRIPT_NAME%.sh}.lock"

log() {
  local level="$1"; shift
  printf '%s [%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$level" "$*" | tee -a "$LOG_FILE"
}
log_info()  { log "INFO"  "$@"; }
log_warn()  { log "WARN"  "$@" >&2; }
log_error() { log "ERROR" "$@" >&2; }

usage() { echo "用法: $0 [-c 配置文件] [-e 环境]"; }

main() {
  # 锁：单实例
  exec 200>"$LOCK_FILE"
  flock -w 60 200 || { log_error "未拿到锁，退出"; exit 1; }

  # 临时目录：EXIT 时清理（必须用全局变量，trap 在 main 返回后执行，看不到 local）
  WORKDIR="$(mktemp -d)"
  trap 'rm -rf "$WORKDIR"' EXIT

  # 参数与配置（完整版解析见第 2 节，这里是简化版）
  local config="" env="prod"
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --config) config="$2"; shift 2 ;;
      --env)    env="$2";    shift 2 ;;
      --help)   usage; exit 0 ;;
      *)        usage; exit 2 ;;
    esac
  done
  [[ -n "$config" && -f "$config" ]] && set -a && source "$config" && set +a
  env="${APP_ENV:-$env}"

  # 业务步骤：幂等 + 重试
  log_info "开始执行，环境=${env}"
  retry 5 curl -fsS --max-time 30 "https://example.com/health"
  log_info "执行完成"
}

# retry 函数与第 4 节相同
retry() {
  local max_attempts="$1"; shift
  local delay=1 elapsed=0 max_elapsed=300 attempt=1 rc=0
  while true; do
    # 注意：不要写成 if "$@"; then return 0; fi 再取 $?——无 else 的 if 失败时返回 0，取不到命令的退出码
    "$@" && return 0
    rc=$?
    if (( attempt >= max_attempts )) || (( elapsed >= max_elapsed )); then
      log_error "重试耗尽（${attempt} 次，耗时 ${elapsed}s），最后退出码 ${rc}"
      return "$rc"
    fi
    log_warn "第 ${attempt} 次失败（退出码 ${rc}），${delay}s 后重试"
    sleep "$delay"
    elapsed=$((elapsed + delay))
    delay=$((delay * 2))
    (( delay > 30 )) && delay=30
    attempt=$((attempt + 1))
  done
}

main "$@"
```

- 骨架使用方式：把业务步骤换成自己的，把 `retry` 函数（第 4 节）贴到 `main` 之前；`flock`、`trap`、`log` 三件套不要删。
- 部署、重启、健康检查、日志归档、巡检、备份这些实战脚本，都是这个骨架换业务步骤，不用每个重写一套。

## 常见坑

- `set -e` 下 `cmd1 && cmd2 || fallback` 的 `fallback` 会在 `cmd1` 成功但 `cmd2` 失败时也跑——复杂逻辑拆成 `if` 写。
- 管道末尾用 `while read` 会跑在子 shell 里，循环里改的变量外面看不到——用 `while read -r line < <(cmd)` 进程替换。
- `trap` 清理里用 `rm -rf "$WORKDIR"`，`WORKDIR` 一定要在 `trap` 挂上之前赋值，且赋值失败时 `set -e` 会先退出导致没挂上清理——`mktemp` 失败要单独判。
- 锁文件描述符不要复用：`exec 200>` 只开一次，脚本里别再拿 200 做别的。
- 取退出码的坑：`if cmd; then ...; fi` 之后 `$?` 是 if 语句本身的状态（无 else 时恒为 0），不是 cmd 的退出码；要取退出码用 `cmd && return 0; rc=$?`（`&&` 链中非末尾命令失败不会触发 `set -e`）。

## 面试与自测

1. `set -euo pipefail` 三个选项各防什么？`set -e` 在什么情况下不会让脚本退出？
2. 配置优先级“命令行 > 环境变量 > 配置文件 > 默认值”在代码里按什么顺序实现？为什么是反过来写？
3. 重试函数为什么必须同时有次数上限和总超时上限？只给次数上限会出什么问题？
4. `flock` 的锁是靠什么持有的？脚本异常退出后锁会怎样？
5. `rm -rf $dir` 在 `$dir` 为空时会发生什么？生产脚本里至少要加哪三道防护？

### 参考答案

1. `-e` 任一命令失败即退出，`-u` 未定义变量即报错，`-o pipefail` 管道任一段失败即算失败。`set -e` 不生效的情况：条件判断（`if`/`while` 的测试命令）、`&&`/`||` 链中除最后一段外、函数返回值被测试时；所以重试函数里用 `if "$@"` 包住命令。
2. 按默认值 → 配置文件 → 环境变量 → 命令行参数的顺序依次覆盖写，后写的覆盖先写的，最终效果就是命令行优先级最高。反过来写是因为每一层都只覆盖已有的值，不用写一堆“如果没设置才赋值”的判断。
3. 次数上限防无限重试，总超时上限防单次重试链拖垮定时任务窗口；只给次数上限时，指数退避的等待时间会越拖越长（如 1+2+4+8+16+30…），一次任务可能占住几十分钟，拖垮后续调度。
4. 靠打开的文件描述符持有（`exec 200>锁文件` 后 `flock 200`）；脚本退出时描述符关闭，锁自动释放，所以异常退出也不会死锁，但拿不到锁时要主动退出，不要静默排队。
5. `$dir` 为空时 `rm -rf $dir` 的词拆分结果可能让 `rm -rf` 作用到当前目录或根目录（取决于展开结果），是删库级事故。三道防护：变量为空保护 `${dir:?}`、删除命令加 `--`、默认 mv 进备份目录而不是直接 `rm -rf`，真删之前先 `ls` 确认。
