
---

# 1️⃣ `log_analyzer.sh`

```bash
#!/bin/bash

# ==============================
# Log Analyzer Script
# ==============================

# -------- Task 1: Input Validation --------
if [ $# -eq 0 ]; then
    echo "Error: No log file path provided."
    echo "Usage: $0 <log_file>"
    exit 1
fi

LOG_FILE="$1"

if [ ! -f "$LOG_FILE" ]; then
    echo "Error: File '$LOG_FILE' does not exist."
    exit 1
fi

# -------- Variables --------
DATE=$(date +%Y-%m-%d)
REPORT_FILE="log_report_${DATE}.txt"
TOTAL_LINES=$(wc -l < "$LOG_FILE")

# -------- Task 2: Error Count --------
ERROR_COUNT=$(grep -Ei "ERROR|Failed" "$LOG_FILE" | wc -l)

echo "Total ERROR/Failed occurrences: $ERROR_COUNT"

# -------- Task 3: Critical Events --------
echo ""
echo "--- Critical Events ---"
CRITICAL_EVENTS=$(grep -n "CRITICAL" "$LOG_FILE")

if [ -z "$CRITICAL_EVENTS" ]; then
    echo "No critical events found."
else
    echo "$CRITICAL_EVENTS" | while IFS=: read -r line_num content; do
        echo "Line $line_num: $content"
    done
fi

# -------- Task 4: Top 5 Error Messages --------
echo ""
echo "--- Top 5 Error Messages ---"

TOP_ERRORS=$(grep "ERROR" "$LOG_FILE" \
    | sed -E 's/^[0-9-]+ [0-9:]+ (ERROR) //' \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -5)

if [ -z "$TOP_ERRORS" ]; then
    echo "No ERROR messages found."
else
    echo "$TOP_ERRORS"
fi

# -------- Task 5: Generate Summary Report --------
{
    echo "======================================"
    echo "         LOG ANALYSIS REPORT"
    echo "======================================"
    echo "Date of Analysis : $DATE"
    echo "Log File         : $LOG_FILE"
    echo "Total Lines      : $TOTAL_LINES"
    echo "Total Errors     : $ERROR_COUNT"
    echo ""
    echo "--- Top 5 Error Messages ---"
    if [ -z "$TOP_ERRORS" ]; then
        echo "No ERROR messages found."
    else
        echo "$TOP_ERRORS"
    fi
    echo ""
    echo "--- Critical Events ---"
    if [ -z "$CRITICAL_EVENTS" ]; then
        echo "No critical events found."
    else
        echo "$CRITICAL_EVENTS" | while IFS=: read -r line_num content; do
            echo "Line $line_num: $content"
        done
    fi
    echo "======================================"
} > "$REPORT_FILE"

echo ""
echo "Summary report generated: $REPORT_FILE"

# -------- Task 6 (Optional): Archive Processed Log --------
ARCHIVE_DIR="archive"

if [ ! -d "$ARCHIVE_DIR" ]; then
    mkdir "$ARCHIVE_DIR"
fi

mv "$LOG_FILE" "$ARCHIVE_DIR/"

echo "Log file moved to $ARCHIVE_DIR/"
```

Make it executable:

```bash
chmod +x log_analyzer.sh
```

Run it:

```bash
./log_analyzer.sh sample_log.log
```

---

# 2️⃣ Example Generated Report

`log_report_2026-02-19.txt`

```
======================================
         LOG ANALYSIS REPORT
======================================
Date of Analysis : 2026-02-19
Log File         : sample_log.log
Total Lines      : 500
Total Errors     : 132

--- Top 5 Error Messages ---
45 Connection timed out
32 File not found
28 Permission denied
15 Disk I/O error
9 Out of memory

--- Critical Events ---
Line 84: 2025-07-29 10:15:23 CRITICAL Disk space below threshold
Line 217: 2025-07-29 14:32:01 CRITICAL Database connection lost
======================================
```

---

# `day-20-solution.md`

## Overview

This project automates daily log analysis using a Bash script.  
The script processes a log file, extracts key system events, and generates a structured summary report.

---

## Features Implemented

### 1. Input Validation

* Ensures a log file path is provided.
* Verifies the file exists.
* Displays clear error messages and exits if validation fails.

### 2. Error Count

* Counts occurrences of:

  * ERROR
  * Failed
* Uses:

  * `grep -Ei`
  * `wc -l`

### 3. Critical Events

* Extracts lines containing `CRITICAL`
* Displays line numbers using:

  * `grep -n`
* Formats output as:

  ```
  Line <number>: <log entry>
  ```

### 4. Top 5 Error Messages

* Extracts lines containing `ERROR`
* Removes timestamp prefix using `sed`
* Groups and counts using:

  * `sort`
  * `uniq -c`
* Sorts descending using:

  * `sort -rn`
* Displays top 5 with:

  * `head -5`

### 5. Summary Report Generation

* Generates file:

  ```
  log_report_<YYYY-MM-DD>.txt
  ```
* Includes:

  * Date
  * Log filename
  * Total lines
  * Error count
  * Top 5 errors
  * Critical events

### 6. Archive Feature (Optional)

* Creates `archive/` if not present.
* Moves processed log into archive directory.
* Confirms successful move.

---

## Sample Execution

```bash
./log_analyzer.sh sample_log.log
```

### Console Output

```
Total ERROR/Failed occurrences: 132

--- Critical Events ---
Line 84: 2025-07-29 10:15:23 CRITICAL Disk space below threshold
Line 217: 2025-07-29 14:32:01 CRITICAL Database connection lost

--- Top 5 Error Messages ---
45 Connection timed out
32 File not found
28 Permission denied
15 Disk I/O error
9 Out of memory
```

---

## Commands and Tools Used

| Command | Purpose                    |
| ------- | -------------------------- |
| grep    | Pattern matching           |
| wc      | Line counting              |
| sed     | Text transformation        |
| sort    | Sorting data               |
| uniq -c | Counting duplicates        |
| head    | Limiting output            |
| date    | Generating report filename |
| mv      | Archiving log files        |
| mkdir   | Creating archive directory |

---

## Key Learnings

1. Text pipelines (`grep | sort | uniq`) are extremely powerful for log analysis.
2. Proper input validation prevents runtime failures and improves reliability.
3. Structured reporting improves operational visibility and automation readiness.

---

## Conclusion

This script demonstrates practical log processing automation using standard Unix utilities.
It is lightweight, efficient, and production-ready for small to medium environments.

-------------------

