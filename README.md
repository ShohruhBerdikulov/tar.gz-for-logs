#!/bin/bash

set -Eeuo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ARCHIVE_ROOT="$SCRIPT_DIR/logs"
TODAY="$(date +%F)"

ARCHIVED=0
SKIPPED=0
ERRORS=0

mkdir -p "$ARCHIVE_ROOT"

echo "========================================"
echo " Log Archive Started"
echo " Date      : $TODAY"
echo " Root Path : $SCRIPT_DIR"
echo " Archive   : $ARCHIVE_ROOT"
echo "========================================"

archive_log() {
local log_file="$1"

```
local file_dir
file_dir="$(dirname "$log_file")"

local rel_dir
rel_dir="${file_dir#$SCRIPT_DIR}"
rel_dir="${rel_dir#/}"

local dest_dir="$ARCHIVE_ROOT"

if [[ -n "$rel_dir" ]]; then
    dest_dir="$ARCHIVE_ROOT/$rel_dir"
fi

mkdir -p "$dest_dir"

local base_name
base_name="$(basename "$log_file" .log)"

local archive_path="$dest_dir/${base_name}_${TODAY}.tar.gz"

if [[ -e "$archive_path" ]]; then
    local counter=1

    while [[ -e "$dest_dir/${base_name}_${TODAY}_${counter}.tar.gz" ]]; do
        ((counter++))
    done

    archive_path="$dest_dir/${base_name}_${TODAY}_${counter}.tar.gz"
fi

echo
echo "[ARCHIVE]"
echo "  Source : $log_file"
echo "  Target : $archive_path"

if tar -czf "$archive_path" \
    -C "$file_dir" \
    "$(basename "$log_file")"; then

    rm -f "$log_file"

    echo "  Status : SUCCESS"

    ((++ARCHIVED))
else
    rm -f "$archive_path" 2>/dev/null || true

    echo "  Status : FAILED"

    ((++ERRORS))
fi
```

}

while IFS= read -r -d '' LOG_FILE; do
archive_log "$LOG_FILE"
done < <(
find "$SCRIPT_DIR" 
-path "$ARCHIVE_ROOT" -prune -o 
-type f 
-name "*.log" 
-daystart 
-not -mtime 0 
-print0
)

echo
echo "[SKIPPED - MODIFIED TODAY]"

while IFS= read -r -d '' LOG_FILE; do
echo "  $LOG_FILE"
((++SKIPPED))
done < <(
find "$SCRIPT_DIR" 
-path "$ARCHIVE_ROOT" -prune -o 
-type f 
-name "*.log" 
-daystart 
-mtime 0 
-print0
)

find "$SCRIPT_DIR" 
-path "$ARCHIVE_ROOT" -prune -o 
-mindepth 1 
-type d 
-empty 
-print0 |
sort -rz |
while IFS= read -r -d '' DIR; do
rmdir "$DIR" 2>/dev/null || true
done

echo
echo "========================================"
echo " Archive Completed"
echo "========================================"
echo " Archived : $ARCHIVED"
echo " Skipped  : $SKIPPED"
echo " Errors   : $ERRORS"
echo "========================================"

[[ $ERRORS -eq 0 ]]





# tar.gz-for-logs
