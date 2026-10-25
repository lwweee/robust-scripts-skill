---
name: robust-scripts
description: 编写健壮、可断点续跑、可迁移的脚本与长时间后台任务。只要你要创建、修改或审查会运行较久、无人值守、处理大批量数据、有外部副作用（写文件/调 API/发消息/改数据库/删除）、或需要交给别人或换机器运行的脚本，就必须先读本文件，即使用户没有明确提到"健壮""断点""后台"。涉及关键词：脚本、批处理、爬取、数据处理、训练、迁移、定时任务、后台运行、nohup、systemd、tmux、检查点、checkpoint、日志、重试、断电重启、换机器运行。用户要求展示、列出、查看已有脚本或脚本运行状态时，也要使用本文件（从 what_script.csv 读取）。
---

# 编写健壮、可续跑、可迁移的脚本

## 心智模型：先接受三个事实

写脚本之前，先承认下面三件事一定会发生。后面所有条目都是它们的推论，理解了这三条，遇到本文没写到的情况也能自己推导出正确做法。

1. **脚本一定会被中断**：关机、重启、OOM、SIGKILL、断网、磁盘写满、会话断开、API 限流。能否从断点继续，决定了一次中断是"重跑一下"还是"丢掉三天的工作"。
2. **脚本一定会换环境**：换机器、换操作系统、换目录、换 Python 版本、换用户、放进容器、交给同事。凡是写死了"此刻这台机器"的东西，迟早出问题。
3. **未来的你读不到现在的你的脑子**：agent 的上下文会丢失，人类接手者也不在场。你知道的启动方式、进度查看方式、已知坑，如果没写进文件，就等于不存在。

## 按规模取用，不要一刀切

全套措施对一次性小脚本是过度设计，对长任务则是底线。动手前先判断级别：

| 级别 | 特征 | 最低要求 |
|---|---|---|
| L0 | 秒级、只读、无副作用、一次性 | 超时、清晰报错、不写死路径 |
| L1 | 分钟级，或会写文件 | L0 + 日志、原子写、幂等、dry-run |
| L2 | 长时间 / 无人值守 / 有外部副作用 / 可能换机器 | 本文全部章节 |

拿不准时按 L2 做。多花十分钟加防护，远比中断后重做便宜。

## 工作流

1. **动手前**：先查 `what_script.csv`（见 G 章的 H 小节）里有没有同类脚本，再检查是否已有同名任务在运行（锁文件、心跳、进程列表）、是否已有 `STATUS.md` 和检查点。有的话先接管它，而不是另起炉灶。
2. **设计时**：先想清楚"处理单元"是什么（一个文件、一条记录、一个 batch），检查点以它为粒度；再想清楚它怎么做到幂等。
3. **编写时**：按下面的检查清单逐项落实，从"参考实现"起步。
4. **试跑**：先 `--dry-run`，再 `--limit 10` 小规模跑，确认输出正确后再全量。
5. **验证中断恢复**：真的用 `kill -9` 杀一次，再启动，确认能续跑且结果无重复无遗漏（见"验收"）。
6. **交付**：写好 `STATUS.md`，在 `what_script.csv` 登记或更新这个脚本，用脱离会话的方式启动，观察一到两个检查点周期确认正常，再告诉用户。

---

## 检查清单

### A. 不丢数据

- **原子写入**：检查点、结果文件先写到同目录的临时文件，`flush` + `fsync` 后用 `os.replace` 改名。原因：崩溃时直接写目标文件会留下半截文件，而下次启动会把它当成有效数据。
- **先落输出，再更新检查点**：顺序反了，崩溃后就会"检查点说做完了，但输出不存在"。宁可多做一次，也不要漏做一次。
- **幂等**：重跑同一个处理单元不产生重复副作用（重复发送、重复扣费、重复插入、重复追加）。常用手段：以确定性 ID 做去重键、`INSERT ... ON CONFLICT`、输出文件名由输入确定而不是随机/时间戳。
- **检查点带 schema 版本号**：代码改动后能识别旧格式。版本不匹配时，若重来会有副作用，就停下报错，不要静默从头开始。
- **检查点损坏要能处理**：解析失败时保留坏文件（改名 `.bad`），再决定回退或报错，不要直接覆盖。
- **不能只靠信号处理**：SIGKILL、断电、内核 OOM 都抓不到信号。信号处理只是"尽量体面地退出"，真正的保障是定期保存检查点（按时间间隔或处理数量触发，例如每 30 秒或每 100 项）。
- **输入保持不变或可检测变化**：长任务跑到一半输入被改了，续跑结果会悄悄出错。必要时在检查点里记录输入的指纹（行数、大小、修改时间或哈希）。

### B. 不重复、不卡死

- **单实例锁**：用文件锁（POSIX 的 `flock`，Windows 的 `msvcrt.locking`，或跨平台的 `portalocker`）。不要只靠 PID 文件，PID 会被复用，文件会残留。
- **所有外部调用都设超时**：网络请求、子进程、数据库连接、锁等待。很多库默认无超时，一次卡住就让整个任务无限挂起。
- **心跳文件**：定期原子写入时间戳和进度，外部据此判断"在跑"还是"卡死"。
- **重试**：指数退避 + 随机抖动 + 次数上限；区分可重试错误（网络抖动、5xx、429）与不可重试错误（参数错误、权限错误、404），后者立即失败并记录，别重试。
- **自动拉起要有限制**：systemd 用 `Restart=on-failure` 配合 `RestartSec`、`StartLimitBurst`，否则会陷入崩溃循环刷爆日志和 API 配额。
- **优雅退出**：收到 SIGTERM/SIGINT/SIGHUP（Windows 为 SIGBREAK）后，设置停止标志，在处理单元之间检查，保存检查点后退出。不要在信号处理函数里做重活。
- **退出码有语义**：0 = 全部完成；非 0 = 失败；约定一个专门的码（如 75）表示"被中断、未完成、可续跑"。supervisor 和通知脚本靠它区分情况。

### C. 不撑爆机器

- **磁盘空间**：启动前和运行中检查剩余空间，不只是日志——输出、临时文件、检查点、缓存都会写满磁盘。低于阈值时停下并报警，而不是写到一半失败。
- **限制并发、内存和速率**：并发数可配置，对 API 做限速，大文件流式读写，不要一次性读入内存。
- **清理**：退出时清理子进程和临时文件；用进程组（POSIX 下 `start_new_session` / `killpg`）避免留下孤儿进程；启动时清理上次崩溃遗留的 `.tmp*` 文件。

### D. 日志

- **所有脚本都要落日志文件**，不要只打印到终端；stdout 和 stderr 都要进日志。后台运行时，终端输出等于没有。
- **日志上限与淘汰**：如果用户要求"日志达到 100MB 时删除最旧的 50%"，按其字面意图实现。在单个文件里原地删除前半部分需要重写整个文件，中途崩溃会丢全部日志，所以用**分段文件**实现等价效果：每段约 10MB，总量到 100MB 时删除最旧的若干段，直到总量 ≤ 50MB，永不删除当前正在写的段。参考实现里有现成的 handler。
- **日志内容**：启动时记录命令行参数、脚本版本（git hash 或文件哈希）、Python 与关键依赖版本、PID、主机名、工作目录、配置摘要；运行中记录进度（`x/N`、速率、预计剩余时间）；异常记录完整堆栈（用 `logging.exception`，并设置 `sys.excepthook` 兜住未捕获异常）。
- **密钥脱敏**：令牌、密码、API key、带凭据的 URL 不进日志、不进检查点、不进异常信息。
- **时间戳用 UTC**，ISO 8601 格式，避免换时区或夏令时后无法对照。
- **日志要克制**：高频循环里不要每项都打 INFO，否则有效信息被淹没，也会加快淘汰。

### E. 安全

- **破坏性操作先 dry-run**：删除、覆盖、批量修改、发送消息类操作必须提供 `--dry-run`，默认先列出将要做什么。
- **删除前先备份或移入回收站**；删除路径来自变量时，先断言变量非空、路径在预期根目录之内（防止 `rm -rf "$DIR/"` 在 `DIR` 为空时变成 `rm -rf /`）。
- **先小规模试跑，再全量**：提供 `--limit N`。
- **最小权限**：不需要 root 就不要用 root；不要为了方便把文件权限设成 777。

### F. 可迁移性（重点）

健壮性不只是"别崩"，还包括"换个地方还能跑"。判断标准很简单：**把整个工作目录拷到另一台干净的机器，装好依赖，执行同一条命令，任务应当能续跑或重新跑通。** 为此：

**F1. 路径与目录**
- 不写死绝对路径、用户名、盘符、主机名、IP、端口。以"根目录"为基准：默认取 `Path(__file__).resolve().parent`，可被环境变量（如 `JOB_HOME`）或命令行参数覆盖。
- 不依赖当前工作目录（cwd）；从任何目录启动结果都应一致。
- 用 `pathlib` 拼路径，不手拼 `/` 或 `\`。临时文件用 `tempfile`，不写死 `/tmp`。
- 检查点和状态文件里存**相对于根目录的相对路径**，不存绝对路径。否则搬家后检查点全部失效。

**F2. 操作系统差异**
- 已知差异点要主动处理：文件锁（`fcntl` 在 Windows 不存在）、信号（`SIGHUP` 在 Windows 不存在，`SIGBREAK` 只在 Windows）、路径分隔符与大小写敏感性、文件名非法字符、Windows 下无法删除/重命名被占用的文件、换行符。
- `open()` 始终显式指定 `encoding="utf-8"`；二进制数据用 `"rb"/"wb"`；不依赖系统 locale 与默认编码。
- 能用标准库完成的，不要 shell out 到 `grep/sed/awk/curl/jq`。必须调用外部程序时：用 `shutil.which` 先检测，缺失时给出清晰的安装提示；用参数列表调用 `subprocess.run([...])`，不用 `shell=True`（既有注入风险，也有 shell 方言差异）。
- 不要假设有 root、sudo、systemd、docker、cron。启动方式要先探测环境再选择，并准备降级：systemd → tmux/screen/`nohup setsid` → Windows 任务计划程序/NSSM。在 `STATUS.md` 里写明用了哪种。

**F3. 依赖与解释器**
- 锁定依赖版本（`requirements.txt` 配精确版本、`pyproject.toml` + lock 文件、`package-lock.json` 等），并记录 Python/Node 版本要求。
- 在脚本启动时检查解释器最低版本和关键依赖，不满足就立即退出并说明怎么修复，而不是跑到一半才抛 `ImportError`。
- 提供一键环境搭建方式（`setup.sh`、`Makefile` target 或 README 中的三行命令），推荐在 venv 或容器中运行，避免污染系统环境、避免和其他项目的依赖冲突。
- 复杂依赖（CUDA、系统库、特定编译器）写进 `Dockerfile` 或明确列在 README 的"环境要求"里。

**F4. 配置与密钥**
- 代码与配置分离。优先级：命令行参数 > 环境变量 > 配置文件 > 默认值。换环境只改配置，不改代码。
- 密钥只从环境变量或独立的密钥文件读取，不硬编码，不提交进版本库；提供 `.env.example` / `config.example.*` 列出需要哪些变量（只写变量名和说明，不写真实值）。
- 外部地址（API 域名、数据源 URL、镜像源）集中在配置里，便于替换。

**F5. 硬件与资源自适应**
- 并发数、batch size、内存上限不要写死：默认基于 `os.cpu_count()` 和可用内存探测，并允许参数覆盖。
- GPU 可选：探测不到时回退到 CPU 并在日志中明确说明，不要直接崩溃。

**F6. 状态与数据可搬运**
- 检查点和中间结果使用稳定、开放的格式（JSON、CSV、Parquet、SQLite）。避免 `pickle` 存检查点——它跨 Python 版本、跨平台、跨代码重构都容易失效，还有反序列化安全风险。
- 检查点中不放机器相关字段（绝对路径、PID、主机名）；如果为了诊断需要记录，就明确标注，且恢复时忽略它们。
- 时间戳统一 UTC；数值格式、排序、哈希不依赖 locale。

**F7. 网络环境**
- 尊重 `HTTP_PROXY / HTTPS_PROXY / NO_PROXY` 等环境变量；不假设能访问某个特定站点或镜像。
- 数据下载支持"已存在则跳过"和校验和验证，便于离线搬运数据后续跑。

**F8. 可复现性**
- 固定随机种子并记录；记录关键库版本、数据来源（URL、版本、日期）。
- 并行、GPU、浮点累加可能导致跨平台结果有微小差异，如存在这类情况，在文档里注明，而不是假装完全可复现。

**F9. 自检命令**
- 提供 `--doctor`（或 `--selfcheck`）：检查 Python/依赖版本、必要外部程序、环境变量、目录写权限、磁盘空间、网络连通性，并打印通过/失败和修复建议。换环境后第一件事就是跑它。

**F10. 声明假设**
- 在 README 或 `STATUS.md` 中写明：已在哪些平台测试过、哪些平台未测、已知不支持项。没测过就如实写"未测试"，不要默认声称跨平台。

### G. 交付与运维（针对 agent 的特别项）

- **写 `STATUS.md`**，作为留给"未来的自己"的交接说明（模板见下）。每次状态变化（启动、停止、迁移、改配置）都更新它。
- **脱离会话运行**：用 systemd、tmux、screen 或 `nohup setsid` 启动，不要用裸的 `cmd &`。会话断开时 SIGHUP 会杀死后台子进程。
- **启动后先观察**：至少等一个检查点周期，确认日志在增长、心跳在更新、进度在前进，再离开。
- **结束或失败时主动通知**：webhook、邮件或写入约定位置，不要让用户靠"过一会儿去看看"来发现任务挂了。
- **完成前先校验**：行数、校验和、抽样检查、与输入的对账。"进程退出码为 0"不等于"结果正确"。
- **如实汇报**：没验证过的不说"已完成"；验证过的写明验证了什么；没做的、没测的、有风险的直接说。
- **用户的明确要求优先**：用户指定了日志上限、目录、启动方式等，按用户的来，本文只是默认做法。

### H. 脚本登记：what_script.csv

每个你创建、接管或修改的脚本（L1 及以上）都要登记到一份专门的 `what_script.csv`。目的：换了新机器或来了新使用者（人或 agent），一眼能看清存在哪些脚本、在哪、怎么启动、依赖什么。没有登记的脚本，对未来的接手者等于不存在。

**位置**：脚本工作区的根目录（所有脚本共同的上级目录）。首次使用时选定位置，写入长期记忆或配置，之后不要换。文件不存在就创建，表头固定。

**只存静态信息，不存运行状态**。"是否在跑"会过期，写进去只会误导。要看状态时现场查（锁文件、心跳时间戳、进程）。

| 列 | 内容 |
|---|---|
| `id` | 唯一且永不改变的标识（如 `news-crawler`） |
| `name` / `purpose` | 名称 / 一句话用途 |
| `path` | 脚本目录或入口，**相对于工作区根目录**，不写绝对路径 |
| `entry_command` | 完整启动命令（相对路径写法） |
| `run_mode` | `systemd` / `tmux` / `cron` / `manual` 等 |
| `runtime` | 解释器及版本要求，如 `python>=3.10` |
| `deps_file` | 依赖声明文件路径（lock 文件、requirements、Dockerfile） |
| `env_var_names` | 需要的环境变量**名字**，绝不写值 |
| `external_programs` | 依赖的外部程序（`ffmpeg`、`jq` 等） |
| `ports` / `resources` | 占用端口 / 资源需求（GPU、内存、磁盘） |
| `status_file` / `heartbeat_path` | `STATUS.md` 与心跳文件路径 |
| `state_paths` | 检查点、输出目录 |
| `data_sources` | 外部输入的来源与重新获取方式（尤其是不随脚本打包的大文件） |
| `lifecycle` | `active` 或 `retired` |
| `created_utc` / `updated_utc` | UTC 时间 |
| `notes` | 其他必要说明 |

多值字段用分号分隔。

**写入规则**：
- 用标准库 `csv`，`utf-8` 编码；先读后写，一个脚本一行，按 `id` 更新而不是追加重复行。
- 原子写（临时文件 + `os.replace`），并发写时加锁，原因同检查点：写到一半崩溃不能留下残缺的表。
- 下线的脚本把 `lifecycle` 改成 `retired`，不删行。历史信息对迁移和排查都有用。
- 以下情况都要更新对应行和 `updated_utc`：新建、迁移、改启动方式、改依赖、改路径、下线。

**用户要求展示脚本列表时**（"有哪些脚本""脚本列表""哪些在跑"等）：
1. 从 `what_script.csv` 读取，不要凭记忆，也不要只靠扫描目录。
2. 对 `active` 的行现场查实时状态（锁文件、心跳是否在更新、进程是否存在），在表里给出名称、用途、状态、启动方式、位置。
3. 把不一致的单独列出：登记了但路径不存在的、目录里有但没登记的。不要自作主张修改，问用户是否同步。
4. 展示完之后，回到用户真正要做的事，继续处理他接下来的需求。

**备份**：`what_script.csv`、脚本本体和依赖声明一起属于备份的"脚本层"，打包规则见 `hermes-backup`。

---

## 参考实现（Python，按需裁剪）

下面的骨架覆盖了：可配置根目录、分段日志、原子写、跨平台单实例锁、信号处理、检查点、心跳、预检、dry-run/limit、退出码语义。实际使用时替换 `list_items()` 和 `process()`，并按级别删减。

```python
#!/usr/bin/env python3
"""可续跑、可迁移、可观测的任务骨架。删减任何一块之前，先想清楚为什么可以删。"""
import argparse, json, logging, os, shutil, signal, sys, threading, time
from datetime import datetime, timezone
from pathlib import Path

SCHEMA_VERSION = 1
EX_INTERRUPTED = 75            # 被中断、未完成、可续跑
MIN_PY = (3, 9)
MIN_FREE_BYTES = 2 * 1024**3   # 剩余磁盘低于 2GB 就停
stop = threading.Event()


def utcnow() -> str:
    return datetime.now(timezone.utc).isoformat(timespec="seconds")


def job_home() -> Path:
    """工作根目录：环境变量优先，否则位于脚本旁。不依赖 cwd。"""
    default = Path(__file__).resolve().parent / "run_state"
    return Path(os.environ.get("JOB_HOME", default))


# ---------- 原子写 ----------
def atomic_write_json(path: Path, obj) -> None:
    tmp = path.with_name(f"{path.name}.tmp{os.getpid()}")
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump(obj, f, ensure_ascii=False)
        f.flush()
        os.fsync(f.fileno())
    os.replace(tmp, path)      # 同一文件系统内原子，Windows 与 POSIX 均可


# ---------- 分段日志：总量到上限时，删最旧的段直到 ≤ 保留比例 ----------
class SegmentedLogHandler(logging.Handler):
    def __init__(self, log_dir, prefix="job", seg_bytes=10 * 1024**2,
                 total_bytes=100 * 1024**2, keep_ratio=0.5):
        super().__init__()
        self.dir = Path(log_dir)
        self.dir.mkdir(parents=True, exist_ok=True)
        self.prefix, self.seg_bytes = prefix, seg_bytes
        self.total_bytes, self.keep_ratio = total_bytes, keep_ratio
        self._fh = None
        self._new_segment()

    def _segments(self):       # 文件名含 UTC 时间戳，字典序即时间序
        return sorted(self.dir.glob(f"{self.prefix}.*.log"))

    def _new_segment(self):
        if self._fh:
            self._fh.close()
        ts = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%S%f")
        self._fh = open(self.dir / f"{self.prefix}.{ts}.log", "a", encoding="utf-8")
        self._purge()

    def _purge(self):
        segs = self._segments()
        sizes = {p: p.stat().st_size for p in segs}
        total = sum(sizes.values())
        if total < self.total_bytes:
            return
        target = self.total_bytes * self.keep_ratio
        for p in segs[:-1]:    # 永不删除当前段
            if total <= target:
                break
            try:
                p.unlink()
                total -= sizes[p]
            except OSError:    # 例如 Windows 下被占用，下次再删
                pass

    def emit(self, record):
        try:
            self._fh.write(self.format(record) + "\n")
            self._fh.flush()
            if self._fh.tell() >= self.seg_bytes:
                self._new_segment()
        except Exception:
            self.handleError(record)

    def close(self):
        if self._fh:
            self._fh.close()
        super().close()


def setup_logging(log_dir: Path) -> None:
    fmt = logging.Formatter("%(asctime)sZ %(levelname)s %(message)s")
    fmt.converter = time.gmtime                    # 日志时间用 UTC
    handlers = [SegmentedLogHandler(log_dir), logging.StreamHandler(sys.stderr)]
    for h in handlers:
        h.setFormatter(fmt)
    logging.basicConfig(level=logging.INFO, handlers=handlers)
    sys.excepthook = lambda t, v, tb: logging.critical("未捕获异常", exc_info=(t, v, tb))


# ---------- 单实例锁（跨平台）----------
def acquire_lock(path: Path):
    f = open(path, "a+")
    try:
        f.seek(0)
        if os.name == "nt":
            import msvcrt
            msvcrt.locking(f.fileno(), msvcrt.LK_NBLCK, 1)
        else:
            import fcntl
            fcntl.flock(f, fcntl.LOCK_EX | fcntl.LOCK_NB)
    except OSError:
        sys.exit("另一个实例正在运行，退出。")
    return f                   # 调用方须持有此引用直到进程结束


# ---------- 信号：只设标志，不做重活 ----------
def install_signal_handlers() -> None:
    def handler(signum, frame):
        stop.set()
    for name in ("SIGTERM", "SIGINT", "SIGHUP", "SIGBREAK"):
        if hasattr(signal, name):
            signal.signal(getattr(signal, name), handler)


# ---------- 预检：尽早失败，并告诉人怎么修 ----------
def preflight(home: Path) -> None:
    if sys.version_info < MIN_PY:
        sys.exit(f"需要 Python >= {'.'.join(map(str, MIN_PY))}，当前 {sys.version.split()[0]}")
    if shutil.disk_usage(home).free < MIN_FREE_BYTES:
        sys.exit("磁盘剩余空间不足，请清理后重试")
    # 在这里补充：必要环境变量、外部程序（shutil.which）、依赖版本、网络连通性


# ---------- 检查点 ----------
def load_checkpoint(path: Path) -> dict:
    fresh = {"schema": SCHEMA_VERSION, "done": []}
    if not path.exists():
        return fresh
    try:
        ck = json.loads(path.read_text(encoding="utf-8"))
    except (json.JSONDecodeError, OSError):
        bad = path.with_suffix(".bad")
        os.replace(path, bad)
        sys.exit(f"检查点损坏，已保留为 {bad}。请人工确认后再决定是否从头开始。")
    if ck.get("schema") != SCHEMA_VERSION:
        sys.exit("检查点版本不匹配：请迁移或确认可以丢弃，避免静默重复处理。")
    return ck


def save_state(home: Path, done: set, total: int) -> None:
    atomic_write_json(home / "checkpoint.json", {"schema": SCHEMA_VERSION, "done": sorted(done)})
    atomic_write_json(home / "heartbeat.json", {"ts": utcnow(), "done": len(done), "total": total})


# ---------- 你需要实现的部分 ----------
def list_items():
    """返回稳定、有序、带确定性 ID 的待处理单元（ID 用相对路径或业务主键）。"""
    raise NotImplementedError


def process(item, dry_run: bool) -> None:
    """必须幂等；先原子写输出；所有网络/子进程调用都设超时。"""
    raise NotImplementedError


def verify(home: Path, items) -> bool:
    """完成前校验：数量对账、校验和、抽样检查。"""
    raise NotImplementedError


def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--dry-run", action="store_true", help="只打印将要做什么，不产生副作用")
    ap.add_argument("--limit", type=int, help="只处理前 N 项，用于小规模试跑")
    ap.add_argument("--doctor", action="store_true", help="只做环境自检")
    args = ap.parse_args()

    home = job_home()
    home.mkdir(parents=True, exist_ok=True)
    setup_logging(home / "logs")
    preflight(home)
    if args.doctor:
        logging.info("环境自检通过")
        return 0

    lock = acquire_lock(home / "job.lock")        # noqa: F841  必须保留引用
    install_signal_handlers()
    logging.info("启动 pid=%s argv=%s python=%s home=%s", os.getpid(), sys.argv,
                 sys.version.split()[0], home)

    done = set(load_checkpoint(home / "checkpoint.json")["done"])
    all_items = list(list_items())
    todo = [i for i in all_items if i not in done]
    if args.limit:
        todo = todo[: args.limit]
    logging.info("总数 %d，已完成 %d，本次待处理 %d", len(all_items), len(done), len(todo))

    last_save = time.monotonic()
    for n, item in enumerate(todo, 1):
        if stop.is_set():
            break
        process(item, args.dry_run)               # 先落输出
        if not args.dry_run:
            done.add(item)                        # 再记检查点
        if time.monotonic() - last_save > 30 or n % 100 == 0:
            save_state(home, done, len(all_items))
            last_save = time.monotonic()
            logging.info("进度 %d/%d", len(done), len(all_items))

    if not args.dry_run:
        save_state(home, done, len(all_items))
    if stop.is_set():
        logging.warning("收到停止信号，已保存检查点，可续跑")
        return EX_INTERRUPTED
    if not args.dry_run and not args.limit and not verify(home, all_items):
        logging.error("结果校验未通过")
        return 1
    logging.info("完成")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 用 systemd 启动（如果环境有 systemd）

```ini
# ~/.config/systemd/user/job.service  （路径、用户按实际环境填写，不要照抄）
[Unit]
Description=Long running job
StartLimitIntervalSec=600
StartLimitBurst=5

[Service]
WorkingDirectory=%h/project
Environment=JOB_HOME=%h/project/run_state
ExecStart=%h/project/.venv/bin/python job.py
Restart=on-failure
RestartSec=30
# 退出码 75 表示被中断但可续跑，也应当被拉起：
RestartForceExitStatus=75
KillSignal=SIGTERM
TimeoutStopSec=60

[Install]
WantedBy=default.target
```

没有 systemd 时的降级：`tmux new -d -s job 'python job.py'`，或 `nohup setsid python job.py >/dev/null 2>&1 &`（日志靠脚本自己写，不要依赖 nohup.out）。要让它在重启后自动恢复，还需要 cron `@reboot` 或同类机制，并在 `STATUS.md` 写明。

---

## STATUS.md 模板

每个长任务的工作目录里放一份，保持更新：

```markdown
# 任务状态：<任务名>

- 最后更新：<UTC 时间>
- 当前状态：运行中 / 已停止 / 已完成 / 失败（原因：…）

## 这个任务做什么
一两句话说明目的、输入、输出。

## 怎么运行
- 环境搭建：<命令>
- 环境自检：<命令，如 python job.py --doctor>
- 启动：<完整命令，含用了 systemd/tmux/nohup 哪一种>
- 小规模试跑：<命令，如 --limit 10>

## 怎么查看进度
- 心跳文件：<路径>（每 N 秒更新；超过 M 秒没更新视为卡死）
- 日志：<路径>（分段滚动，总量上限 100MB）
- 进度：<如何读取 x/N>

## 怎么停止 / 恢复
- 停止：<命令，如 systemctl --user stop job / kill -TERM $(cat ...)>
- 恢复：<同启动命令即可，从检查点续跑>
- 检查点位置与格式版本：<路径、schema 版本>

## 迁移到新机器
1. 复制 <目录列表>
2. 安装依赖：<命令>
3. 设置环境变量：<变量名列表，不含真实值>
4. 运行 --doctor，再用启动命令续跑

## 环境假设
- 已测试平台：…
- 未测试 / 不支持：…
- 依赖外部程序：…

## 已知问题与注意事项
- …

## 完成校验
- 校验了什么、结果如何、尚未校验的部分
```

---

## 验收：交付前必须实际做的几件事

只写不测等于没做。用下面的方式证明它真的健壮：

1. **杀进程测试**：任务跑到一半时执行 `kill -9`（Windows 用 `taskkill /F`），重新启动，确认从检查点续跑、没有重复处理、没有遗漏、没有残留的半截文件。
2. **重复启动测试**：同时启动第二个实例，确认被锁拒绝而不是两份互相覆盖。
3. **磁盘与超时测试**（至少想清楚）：外部服务不响应时，脚本是超时失败并重试，而不是永久挂起。
4. **搬家测试**：把工作目录复制到另一个路径（最好是另一台机器或干净容器），按 `STATUS.md` 的步骤操作，确认能跑通或续跑。做不到时，至少换一个路径、换一个 cwd 启动一次。
5. **日志上限测试**：临时把上限调小（如总量 1MB），确认旧段被淘汰、当前段不丢、程序不崩。
6. **结果校验**：用与脚本本身无关的方式核对输出（数量、抽样、校验和），再对用户宣布完成。
7. **登记核对**：`what_script.csv` 里有这个脚本的最新一行，路径是相对路径，`entry_command` 照抄能直接运行，没有写入任何密钥值。

测不了的项目，在交付说明里直说"未测试"。

---

## 常见反模式（看到就改）

- 脚本里出现 `/home/某用户/...`、`C:\...`、`localhost:某端口`、写死的主机名。
- 直接 `open(目标文件, "w")` 写检查点或结果。
- 先更新检查点，再写输出。
- `requests.get(url)` 不带 `timeout`；`subprocess.run(...)` 不带 `timeout`。
- `except Exception: pass`，把错误吞掉。
- 无限重试、没有退避；对 4xx 也重试。
- 用 `pickle` 存检查点；检查点里存绝对路径或 PID。
- 裸 `cmd &` 后台运行，输出靠终端。
- 日志只有 print，没有文件，或者没有上限把磁盘写满。
- 依赖只写"装一下 pandas"，不写版本和 Python 要求。
- 密钥写在脚本里，或被打进日志。
- 宣布"跑完了"，依据只是进程退出了。
- 没有 `STATUS.md`，下一个会话（或下一个人）不知道任务在不在跑、怎么停、怎么续。