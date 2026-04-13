# 06 — Agent Scripts

## agent/scripts/run-tier.sh

```bash
#!/bin/bash
set -euo pipefail

TIER="${1:?Usage: run-tier.sh <10min|3hr|24hr>}"
BASE_DIR="$HOME/{project}"
LOG_DIR="$BASE_DIR/agent/logs"
PYTHON="$BASE_DIR/agent/.venv/bin/python3"
CLAUDE="/opt/homebrew/bin/claude"
mkdir -p "$LOG_DIR"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$LOG_DIR/$TIER.log"; }

case "$TIER" in
  10min)
    # Check for raw files
    shopt -s nullglob
    md_files=("$BASE_DIR/sources/raw/"*.md)
    shopt -u nullglob
    if [ "${#md_files[@]}" -eq 0 ]; then
      exit 0
    fi

    # Internet check
    curl -sf --max-time 3 https://www.google.com > /dev/null 2>&1 || exit 0

    # Poll telegram responses (non-fatal)
    "$PYTHON" "$BASE_DIR/agent/scripts/telegram-bot.py" poll >> "$LOG_DIR/$TIER.log" 2>&1 || true

    log "START ${#md_files[@]} raw files"

    cd "$BASE_DIR"
    if "$CLAUDE" -p "$(cat "$BASE_DIR/agent/prompts/10min.md")" \
      --add-dir "$BASE_DIR" \
      --allowedTools "Read,Write,Edit,Bash,Glob,Grep" \
      --dangerously-skip-permissions \
      >> "$LOG_DIR/$TIER.log" 2>&1; then
      rm -f "$BASE_DIR/sources/raw/"*.md
    else
      log "ERROR: Claude failed, raw files preserved"
    fi

    # Urgent detection: kickstart 3hr if needed
    if grep -rlq -e "긴급" -e "urgent" -e "ASAP" "$BASE_DIR/sources/ingest/" 2>/dev/null; then
      log "URGENT detected, triggering 3hr"
      launchctl kickstart "gui/$(id -u)/com.$(whoami).ingest-3hr" 2>/dev/null || true
    fi

    log "DONE"
    ;;

  3hr)
    curl -sf --max-time 3 https://www.google.com > /dev/null 2>&1 || exit 0

    shopt -s nullglob
    ingest_files=("$BASE_DIR/sources/ingest/"*.md)
    shopt -u nullglob
    if [ "${#ingest_files[@]}" -eq 0 ]; then
      exit 0
    fi

    log "START ${#ingest_files[@]} ingest files"

    "$PYTHON" "$BASE_DIR/agent/scripts/telegram-bot.py" poll >> "$LOG_DIR/$TIER.log" 2>&1 || true

    cd "$BASE_DIR"
    if "$CLAUDE" -p "$(cat "$BASE_DIR/agent/prompts/3hr.md")" \
      --add-dir "$BASE_DIR" \
      --allowedTools "Read,Write,Edit,Bash,Glob,Grep" \
      --dangerously-skip-permissions \
      >> "$LOG_DIR/$TIER.log" 2>&1; then
      rm -f "$BASE_DIR/sources/ingest/"*.md
      rm -f "$BASE_DIR/sources/telegram/"*-response.md "$BASE_DIR/sources/telegram/"*-callback.md
    else
      log "ERROR: Claude failed, ingest files preserved"
    fi

    log "DONE"
    ;;

  24hr)
    curl -sf --max-time 3 https://www.google.com > /dev/null 2>&1 || exit 0

    log "START"

    "$BASE_DIR/agent/scripts/backup.sh" >> "$LOG_DIR/$TIER.log" 2>&1

    "$CLAUDE" -p "$(cat "$BASE_DIR/agent/prompts/24hr.md")" \
      --add-dir "$BASE_DIR" \
      --allowedTools "Read,Write,Edit,Bash,Glob,Grep" \
      --dangerously-skip-permissions \
      >> "$LOG_DIR/$TIER.log" 2>&1

    log "DONE"
    ;;

  *)
    echo "Error: unknown tier '$TIER'" >&2
    exit 1
    ;;
esac
```

---

## agent/scripts/telegram-bot.py

```python
#!/usr/bin/env python3
"""
Telegram Bot — decision requests and response collection.

Usage:
  telegram-bot.py send "message"
  telegram-bot.py send "question" "btn1" "btn2"
  telegram-bot.py poll
  telegram-bot.py queue "message" ["btn1" ...]
  telegram-bot.py flush

Environment (agent/.env):
  TELEGRAM_BOT_TOKEN=<token>
  TELEGRAM_CHAT_ID=<chat_id>
"""

import json, os, sys, urllib.error, urllib.parse, urllib.request
from datetime import datetime, timezone, timedelta
from pathlib import Path
from typing import List, Optional

REPO_ROOT = Path(__file__).resolve().parents[2]
ENV_FILE  = REPO_ROOT / "agent" / ".env"
SOURCES   = REPO_ROOT / "sources" / "telegram"
OUTBOX    = SOURCES / ".outbox"
LAST_ID   = SOURCES / ".last_update_id"
KST = timezone(timedelta(hours=9))

def load_env(path: Path) -> dict:
    env = {}
    if not path.exists(): return env
    for line in path.read_text().splitlines():
        line = line.strip()
        if not line or line.startswith("#"): continue
        if "=" in line:
            key, _, val = line.partition("=")
            env[key.strip()] = val.strip()
    return env

def get_config() -> tuple[str, str]:
    env = load_env(ENV_FILE)
    token = env.get("TELEGRAM_BOT_TOKEN") or os.environ.get("TELEGRAM_BOT_TOKEN", "")
    chat_id = env.get("TELEGRAM_CHAT_ID") or os.environ.get("TELEGRAM_CHAT_ID", "")
    if not token or not chat_id:
        sys.exit("[telegram-bot] ERROR: TELEGRAM_BOT_TOKEN and TELEGRAM_CHAT_ID must be set in agent/.env")
    return token, chat_id

def api_call(token: str, method: str, payload: dict) -> dict:
    url = f"https://api.telegram.org/bot{token}/{method}"
    data = json.dumps(payload).encode()
    req = urllib.request.Request(url, data=data, headers={"Content-Type": "application/json"}, method="POST")
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:
            return json.loads(resp.read())
    except urllib.error.HTTPError as exc:
        body = exc.read().decode(errors="replace")
        sys.exit(f"[telegram-bot] HTTP {exc.code}: {body}")
    except urllib.error.URLError as exc:
        sys.exit(f"[telegram-bot] Network error: {exc.reason}")

def build_inline_keyboard(buttons: list[str]) -> list:
    return [[{"text": label, "callback_data": f"btn_{i}_{label}"}] for i, label in enumerate(buttons)]

def send_message(token: str, chat_id: str, text: str, buttons: Optional[List[str]] = None) -> dict:
    payload: dict = {"chat_id": chat_id, "text": text}
    if buttons:
        payload["reply_markup"] = {"inline_keyboard": build_inline_keyboard(buttons)}
    result = api_call(token, "sendMessage", payload)
    if result.get("ok"):
        print(f"[telegram-bot] Sent message_id={result['result']['message_id']}")
    return result

def now_kst() -> datetime: return datetime.now(tz=KST)
def kst_filename_ts(dt: datetime) -> str: return dt.strftime("%Y%m%d-%H%M%S")
def kst_iso(dt: datetime) -> str: return dt.strftime("%Y-%m-%dT%H:%M:%S+09:00")

def read_last_id() -> int:
    if LAST_ID.exists():
        try: return int(LAST_ID.read_text().strip())
        except ValueError: pass
    return 0

def write_last_id(update_id: int) -> None:
    LAST_ID.write_text(str(update_id))

def save_response(text: str, from_user: str) -> Path:
    SOURCES.mkdir(parents=True, exist_ok=True)
    dt = now_kst()
    fname = SOURCES / f"{kst_filename_ts(dt)}-response.md"
    content = f"---\nsource: telegram\ntype: response\nreceived: {kst_iso(dt)}\nfrom: {from_user}\n---\n\n{text}\n"
    fname.write_text(content)
    return fname

def save_callback(callback_data: str, question_text: str, from_user: str) -> Path:
    SOURCES.mkdir(parents=True, exist_ok=True)
    dt = now_kst()
    fname = SOURCES / f"{kst_filename_ts(dt)}-callback.md"
    content = f"---\nsource: telegram\ntype: callback\nreceived: {kst_iso(dt)}\nfrom: {from_user}\nquestion: {question_text!r}\n---\n\n{callback_data}\n"
    fname.write_text(content)
    return fname

def answer_callback_query(token: str, callback_query_id: str) -> None:
    api_call(token, "answerCallbackQuery", {"callback_query_id": callback_query_id})

def poll(token: str, chat_id: str) -> None:
    last_id = read_last_id()
    params: dict = {"timeout": 5, "allowed_updates": ["message", "callback_query"]}
    if last_id: params["offset"] = last_id + 1
    result = api_call(token, "getUpdates", params)
    updates = result.get("result", [])
    if not updates:
        print("[telegram-bot] No new updates.")
        return
    for update in updates:
        uid = update["update_id"]
        if "message" in update:
            msg = update["message"]
            sender_id = str(msg.get("from", {}).get("id", ""))
            if str(msg.get("chat", {}).get("id", "")) != str(chat_id) and sender_id != str(chat_id):
                write_last_id(uid); continue
            text = msg.get("text", "")
            username = msg.get("from", {}).get("username") or msg.get("from", {}).get("first_name", "unknown")
            if text:
                path = save_response(text, username)
                print(f"[telegram-bot] Saved response → {path.name}")
        elif "callback_query" in update:
            cq = update["callback_query"]
            callback_data = cq.get("data", "")
            from_user = cq.get("from", {}).get("username") or cq.get("from", {}).get("first_name", "unknown")
            question_text = cq.get("message", {}).get("text", "")
            path = save_callback(callback_data, question_text, from_user)
            print(f"[telegram-bot] Saved callback → {path.name}")
            answer_callback_query(token, cq["id"])
        write_last_id(uid)

def queue_message(text: str, buttons: Optional[List[str]] = None) -> None:
    OUTBOX.mkdir(parents=True, exist_ok=True)
    ts = now_kst().strftime("%Y%m%d-%H%M%S-%f")
    fpath = OUTBOX / f"{ts}.json"
    fpath.write_text(json.dumps({"text": text, "buttons": buttons}, ensure_ascii=False, indent=2))
    print(f"[telegram-bot] Queued → {fpath.name}")

def flush(token: str, chat_id: str) -> None:
    OUTBOX.mkdir(parents=True, exist_ok=True)
    files = sorted(OUTBOX.glob("*.json"))
    if not files: print("[telegram-bot] Outbox is empty."); return
    for fpath in files:
        try:
            data = json.loads(fpath.read_text())
            text = data.get("text", "")
            if not text: fpath.unlink(); continue
            send_message(token, chat_id, text, data.get("buttons"))
            fpath.unlink()
        except Exception as exc:
            print(f"[telegram-bot] ERROR flushing {fpath.name}: {exc}")

def main() -> None:
    args = sys.argv[1:]
    if not args: print(__doc__); sys.exit(0)
    cmd = args[0]
    if cmd == "send":
        if len(args) < 2: sys.exit("Usage: telegram-bot.py send \"message\" [\"btn1\" ...]")
        token, chat_id = get_config()
        send_message(token, chat_id, args[1], args[2:] if len(args) > 2 else None)
    elif cmd == "poll":
        token, chat_id = get_config()
        poll(token, chat_id)
    elif cmd == "flush":
        token, chat_id = get_config()
        flush(token, chat_id)
    elif cmd == "queue":
        if len(args) < 2: sys.exit("Usage: telegram-bot.py queue \"message\" [\"btn1\" ...]")
        queue_message(args[1], args[2:] if len(args) > 2 else None)
    else:
        sys.exit(f"[telegram-bot] Unknown command: {cmd!r}")

if __name__ == "__main__":
    main()
```

---

## agent/scripts/backup.sh

```bash
#!/bin/bash
set -euo pipefail

SOURCE_DIR="$HOME/{project}/memory"
BACKUP_BASE_DIR="$HOME/Library/Mobile Documents/com~apple~CloudDocs/{project}-backup"
MAX_BACKUPS=10

mkdir -p "$BACKUP_BASE_DIR"
TIMESTAMP=$(date '+%Y-%m-%d-%H%M')
ARCHIVE_NAME="memory-${TIMESTAMP}.tar.gz"
ARCHIVE_PATH="$BACKUP_BASE_DIR/$ARCHIVE_NAME"

tar -czf "$ARCHIVE_PATH" -C "$HOME/{project}" memory

FILE_SIZE=$(du -h "$ARCHIVE_PATH" | cut -f1)
BACKUP_COUNT=$(ls -1 "$BACKUP_BASE_DIR"/memory-*.tar.gz 2>/dev/null | wc -l)

if [ "$BACKUP_COUNT" -gt "$MAX_BACKUPS" ]; then
    DELETE_COUNT=$((BACKUP_COUNT - MAX_BACKUPS))
    ls -1 "$BACKUP_BASE_DIR"/memory-*.tar.gz | sort | head -n "$DELETE_COUNT" | xargs -I {} rm -f "{}"
    BACKUP_COUNT=$MAX_BACKUPS
fi

echo "[backup] Creating: $ARCHIVE_NAME"
echo "[backup] Size: $FILE_SIZE"
echo "[backup] Done. Total backups: $BACKUP_COUNT"
```

---

## agent/scripts/dedup-ingest.sh

```bash
#!/bin/bash
set -euo pipefail

REPO_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
RAW_DIR="${REPO_ROOT}/sources/raw"
INGEST_DIR="${REPO_ROOT}/sources/ingest"
SEEN_HASHES="${REPO_ROOT}/sources/.seen_hashes"
HASH_THRESHOLD=10000
HASH_TRIM_TARGET=5000

if [[ ! -f "${SEEN_HASHES}" ]]; then touch "${SEEN_HASHES}"; fi

processed_count=0
skipped_count=0

shopt -s nullglob
md_files=("${RAW_DIR}"/*.md)
shopt -u nullglob

if [[ ${#md_files[@]} -eq 0 ]]; then
    echo "[dedup-ingest] No .md files found in ${RAW_DIR}"
    exit 0
fi

for file in "${md_files[@]}"; do
    filename=$(basename "$file")
    file_hash=$(shasum -a 256 "$file" | awk '{print $1}')

    if grep -q "^${file_hash}$" "${SEEN_HASHES}"; then
        echo "[dedup-ingest] Skip (dup): ${filename}"
        ((skipped_count++))
        continue
    fi

    timestamp=$(date +%s)
    base_name="${filename%.md}"
    new_filename="${base_name}-${timestamp}.md"
    mv "$file" "${INGEST_DIR}/${new_filename}"
    echo "[dedup-ingest] Processed: ${filename} → ${new_filename}"
    echo "${file_hash}" >> "${SEEN_HASHES}"
    ((processed_count++))
done

line_count=$(wc -l < "${SEEN_HASHES}")
if [[ ${line_count} -gt ${HASH_THRESHOLD} ]]; then
    tail -n "${HASH_TRIM_TARGET}" "${SEEN_HASHES}" > "${SEEN_HASHES}.tmp"
    mv "${SEEN_HASHES}.tmp" "${SEEN_HASHES}"
fi

echo "[dedup-ingest] Done: ${processed_count} files processed"
exit 0
```

---

## agent/scripts/calendar.sh

```bash
#!/bin/bash
# macOS 캘린더 조회 스크립트
# 사용법:
#   ./agent/scripts/calendar.sh              # 오늘
#   ./agent/scripts/calendar.sh today        # 오늘
#   ./agent/scripts/calendar.sh tomorrow     # 내일
#   ./agent/scripts/calendar.sh week         # 이번 주 (7일)
#   ./agent/scripts/calendar.sh 3            # 오늘부터 3일
#   ./agent/scripts/calendar.sh 2026-04-15   # 특정 날짜

set -euo pipefail

run_offset() {
    local days_offset=$1
    local days_range=$2
    local tmpfile
    tmpfile=$(mktemp /tmp/calendar_query.XXXXXX.scpt)
    cat > "$tmpfile" << 'ENDSCRIPT'
tell application "Calendar" to launch
delay 1
tell application "Calendar"
    set baseDate to current date
    set hours of baseDate to 0
    set minutes of baseDate to 0
    set seconds of baseDate to 0
    set targetDate to baseDate + (DAYS_OFFSET * days)
    set endDate to targetDate + (DAYS_RANGE * days)
    set output to ""
    repeat with cal in every calendar
        set calName to name of cal
        try
            set evts to (every event of cal whose start date ≥ targetDate and start date < endDate)
            repeat with evt in evts
                set evtSummary to summary of evt
                set evtStart to start date of evt
                set evtEnd to end date of evt
                try
                    set evtLoc to location of evt
                    if evtLoc is missing value then set evtLoc to ""
                on error
                    set evtLoc to ""
                end try
                try
                    set evtDesc to description of evt
                    if evtDesc is missing value then set evtDesc to ""
                on error
                    set evtDesc to ""
                end try
                set output to output & calName & " | " & evtSummary & " | " & (evtStart as string) & " ~ " & (evtEnd as string) & " | " & evtLoc & " | " & evtDesc & linefeed
            end repeat
        end try
    end repeat
    return output
end tell
ENDSCRIPT
    sed -i '' "s/DAYS_OFFSET/$days_offset/g; s/DAYS_RANGE/$days_range/g" "$tmpfile"
    osascript "$tmpfile"
    rm -f "$tmpfile"
}

run_date() {
    local year=$1 month=$2 day=$3
    local tmpfile
    tmpfile=$(mktemp /tmp/calendar_query.XXXXXX.scpt)
    cat > "$tmpfile" << 'ENDSCRIPT'
tell application "Calendar" to launch
delay 1
tell application "Calendar"
    set targetDate to current date
    set day of targetDate to 1
    set month of targetDate to TARGET_MONTH
    set year of targetDate to TARGET_YEAR
    set day of targetDate to TARGET_DAY
    set hours of targetDate to 0
    set minutes of targetDate to 0
    set seconds of targetDate to 0
    set endDate to targetDate + (1 * days)
    set output to ""
    repeat with cal in every calendar
        set calName to name of cal
        try
            set evts to (every event of cal whose start date ≥ targetDate and start date < endDate)
            repeat with evt in evts
                set evtSummary to summary of evt
                set evtStart to start date of evt
                set evtEnd to end date of evt
                try
                    set evtLoc to location of evt
                    if evtLoc is missing value then set evtLoc to ""
                on error
                    set evtLoc to ""
                end try
                try
                    set evtDesc to description of evt
                    if evtDesc is missing value then set evtDesc to ""
                on error
                    set evtDesc to ""
                end try
                set output to output & calName & " | " & evtSummary & " | " & (evtStart as string) & " ~ " & (evtEnd as string) & " | " & evtLoc & " | " & evtDesc & linefeed
            end repeat
        end try
    end repeat
    return output
end tell
ENDSCRIPT
    sed -i '' "s/TARGET_YEAR/$year/g; s/TARGET_MONTH/$month/g; s/TARGET_DAY/$day/g" "$tmpfile"
    osascript "$tmpfile"
    rm -f "$tmpfile"
}

ARG="${1:-today}"

case "$ARG" in
    today)    echo "=== 오늘 일정 ==="; run_offset 0 1 ;;
    tomorrow) echo "=== 내일 일정 ==="; run_offset 1 1 ;;
    week)     echo "=== 이번 주 일정 (7일) ==="; run_offset 0 7 ;;
    [0-9]|[0-9][0-9])
        echo "=== 오늘부터 ${ARG}일간 일정 ==="; run_offset 0 "$ARG" ;;
    [0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9])
        Y=$(echo "$ARG" | cut -d- -f1)
        M=$((10#$(echo "$ARG" | cut -d- -f2)))
        D=$((10#$(echo "$ARG" | cut -d- -f3)))
        echo "=== ${ARG} 일정 ==="; run_date "$Y" "$M" "$D" ;;
    *)
        echo "사용법: calendar.sh [today|tomorrow|week|N|YYYY-MM-DD]"
        exit 1 ;;
esac
```
