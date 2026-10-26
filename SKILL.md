---
name: seekdb-deploy
version: 1.0.0
description: >-
  Deploy, start and validate a seekdb instance on Linux, macOS, Windows/WSL2 or Docker,
  then extract the deployment telemetry UUID from <base-dir>/run/telemetry.json.
  Use when the user asks to deploy / install / start seekdb, to obtain a seekdb
  telemetry or instance UUID (遥测 ID), or to diagnose a seekdb instance that will not
  start, cannot reach the telemetry endpoint, or produces no telemetry file.
  部署 seekdb、启动实例、获取遥测 ID、排查 seekdb 起不来、遥测上报不通，都用本技能。
  Do not use for query tuning, schema design, or application development against an
  already healthy seekdb instance.
---

# seekdb 部署与遥测 ID 提取

把一个 seekdb 实例从「装上」带到「跑通」，并拿到它的**部署遥测 ID**（36 位带连字符的 UUID）。

## 何时用 / 何时不用

**用**：部署或重装 seekdb；启动后确认能连上跑 SQL；拿到 `run/telemetry.json` 里的 `content.id`；实例起不来 / 连不上 / 没有遥测文件 / 遥测上报不出去时排查。

**不用**：已经健康的实例上的查询调优、表结构设计、业务开发；向量索引调参。

## 硬性前提

| 项 | 要求 |
|---|---|
| 资源 | 最低 1 物理核 + 2GB 可用内存（实测 1C1G 也能起，但不推荐） |
| 磁盘 | 数据盘 + 日志盘；建议 datafile 2G 起，log_disk 2G 起 |
| 出网 | **必须能访问 `https://openwebapi.oceanbase.com`**。遥测上报走这个端点；被墙或纯内网环境本地有 UUID 但平台查不到 |
| 端口 | 默认 SQL 端口 `2881`；官方 docker 示例另外映射 `2886`（RPC） |

## 第 1 步：选部署方式

| 环境 | 方式 | base-dir（决定遥测 ID 的路径因子） |
|---|---|---|
| Linux 云服务器（推荐） | `yum install seekdb` 或一键脚本 | `/var/lib/oceanbase` |
| 任意 Linux / WSL2 | 一键脚本或 RPM/DEB | `/var/lib/oceanbase` |
| macOS | `brew tap oceanbase/seekdb && brew install seekdb` | 脚本指定 |
| Docker（1.4+ 镜像） | `docker run` / compose | **`/var/lib/seekdb`** |
| Windows 原生 | 无官方二进制，走嵌入式 SDK | `<你的数据目录>` |

> ⚠️ **Docker 镜像 1.4+ 用 `/var/lib/seekdb`，1.3.x 旧镜像用 `/var/lib/oceanbase`**。搞错目录就找不到 `run/telemetry.json`。
> 官方文档明确：**在 Windows 上跑容器必须用 docker named volume，不要挂宿主目录**。

## 第 2 步：部署

### Linux（yum）

```bash
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/oceanbase/OceanBase.repo
sudo yum install -y seekdb obclient
sudo systemctl start seekdb
sudo systemctl status seekdb        # 需要看到 "seekdb is ready and running"
```

### Linux（一键脚本，适合 WSL2）

```bash
curl -fsSL https://obportal.s3.ap-southeast-1.amazonaws.com/download-center/opensource/seekdb/seekdb_install.sh | bash
```

### Docker

```bash
docker run -d --name seekdb \
  -p 2881:2881 -p 2886:2886 \
  -v seekdb:/var/lib/seekdb \
  -e CPU_COUNT=2 -e MEMORY_LIMIT=2G -e LOG_DISK_SIZE=2G -e DATAFILE_SIZE=2G \
  oceanbase/seekdb:latest
```

### 配置文件

两种安装方式下配置文件位置不同，两个都要看一眼：

```
/etc/oceanbase/seekdb.cnf     # yum / 老版本
/etc/seekdb/seekdb.cnf         # 新版脚本 / docker 镜像
```

常用项：

```ini
port=2881
data-dir=/var/lib/oceanbase/store
redo-dir=/var/lib/oceanbase/store/redo
cpu_count=2
memory_limit=2G
log_disk_size=2G
datafile_size=2G
datafile_maxsize=50G
```

> 改完配置要重启：`sudo systemctl restart seekdb`。

## 第 3 步：验证就绪

```bash
obclient -h127.0.0.1 -uroot -P2881 -A oceanbase -e "SELECT 1;"
```

能返回 `1` 才算真起来了。systemctl 显示 running 但连不上，多半是还在 bootstrap 或端口没监听。

Linux 上可直接跑一键自检：

```bash
bash scripts/seekdb_ready_check.sh
```

它会依次检查：进程/服务 → 端口监听 → SQL 连通 → 遥测端点出网 → 遥测文件与 ID。

## 第 4 步：提取遥测 ID

遥测文件在**实例运行目录**下：

```
<base-dir>/run/telemetry.json
```

结构：

```json
{ "content": { "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" } }
```

一行命令（题目通用写法）：

```bash
cd <base-dir>
python3 -c 'import json; print(json.load(open("run/telemetry.json"))["content"]["id"])'
```

跨平台脚本（Windows / Linux / macOS 通用，会自动搜候选目录）：

```bash
python scripts/seekdb_telemetry_id.py
python scripts/seekdb_telemetry_id.py --base-dir /var/lib/oceanbase
python scripts/seekdb_telemetry_id.py --json     # 输出完整 JSON，便于程序消费
```

拿到后**只提交 UUID 字符串本身**：不带引号、不带换行、不带 `"id":` 前缀、不提交整个文件或文件路径。

## 常见故障

| 现象 | 原因 | 处理 |
|---|---|---|
| 进程 exit 1，日志 `can not get mac address` / `OB_ERR_UNEXPECTED` | 受限环境拿不到 MAC 地址，UUID 节点初始化失败 | 用真实云主机 / 正常容器；确认没有网络访问被策略限制 |
| `observer init fail ... [server_start 4/14]` | 同上，bootstrap 中途失败 | 看 `log/seekdb.log` 里第一条 ERROR，前面那条 WDIAG 才是根因 |
| 起得来但 `run/telemetry.json` 不存在 | 还没写完 / 目录找错 / 嵌入式模式路径不同 | 确认当前目录是运行目录；搜 `find / -name telemetry.json 2>/dev/null` |
| 本地有 UUID 但平台查不到 | 遥测没上报出去（无外网 / 端点被拦） | `curl -I https://openwebapi.oceanbase.com`；确认遥测未被关闭 |
| 遥测被关了 | 设过 `TELEMETRY_ENABLED=false` | `export TELEMETRY_ENABLED=true` 并重启实例；**不要关**，否则上报没了 |
| 装了但 `systemctl` 找不到服务 | 用的是脚本/容器部署，不是 rpm | 改用进程检查或 `docker exec` 进容器操作 |

## 参考

- `references/deployment.md` —— 各平台完整部署配方、WSL2/Docker 前置条件、卸载
- `references/telemetry.md` —— 遥测 ID 的生成机制（为什么必须自己部署、为什么路径会影响 ID）

## 附录：脚本源码

以下脚本同时以独立文件形式提供在 `scripts/` 目录，此处内联源码，以便在只分发 `SKILL.md` 的环境下也能直接使用。

### `scripts/seekdb_telemetry_id.py`

```python
#!/usr/bin/env python3
"""Extract the seekdb deployment telemetry UUID from <base-dir>/run/telemetry.json.

Works on Linux, macOS and Windows. Searches the usual base directories when no
path is given, and prints the bare UUID so it can be copied into a form.

Usage:
    python seekdb_telemetry_id.py                 # auto-discover, print UUID
    python seekdb_telemetry_id.py --base-dir DIR  # explicit base dir
    python seekdb_telemetry_id.py --json          # print the whole content object
    python seekdb_telemetry_id.py --validate-only # only check that an ID is present
"""

from __future__ import annotations

import argparse
import json
import os
import re
import sys
from pathlib import Path

# <base-dir>/run/telemetry.json
TELEMETRY_RELATIVE_PATH = Path("run") / "telemetry.json"

# 36-character canonical UUID: 8-4-4-4-12 hex groups separated by hyphens.
UUID_PATTERN = re.compile(
    r"^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$"
)

# Ordered by likelihood. The Docker 1.4+ image uses /var/lib/seekdb; the RPM and
# the install script use /var/lib/oceanbase.
CANDIDATE_BASE_DIRS = [
    "/var/lib/seekdb",
    "/var/lib/oceanbase",
    "/var/lib/oceanbase/store",
    "/etc/oceanbase",
    "/opt/seekdb",
    "/opt/oceanbase",
    os.path.expanduser("~/seekdb"),
    os.path.expanduser("~/oceanbase"),
]


def is_uuid(value: object) -> bool:
    return isinstance(value, str) and bool(UUID_PATTERN.match(value.strip()))


def read_telemetry(base_dir: Path) -> dict | None:
    """Return the parsed telemetry document, or None when it is unusable."""
    path = base_dir / TELEMETRY_RELATIVE_PATH
    if not path.is_file():
        return None
    try:
        # utf-8-sig tolerates both a BOM-prefixed and a plain UTF-8 file.
        with path.open("r", encoding="utf-8-sig") as handle:
            document = json.load(handle)
    except (OSError, json.JSONDecodeError) as exc:
        print(f"[warn] cannot parse {path}: {exc}", file=sys.stderr)
        return None
    if not isinstance(document, dict):
        print(f"[warn] {path} does not contain a JSON object", file=sys.stderr)
        return None
    return document


def extract_id(document: dict) -> str | None:
    """Pull content.id out of the telemetry document.

    Older or partial documents may nest the id elsewhere, so fall back to a
    shallow scan for any canonical UUID string.
    """
    content = document.get("content")
    if isinstance(content, dict):
        candidate = content.get("id")
        if is_uuid(candidate):
            return candidate.strip()

    stack = [document]
    while stack:
        node = stack.pop()
        if isinstance(node, dict):
            for key, value in node.items():
                if key == "id" and is_uuid(value):
                    return value.strip()
                stack.append(value)
        elif isinstance(node, list):
            stack.extend(node)
        elif is_uuid(node):
            return node.strip()
    return None


def candidate_dirs(explicit: str | None) -> list[Path]:
    if explicit:
        return [Path(explicit).expanduser().resolve()]

    found: list[Path] = []
    for raw in CANDIDATE_BASE_DIRS:
        path = Path(raw)
        if path.is_dir() and path not in found:
            found.append(path)

    # Also treat the current working directory as a base dir, which is what the
    # documented one-liner assumes.
    cwd = Path.cwd().resolve()
    if cwd not in found:
        found.insert(0, cwd)

    # Last resort: walk a few likely roots for a run/telemetry.json.
    for root in ("/var/lib", os.path.expanduser("~")):
        root_path = Path(root)
        if not root_path.is_dir():
            continue
        try:
            for candidate in root_path.rglob("run/telemetry.json"):
                parent = candidate.parent.parent.resolve()
                if parent not in found:
                    found.append(parent)
        except (OSError, PermissionError):
            continue
    return found


def main() -> int:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--base-dir", help="seekdb instance base directory")
    parser.add_argument("--json", action="store_true", help="print the whole content object")
    parser.add_argument("--validate-only", action="store_true", help="only verify an ID is present")
    args = parser.parse_args()

    searched: list[str] = []
    for base_dir in candidate_dirs(args.base_dir):
        searched.append(str(base_dir))
        document = read_telemetry(base_dir)
        if document is None:
            continue

        if args.json:
            content = document.get("content")
            print(json.dumps(content if content is not None else document, ensure_ascii=False, indent=2))
            return 0

        telemetry_id = extract_id(document)
        if telemetry_id:
            if args.validate_only:
                print(f"[ok] telemetry id present in {base_dir}")
            else:
                print(telemetry_id)
            return 0

        print(f"[warn] {base_dir} has no content.id telemetry UUID", file=sys.stderr)

    print("[error] no telemetry UUID found.", file=sys.stderr)
    print(f"        searched base dirs: {', '.join(searched) if searched else '(none)'}", file=sys.stderr)
    print("        expected file: <base-dir>/run/telemetry.json", file=sys.stderr)
    print("        hints:", file=sys.stderr)
    print("          - the instance may still be bootstrapping; retry in a minute", file=sys.stderr)
    print("          - you may be in the wrong directory: cd into the instance base dir", file=sys.stderr)
    print("          - Docker 1.4+ images use /var/lib/seekdb, older ones /var/lib/oceanbase", file=sys.stderr)
    print("          - telemetry can be disabled with TELEMETRY_ENABLED=false; it must stay enabled", file=sys.stderr)
    return 1


if __name__ == "__main__":
    sys.exit(main())
```

### `scripts/seekdb_ready_check.sh`

```bash
#!/usr/bin/env bash
# Readiness check for a seekdb instance.
#
# Verifies, in order: process/service -> SQL port -> SQL query -> telemetry
# endpoint reachability -> telemetry file and UUID.
#
# Usage:
#   bash seekdb_ready_check.sh [base-dir] [sql-port]
#
# Defaults: base-dir=/var/lib/seekdb (Docker 1.4+) falling back to
# /var/lib/oceanbase, port=2881.

set -uo pipefail

BASE_DIR="${1:-}"
SQL_PORT="${2:-2881}"
TELEMETRY_ENDPOINT="https://openwebapi.oceanbase.com"

pass=0
fail=0
warn=0

ok()   { printf '  [ ok ] %s\n' "$1"; pass=$((pass + 1)); }
bad()  { printf '  [FAIL] %s\n' "$1"; fail=$((fail + 1)); }
note() { printf '  [warn] %s\n' "$1"; warn=$((warn + 1)); }

echo "== seekdb readiness check =="

# ---- 1. process / service -------------------------------------------------
if command -v systemctl >/dev/null 2>&1 && systemctl list-unit-files 2>/dev/null | grep -q '^seekdb\.service'; then
  if systemctl is-active --quiet seekdb; then
    ok "systemd service 'seekdb' is active"
  else
    bad "systemd service 'seekdb' is not active (systemctl status seekdb)"
  fi
elif pgrep -f '[s]eekdb' >/dev/null 2>&1; then
  ok "a seekdb process is running"
else
  bad "no seekdb process or systemd service found"
fi

# ---- 2. SQL port ----------------------------------------------------------
if command -v ss >/dev/null 2>&1 && ss -lnt 2>/dev/null | grep -q ":${SQL_PORT}\b"; then
  ok "SQL port ${SQL_PORT} is listening"
elif command -v netstat >/dev/null 2>&1 && netstat -lnt 2>/dev/null | grep -q ":${SQL_PORT}\b"; then
  ok "SQL port ${SQL_PORT} is listening"
else
  note "cannot confirm port ${SQL_PORT} is listening (no ss/netstat?)"
fi

# ---- 3. SQL query ---------------------------------------------------------
if command -v obclient >/dev/null 2>&1; then
  if obclient -h127.0.0.1 -uroot -P"${SQL_PORT}" -A oceanbase -e "SELECT 1;" >/dev/null 2>&1; then
    ok "SELECT 1 succeeded via obclient"
  else
    bad "obclient could not run SELECT 1 (instance still bootstrapping, or wrong port)"
  fi
else
  note "obclient not installed; skipping the SQL query check"
fi

# ---- 4. telemetry endpoint reachability -----------------------------------
if command -v curl >/dev/null 2>&1; then
  if curl -s -o /dev/null --max-time 10 "${TELEMETRY_ENDPOINT}"; then
    ok "telemetry endpoint ${TELEMETRY_ENDPOINT} is reachable"
  else
    bad "telemetry endpoint ${TELEMETRY_ENDPOINT} is NOT reachable; the UUID will never be reported"
  fi
else
  note "curl not available; cannot verify telemetry endpoint reachability"
fi

# ---- 5. telemetry file and UUID -------------------------------------------
if [ -z "${BASE_DIR}" ]; then
  for candidate in /var/lib/seekdb /var/lib/oceanbase /var/lib/oceanbase/store "$PWD"; do
    if [ -f "${candidate}/run/telemetry.json" ]; then
      BASE_DIR="${candidate}"
      break
    fi
  done
fi

if [ -n "${BASE_DIR}" ] && [ -f "${BASE_DIR}/run/telemetry.json" ]; then
  ok "telemetry file found at ${BASE_DIR}/run/telemetry.json"
  TELEMETRY_ID="$(python3 -c 'import json,sys; print(json.load(open(sys.argv[1]))["content"]["id"])' "${BASE_DIR}/run/telemetry.json" 2>/dev/null)"
  if [ -n "${TELEMETRY_ID}" ]; then
    ok "telemetry UUID: ${TELEMETRY_ID}"
    echo
    echo ">>> Submit this exact string (no quotes, no path, no file):"
    echo ">>> ${TELEMETRY_ID}"
  else
    bad "could not read content.id from the telemetry file"
  fi
else
  bad "no telemetry file found (looked in '${BASE_DIR:-auto}', /var/lib/seekdb, /var/lib/oceanbase, \$PWD)"
  note "the instance may still be bootstrapping, or you are not in the base dir"
fi

echo
echo "passed=${pass} failed=${fail} warnings=${warn}"
[ "${fail}" -eq 0 ]
```

<!-- OB-S:IYUEm_CcFM5hkQye6GFbIw -->
