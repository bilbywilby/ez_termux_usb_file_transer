# ez_termux_usb_file_transer

A powerful CLI tool for transferring files from Termux to USB flash drives on Android.

## Features

- 📡 **Auto-detection** of USB drives via `/proc/mounts` parsing
- 🎨 **Colored interactive CLI** with menus and progress
- 📦 **rsync-powered transfers** with FAT32/exFAT-safe flags
- 🗑️ **Safe delete** with typed confirmation
- 📁 **Staging fallback** for devices without direct USB write access
- ⚙️ **Configurable** via `config/transfer.conf`
- 📋 **Logging** to `~/.ez_usb_transfer.log`
- ⏰ **Scheduler-ready** (works with `termux-job-scheduler`)

## Requirements

- Android 11+ (Android 12+ recommended)
- Termux (F-Droid build recommended)
- USB OTG adapter
- USB flash drive formatted as FAT32, exFAT, or NTFS

## Installation

```bash
# Clone
git clone https://github.com/bilbywilby/ez_termux_usb_file_transer.git
cd ez_termux_usb_file_transer

# Run setup (installs deps, grants permissions)
./scripts/setup.sh

# Run
./ez_usb_transfer.sh
```

## Quick Start

```bash
./ez_usb_transfer.sh
```

You'll be presented with a menu:

```
  ╔══════════════════════════════════════════════════════════╗
  ║          EZ TERMUX → USB FLASH DRIVE TRANSFER           ║
  ╚══════════════════════════════════════════════════════════╝

  Main Menu
  ┌─────────────────────────────────────────────┐
  │ 1  Scan & Display All Locations            │
  │ 2  Select Source (Termux directory)        │
  │ 3  Select Destination (USB drive)          │
  │ 4  Transfer (Source → Destination)         │
  │ 5  Transfer + Delete Source                │
  │ 6  Stage to Internal Storage (fallback)    │
  │ 7  Browse Source Directory                 │
  │ 8  View Transfer Log                       │
  │ 0  Quit                                    │
  └─────────────────────────────────────────────┘
```

## Configuration

Edit `config/transfer.conf` to customize:

```bash
# Default source directory
DEFAULT_SOURCE="$HOME"

# Default destination (auto-detected if empty)
DEFAULT_DEST=""

# rsync flags for FAT32/exFAT (default)
RSYNC_FAT_FLAGS="-vrt --progress --inplace --omit-dir-times --no-perms"

# rsync flags for ext4/f2fs
RSYNC_UNIX_FLAGS="-avz --progress"

# Log file location
LOG_FILE="$HOME/.ez_usb_transfer.log"

# Max log file size in KB (0 = unlimited)
LOG_MAX_SIZE_KB=5120
```

## Non-Interactive / Scheduled Mode

```bash
# Transfer a specific directory (exits if no USB present)
./ez_usb_transfer.sh --src ~/my_project --dest auto

# Transfer and delete source
./ez_usb_transfer.sh --src ~/my_project --dest auto --delete

# Just scan and report
./ez_usb_transfer.sh --scan
```

### With termux-job-scheduler:

```bash
pkg install termux-job-scheduler
termux-job-scheduler -s ./ez_usb_transfer.sh --period-ms 21600000 --persisted true
```

## Troubleshooting

See [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) for common issues.

## License

MIT — see [LICENSE](LICENSE).
```

---

## `LICENSE`

```
MIT License

Copyright (c) 2024 bilbywilby

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## `.gitignore`

```
# Logs
*.log
~/.ez_usb_transfer.log

# Temp files
*.tmp
*.swp
*~

# OS files
.DS_Store
Thumbs.db

# Config overrides (keep defaults in repo)
config/transfer.conf.local
```

---

## `ez_usb_transfer.sh` (Main Entry Point)

```bash
#!/data/data/com.termux/files/usr/bin/bash
#
# ez_usb_transfer.sh — EZ Termux → USB Flash Drive Transfer Tool
# Version: 1.0.0
#
# Usage:
#   ./ez_usb_transfer.sh              # Interactive mode
#   ./ez_usb_transfer.sh --scan       # Scan only
#   ./ez_usb_transfer.sh --src <dir> --dest <path>  # Non-interactive
#   ./ez_usb_transfer.sh --src <dir> --dest auto --delete
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LIB_DIR="$SCRIPT_DIR/lib"
CONFIG_FILE="$SCRIPT_DIR/config/transfer.conf"

# Load config
[ -f "$CONFIG_FILE" ] && source "$CONFIG_FILE"

# Load libraries
source "$LIB_DIR/utils.sh"
source "$LIB_DIR/detect.sh"
source "$LIB_DIR/transfer.sh"

# Defaults
DEFAULT_SOURCE="${DEFAULT_SOURCE:-$HOME}"
DEFAULT_DEST="${DEFAULT_DEST:-}"
LOG_FILE="${LOG_FILE:-$HOME/.ez_usb_transfer.log}"
LOG_MAX_SIZE_KB="${LOG_MAX_SIZE_KB:-5120}"

# State
SOURCE="${SOURCE:-}"
DEST="${DEST:-}"

# ─── Argument Parsing ─────────────────────────────────────────────────────────

parse_args() {
    local mode="interactive"
    local delete_flag=false

    while [ $# -gt 0 ]; do
        case "$1" in
            --scan)
                mode="scan"
                shift
                ;;
            --src)
                mode="non-interactive"
                SOURCE="$2"
                shift 2
                ;;
            --dest)
                DEST="$2"
                [ "$DEST" = "auto" ] && DEST="$(detect_first_usb || true)"
                shift 2
                ;;
            --delete)
                delete_flag=true
                shift
                ;;
            --help|-h)
                show_help
                exit 0
                ;;
            *)
                error "Unknown option: $1"
                show_help
                exit 1
                ;;
        esac
    done

    case "$mode" in
        scan)
            scan_all
            exit 0
            ;;
        non-interactive)
            non_interactive_transfer "$delete_flag"
            ;;
        interactive)
            interactive_loop
            ;;
    esac
}

non_interactive_transfer() {
    local delete_flag="$1"
    log "INFO" "Non-interactive mode: $SOURCE → $DEST"

    if [ -z "$DEST" ]; then
        log "ERROR" "No USB drive found. Exiting."
        echo "No USB drive detected. Plug one in and re-run." >&2
        exit 1
    fi

    if [ ! -d "$SOURCE" ]; then
        log "ERROR" "Source not found: $SOURCE"
        echo "Source not found: $SOURCE" >&2
        exit 1
    fi

    perform_transfer "$SOURCE" "$DEST"

    if [ "$delete_flag" = true ]; then
        log "WARN" "Deleting source: $SOURCE"
        rm -rf "$SOURCE"
        log "INFO" "Source deleted."
        echo "Transfer complete. Source deleted."
    else
        echo "Transfer complete. Source preserved at: $SOURCE"
    fi
}

show_help() {
    cat <<EOF
${BOLD}ez_usb_transfer.sh${NC} — Termux → USB Flash Drive Transfer Tool

${BOLD}Usage:${NC}
  ./ez_usb_transfer.sh [OPTIONS]

${BOLD}Options:${NC}
  (no args)              Interactive menu mode
  --scan                 Scan and display all locations, then exit
  --src <dir>            Source directory (non-interactive)
  --dest <path|auto>     Destination (USB path or 'auto' to detect)
  --delete               Delete source after successful transfer
  -h, --help             Show this help

${BOLD}Examples:${NC}
  ./ez_usb_transfer.sh
  ./ez_usb_transfer.sh --scan
  ./ez_usb_transfer.sh --src ~/project --dest auto
  ./ez_usb_transfer.sh --src ~/project --dest /mnt/media_rw/ABCD-1234 --delete
EOF
}

# ─── Interactive Menu ─────────────────────────────────────────────────────────

interactive_loop() {
    # Pre-flight
    preflight_check

    while true; do
        menu_main
    done
}

preflight_check() {
    if [ ! -d "$HOME/storage" ]; then
        clear_screen
        printf "\n  ${BOLD}${CYAN}EZ USB Transfer Setup${NC}\n\n"
        warn "Storage permission not configured."
        printf "  Running 'termux-setup-storage'...\n\n"
        termux-setup-storage
        printf "\n  ${GREEN}Done.${NC}\n"
        pause
    fi
}

menu_main() {
    header
    printf "\n  ${BOLD}Main Menu${NC}\n"
    printf "  ┌─────────────────────────────────────────────┐\n"
    printf "  │ ${CYAN}1${NC}  Scan & Display All Locations           │\n"
    printf "  │ ${CYAN}2${NC}  Select Source (Termux directory)        │\n"
    printf "  │ ${CYAN}3${NC}  Select Destination (USB drive)          │\n"
    printf "  │ ${CYAN}4${NC}  Transfer (Source → Destination)         │\n"
    printf "  │ ${CYAN}5${NC}  Transfer + Delete Source                │\n"
    printf "  │ ${CYAN}6${NC}  Stage to Internal Storage (fallback)    │\n"
    printf "  │ ${CYAN}7${NC}  Browse Source Directory                 │\n"
    printf "  │ ${CYAN}8${NC}  View Transfer Log                       │\n"
    printf "  │ ${CYAN}0${NC}  Quit                                    │\n"
    printf "  └─────────────────────────────────────────────┘\n"

    # Show current selections
    printf "\n"
    [ -n "$SOURCE" ] && printf "  ${DIM}Source: %s${NC}\n" "$SOURCE"
    [ -n "$DEST" ] && printf "  ${DIM}Dest:   %s${NC}\n" "$DEST"

    printf "\n  ${BOLD}>${NC} "
    read -r choice

    case "$choice" in
        1) scan_all ;;
        2) select_source ;;
        3) select_destination ;;
        4) menu_transfer false ;;
        5) menu_transfer true ;;
        6) stage_fallback ;;
        7) browse_source ;;
        8) view_log ;;
        0) printf "\n  ${GREEN}Bye.${NC}\n"; log "INFO" "Session ended."; exit 0 ;;
        *) warn "Invalid option."; pause ;;
    esac
}

menu_transfer() {
    local delete_flag="$1"
    require_source_dest
    [ -z "$SOURCE" ] || [ -z "$DEST" ] && return
    perform_transfer "$SOURCE" "$DEST" "$delete_flag"
}

view_log() {
    header
    printf "\n  ${BOLD}Transfer Log${NC}\n\n"
    if [ -f "$LOG_FILE" ]; then
        tail -50 "$LOG_FILE" | sed "s/^/  /"
    else
        info "No log file yet. Transfers will be logged to: $LOG_FILE"
    fi
    pause
}

# ─── Scan ─────────────────────────────────────────────────────────────────────

scan_all() {
    header
    printf "\n  ${BOLD}─── TERMUX FILE LOCATIONS ───${NC}\n\n"

    local termux_dirs
    termux_dirs=$(list_termux_dirs)
    if [ -n "$termux_dirs" ]; then
        printf "  ${BOLD}%-50s %s${NC}\n" "Path" "Size"
        printf "  %-50s %s\n" "──────────────────────────────────────────────────" "────"
        while IFS=$'\t' read -r path size; do
            printf "  %-50s ${CYAN}%s${NC}\n" "$path" "$size"
        done <<< "$termux_dirs"
    else
        warn "No Termux directories found."
    fi

    printf "\n  ${BOLD}─── USB / FLASH DRIVE LOCATIONS ───${NC}\n\n"

    local usb_drives
    usb_drives=$(detect_usb_drives)
    if [ -n "$usb_drives" ]; then
        local i=1
        while IFS= read -r drive; do
            local fs_type
            fs_type=$(get_fs_type "$drive")
            local size_info
            size_info=$(df -h "$drive" 2>/dev/null | awk 'NR==2 {print $2" total, "$4" free"}' || echo "N/A")
            printf "  ${GREEN}[%d]${NC} ${BOLD}%s${NC}\n" "$i" "$drive"
            printf "      ${DIM}FS: %s | %s${NC}\n" "$fs_type" "$size_info"
            i=$((i+1))
        done <<< "$usb_drives"
    else
        warn "No USB drives detected."
        printf "  ${DIM}  • Ensure drive is plugged in via OTG${NC}\n"
        printf "  ${DIM}  • Format as FAT32/exFAT/NTFS${NC}\n"
        printf "  ${DIM}  • Grant Termux 'All files access'${NC}\n"
        printf "  ${DIM}  • Try: cat /proc/mounts | grep media_rw${NC}\n"
    fi

    if [ -d "$HOME/storage/external-1" ]; then
        printf "\n  ${BOLD}─── ADDITIONAL ───${NC}\n"
        printf "  ${CYAN}~/storage/external-1${NC} → $(readlink -f "$HOME/storage/external-1" 2>/dev/null || echo "?")\n"
    fi

    pause
}

# ─── Source Selection ─────────────────────────────────────────────────────────

select_source() {
    header
    printf "\n  ${BOLD}Select Source Directory${NC}\n\n"

    local termux_dirs
    termux_dirs=$(list_termux_dirs)
    local i=1
    local -a dir_list=()

    printf "  ${BOLD}%-50s${NC}\n" "Available Directories"
    printf "  %-50s\n" "──────────────────────────────────────────────────"

    while IFS=$'\t' read -r path size; do
        printf "  ${CYAN}[%d]${NC} %-46s ${DIM}(%s)${NC}\n" "$i" "$path" "$size"
        dir_list+=("$path")
        i=$((i+1))
    done <<< "$termux_dirs"

    printf "\n  ${BOLD}>${NC} Enter number (or full path): "
    read -r choice

    if [[ "$choice" =~ ^[0-9]+$ ]] && [ "$choice" -ge 1 ] && [ "$choice" -le "${#dir_list[@]}" ]; then
        SOURCE="${dir_list[$((choice-1))]}"
        success "Source: ${BOLD}$SOURCE${NC}"
    elif [ -d "$choice" ]; then
        SOURCE="$choice"
        success "Source: ${BOLD}$SOURCE${NC}"
    else
        error "Invalid selection."
    fi
    pause
}

browse_source() {
    header
    printf "\n  ${BOLD}Browse Source${NC}\n"
    printf "  ${DIM}Commands: ls [dir], cd [dir], back, done${NC}\n\n"

    local cwd="$HOME"
    while true; do
        printf "  ${BOLD}%s${NC} ${BOLD}>${NC} " "$cwd"
        read -r input

        case "$input" in
            back) cwd=$(dirname "$cwd"); [ "$cwd" = "/" ] && cwd="$HOME" ;;
            done) SOURCE="$cwd"; success "Source: $SOURCE"; return ;;
            ls*)
                local dir="${input#ls }"
                [ -z "$dir" ] && dir="$cwd"
                [ -d "$dir" ] || dir="$cwd/$dir"
                ls -lh "$dir" 2>/dev/null | sed 's/^/    /'
                ;;
            cd*)
                local dir="${input#cd }"
                if [ -d "$dir" ]; then cwd="$dir"
                elif [ -d "$cwd/$dir" ]; then cwd="$cwd/$dir"
                else warn "Not a directory"; fi
                ;;
            *)
                if [ -d "$input" ]; then cwd="$input"
                elif [ -d "$cwd/$input" ]; then cwd="$cwd/$input"
                else warn "Not found"; fi
                ;;
        esac
    done
}

# ─── Destination Selection ────────────────────────────────────────────────────

select_destination() {
    header
    printf "\n  ${BOLD}Select Destination (USB Drive)${NC}\n\n"

    local usb_drives
    usb_drives=$(detect_usb_drives)

    if [ -z "$usb_drives" ]; then
        error "No USB drives detected."
        printf "\n  ${YELLOW}Troubleshooting:${NC}\n"
        printf "  1. Drive plugged in via OTG?\n"
        printf "  2. Formatted as FAT32/exFAT/NTFS?\n"
        printf "  3. 'All files access' granted to Termux?\n"
        printf "  4. Try: termux-storage-get\n"
        printf "  5. Check: cat /proc/mounts | grep media_rw\n"

        # Try termux-storage-get as fallback
        if command -v termux-storage-get &>/dev/null; then
            printf "\n  ${BOLD}>${NC} Try termux-storage-get? (y/N): "
            read -r try
            if [ "$try" = "y" ] || [ "$try" = "Y" ]; then
                termux-storage-get
                usb_drives=$(detect_usb_drives)
                [ -d "$HOME/storage/external-1" ] && usb_drives="$usb_drives
$HOME/storage/external-1"
            fi
        fi

        [ -z "$usb_drives" ] && { pause; return; }
    fi

    local i=1
    local -a drive_list=()
    while IFS= read -r drive; do
        local fs_type
        fs_type=$(get_fs_type "$drive")
        local free
        free=$(df -h "$drive" 2>/dev/null | awk 'NR==2 {print $4}')
        printf "  ${GREEN}[%d]${NC} ${BOLD}%s${NC}  ${DIM}(%s, %s free)${NC}\n" "$i" "$drive" "$fs_type" "$free"
        drive_list+=("$drive")
        i=$((i+1))
    done <<< "$usb_drives"

    printf "\n  ${BOLD}>${NC} Enter number: "
    read -r choice

    if [[ "$choice" =~ ^[0-9]+$ ]] && [ "$choice" -ge 1 ] && [ "$choice" -le "${#drive_list[@]}" ]; then
        DEST="${drive_list[$((choice-1))]}"
        success "Destination: ${BOLD}$DEST${NC}"
    else
        error "Invalid selection."
    fi
    pause
}

# ─── Staging Fallback ─────────────────────────────────────────────────────────

stage_fallback() {
    if [ -z "$SOURCE" ]; then
        warn "No source selected. Selecting now..."
        select_source
    fi
    [ -z "$SOURCE" ] && return

    header
    printf "\n  ${BOLD}Stage to Internal Storage${NC}\n\n"
    printf "  ${DIM}Direct USB write unavailable. Files will be copied to${NC}\n"
    printf "  ${DIM}~/storage/downloads/usb_transfer/ — then use Files app to move to USB.${NC}\n\n"

    local stage="$HOME/storage/downloads/usb_transfer"
    mkdir -p "$stage"

    local basename_src
    basename_src=$(basename "$SOURCE")

    info "Copying to staging area..."
    cp -r "$SOURCE" "$stage/"
    success "Staged at: ${BOLD}$stage/$basename_src${NC}"

    log "INFO" "Staged: $SOURCE → $stage/$basename_src"

    printf "\n  ${YELLOW}Next steps:${NC}\n"
    printf "  1. Open your ${BOLD}Files app${NC}\n"
    printf "  2. Go to ${CYAN}Download/usb_transfer/$basename_src${NC}\n"
    printf "  3. Move to USB drive\n"
    printf "  4. Verify, then clean up:\n"
    printf "     ${DIM}rm -rf '$stage' && rm -rf '$SOURCE'${NC}\n"
    pause
}

# ─── Guards ───────────────────────────────────────────────────────────────────

require_source_dest() {
    if [ -z "${SOURCE:-}" ]; then
        header
        warn "No source selected."
        select_source
    fi
    if [ -z "${DEST:-}" ]; then
        header
        warn "No destination selected."
        select_destination
    fi
}

# ─── Entry Point ──────────────────────────────────────────────────────────────

parse_args "$@"
```

---

## `lib/utils.sh`

```bash
#!/data/data/com.termux/files/usr/bin/bash
#
# utils.sh — Colors, logging, and helper functions
#

# ─── Colors ───────────────────────────────────────────────────────────────────
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
DIM='\033[2m'
NC='\033[0m'

# ─── Display Helpers ──────────────────────────────────────────────────────────
clear_screen() { clear 2>/dev/null || printf '\033[2J\033[H'; }

header() {
    clear_screen
    printf "${BOLD}${CYAN}"
    cat <<'EOF'
  ╔══════════════════════════════════════════════════════════╗
  ║          EZ TERMUX → USB FLASH DRIVE TRANSFER           ║
  ╚══════════════════════════════════════════════════════════╝
EOF
    printf "${NC}"
}

info()    { printf "  ${BLUE}ℹ${NC} %s\n" "$1"; }
success() { printf "  ${GREEN}✓${NC} %s\n" "$1"; }
warn()    { printf "  ${YELLOW}⚠${NC} %s\n" "$1"; }
error()   { printf "  ${RED}✗${NC} %s\n" "$1"; }
pause()   { printf "\n  ${DIM}Press Enter to continue...${NC}"; read -r; }

# ─── Logging ──────────────────────────────────────────────────────────────────
log() {
    local level="$1"
    local msg="$2"
    local log_file="${LOG_FILE:-$HOME/.ez_usb_transfer.log}"
    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # Rotate log if too large
    local max_kb="${LOG_MAX_SIZE_KB:-5120}"
    if [ "$max_kb" -gt 0 ] && [ -f "$log_file" ]; then
        local size_kb
        size_kb=$(du -sk "$log_file" 2>/dev/null | cut -f1)
        if [ "${size_kb:-0}" -gt "$max_kb" ]; then
            mv "$log_file" "${log_file}.1"
        fi
    fi

    printf "%s [%s] %s\n" "$timestamp" "$level" "$msg" >> "$log_file"
}
```

---

## `lib/detect.sh`

```bash
#!/data/data/com.termux/files/usr/bin/bash
#
# detect.sh — USB drive and storage location detection
#

# Detect USB drives from /proc/mounts
# Returns newline-separated list of mount points
detect_usb_drives() {
    # Primary: /mnt/media_rw/<UUID>
    local drives
    drives=$(awk '($2 ~ "^/mnt/media_rw/[0-9A-Fa-f-]+$") && ($3 ~ /vfat|exfat|ntfs|fuseblk/) {print $2}' \
        /proc/mounts 2>/dev/null | sort -u)

    # Fallback: /storage/<UUID> (FUSE mount, better permissions on some devices)
    if [ -z "$drives" ]; then
        drives=$(awk '($2 ~ "^/storage/[0-9A-Fa-f-]+$") && ($3 ~ /vfat|exfat|ntfs|fuseblk/) {print $2}' \
            /proc/mounts 2>/dev/null | sort -u)
    fi

    # Also check ~/storage/external-1
    if [ -d "$HOME/storage/external-1" ]; then
        drives="${drives:+$drives
}$HOME/storage/external-1"
    fi

    printf "%s" "$drives"
}

# Get the first detected USB drive (for --dest auto)
detect_first_usb() {
    detect_usb_drives | head -1
}

# Get filesystem type for a mount point
get_fs_type() {
    local mount="$1"
    awk -v m="$mount" '$2==m {print $3}' /proc/mounts 2>/dev/null || echo "unknown"
}

# List key Termux directories with sizes
list_termux_dirs() {
    local dirs=(
        "$HOME"
        "$HOME/.termux"
        "$HOME/storage"
        "$HOME/storage/downloads"
        "$HOME/storage/shared"
        "$PREFIX"
        "$PREFIX/var"
    )
    for d in "${dirs[@]}"; do
        [ -d "$d" ] && printf "%s\t%s\n" "$d" "$(du -sh "$d" 2>/dev/null | cut -f1 || echo '?')"
    done
}

# Check if a path is writable
is_writable() {
    local path="$1"
    [ -d "$path" ] && [ -w "$path" ]
}
```

---

## `lib/transfer.sh`

```bash
#!/data/data/com.termux/files/usr/bin/bash
#
# transfer.sh — Core transfer logic
#

# Perform the actual file transfer
# Args: $1=source $2=dest $3=delete_flag (optional, "true"/"false")
perform_transfer() {
    local src="$1"
    local dest="$2"
    local delete_flag="${3:-false}"

    local basename_src
    basename_src=$(basename "$src")

    header
    printf "\n  ${BOLD}Transfer${NC}\n\n"
    printf "  From: ${CYAN}%s${NC}\n" "$src"
    printf "  To:   ${CYAN}%s${NC}\n" "$dest"

    # Verify source exists
    if [ ! -d "$src" ] && [ ! -f "$src" ]; then
        error "Source not found: $src"
        pause
        return 1
    fi

    # Verify destination is writable
    if ! is_writable "$dest"; then
        error "No write access to: $dest"
        printf "\n  ${YELLOW}Options:${NC}\n"
        printf "  • Grant 'All files access' to Termux (Settings → Apps → Termux)\n"
        printf "  • Run: termux-storage-get (select the USB drive)\n"
        printf "  • Use option 6 (stage to internal storage)\n"
        pause
        return 1
    fi

    # Determine filesystem and pick rsync flags
    local fs_type
    fs_type=$(get_fs_type "$dest")
    printf "  ${DIM}Filesystem: %s${NC}\n" "$fs_type"

    local rsync_flags
    if [ "$fs_type" = "ext4" ] || [ "$fs_type" = "f2fs" ]; then
        rsync_flags="${RSYNC_UNIX_FLAGS:--avz --progress}"
    else
        rsync_flags="${RSYNC_FAT_FLAGS:--vrt --progress --inplace --omit-dir-times --no-perms}"
    fi
    printf "  ${DIM}Flags: %s${NC}\n\n" "$rsync_flags"

    log "INFO" "Transfer: $src → $dest (fs=$fs_type)"

    # Execute transfer
    local exit_code=0
    if command -v rsync &>/dev/null; then
        info "Using rsync..."
        # shellcheck disable=SC2086
        rsync $rsync_flags "$src" "$dest/" 2>&1 | sed 's/^/    /' || exit_code=$?
    else
        warn "rsync not found. Using cp -r (install: pkg install rsync)"
        cp -r "$src" "$dest/" 2>&1 | sed 's/^/    /' || exit_code=$?
    fi

    printf "\n"

    if [ $exit_code -eq 0 ]; then
        success "Transfer complete!"
        log "INFO" "Transfer successful: $src → $dest"

        # Verification
        printf "\n  ${BOLD}Verification:${NC}\n"
        ls -lh "$dest/$basename_src" 2>/dev/null | head -20 | sed 's/^/    /'

        # Size comparison
        local src_size dst_size
        src_size=$(du -sh "$src" 2>/dev/null | cut -f1)
        dst_size=$(du -sh "$dest/$basename_src" 2>/dev/null | cut -f1)
        printf "\n  Source: ${CYAN}%s${NC} | Destination: ${CYAN}%s${NC}\n" "$src_size" "$dst_size"

        if [ "$src_size" != "$dst_size" ]; then
            warn "Size mismatch! Verify manually before deleting source."
            log "WARN" "Size mismatch: src=$src_size, dst=$dst_size"
        fi
    else
        error "Transfer failed (exit code: $exit_code)"
        log "ERROR" "Transfer failed (exit=$exit_code): $src → $dest"
        pause
        return 1
    fi

    # Delete source if requested
    if [ "$delete_flag" = "true" ]; then
        printf "\n  ${YELLOW}⚠  PERMANENTLY DELETE: $src${NC}\n"
        printf "  ${BOLD}>${NC} Type 'DELETE' to confirm: "
        read -r confirm
        if [ "$confirm" = "DELETE" ]; then
            rm -rf "$src"
            success "Source deleted."
            log "INFO" "Source deleted: $src"
        else
            info "Deletion cancelled. Source preserved."
        fi
    fi

    pause
}
```

---

## `scripts/setup.sh`

```bash
#!/data/data/com.termux/files/usr/bin/bash
#
# setup.sh — One-time setup for ez_usb_transfer
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BOLD='\033[1m'
NC='\033[0m'

echo ""
echo "  ${BOLD}╔══════════════════════════════════════════╗"
echo "  ║       EZ USB TRANSFER — SETUP          ║"
echo "  ╚══════════════════════════════════════════╝${NC}"
echo ""

# 1. Check for required packages
echo "  ${BOLD}[1/5]${NC} Checking dependencies..."
MISSING=()
for pkg in rsync coreutils; do
    if ! pkg list-installed 2>/dev/null | grep -q "^${pkg}@stable"; then
        MISSING+=("$pkg")
    fi
done

if [ ${#MISSING[@]} -gt 0 ]; then
    echo "  Installing: ${MISSING[*]}"
    pkg install -y "${MISSING[@]}"
else
    echo "  ${GREEN}✓${NC} All dependencies installed."
fi

# 2. Make scripts executable
echo ""
echo "  ${BOLD}[2/5]${NC} Setting permissions..."
chmod +x "$SCRIPT_DIR/ez_usb_transfer.sh"
chmod +x "$SCRIPT_DIR/scripts/"*.sh
chmod +x "$SCRIPT_DIR/lib/"*.sh
echo "  ${GREEN}✓${NC} Done."

# 3. Storage permission
echo ""
echo "  ${BOLD}[3/5]${NC} Setting up storage access..."
if [ -d "$HOME/storage" ]; then
    echo "  ${GREEN}✓${NC} Storage already configured."
else
    termux-setup-storage
    echo "  ${GREEN}✓${NC} Storage permission granted."
fi

# 4. Check for "All files access"
echo ""
echo "  ${BOLD}[4/5]${NC} Checking USB access..."
if [ -d "/mnt/media_rw" ] && [ "$(ls /mnt/media_rw 2>/dev/null | wc -l)" -gt 0 ]; then
    echo "  ${GREEN}✓${NC} USB media_rw directory accessible."
else
    echo "  ${YELLOW}⚠${NC} /mnt/media_rw is empty or inaccessible."
    echo "  ${YELLOW}   You may need to grant 'All files access':${NC}"
    echo "  ${YELLOW}   Settings → Apps → Termux → Special app access → All files access${NC}"
    echo ""
    echo "  ${YELLOW}   Without this, direct USB write may not work.${NC}"
    echo "  ${YELLOW}   The staging fallback (option 6) will still work.${NC}"
fi

# 5. Test USB detection
echo ""
echo "  ${BOLD}[5/5]${NC} Testing USB detection..."
source "$SCRIPT_DIR/lib/utils.sh"
source "$SCRIPT_DIR/lib/detect.sh"
USB=$(detect_usb_drives)
if [ -n "$USB" ]; then
    echo "  ${GREEN}✓${NC} USB drive(s) detected:"
    echo "$USB" | sed 's/^/      /'
else
    echo "  ${YELLOW}⚠${NC} No USB drive detected (normal if not plugged in)."
fi

echo ""
echo "  ${BOLD}Setup complete!${NC}"
echo ""
echo "  Run: ${GREEN}./ez_usb_transfer.sh${NC}"
echo ""
```

---

## `scripts/cleanup.sh`

```bash
#!/data/data/com.termux/files/usr/bin/bash
#
# cleanup.sh — Post-transfer cleanup
#

set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BOLD='\033[1m'
NC='\033[0m'

echo ""
echo "  ${BOLD}EZ USB Transfer — Cleanup${NC}"
echo ""

STAGE="$HOME/storage/downloads/usb_transfer"
LOG="$HOME/.ez_usb_transfer.log"
LOG_OLD="$HOME/.ez_usb_transfer.log.1"

# 1. Clean staging area
if [ -d "$STAGE" ]; then
    SIZE=$(du -sh "$STAGE" 2>/dev/null | cut -f1)
    echo "  Staging area: $STAGE (${SIZE})"
    echo "  Remove? (y/N): "
    read -r confirm
    if [ "$confirm" = "y" ] || [ "$confirm" = "Y" ]; then
        rm -rf "$STAGE"
        echo "  ${GREEN}✓${NC} Staging area removed."
    fi
else
    echo "  No staging area found."
fi

echo ""

# 2. Clean old logs
if [ -f "$LOG_OLD" ]; then
    echo "  Old log: $LOG_OLD"
    echo "  Remove? (y/N): "
    read -r confirm
    if [ "$confirm" = "y" ] || [ "$confirm" = "Y" ]; then
        rm -f "$LOG_OLD"
        echo "  ${GREEN}✓${NC} Old log removed."
    fi
fi

echo ""
echo "  ${BOLD}Cleanup complete.${NC}"
```

---

## `config/transfer.conf`

```bash
# ─────────────────────────────────────────────────────────────────────────────
# ez_usb_transfer configuration
# Copy this file and edit values as needed.
# ─────────────────────────────────────────────────────────────────────────────

# Default source directory (used if none specified)
DEFAULT_SOURCE="$HOME"

# Default destination (empty = auto-detect first USB drive)
DEFAULT_DEST=""

# rsync flags for FAT32/exFAT/NTFS (USB flash drives)
#   -vrt         : verbose, recursive, preserve times
#   --progress   : per-file progress bar
#   --inplace    : write directly (no temp files) — CRITICAL for FAT32
#   --omit-dir-times : skip dir timestamps (not supported on FAT)
#   --no-perms   : don't preserve Unix permissions (meaningless on FAT)
RSYNC_FAT_FLAGS="-vrt --progress --inplace --omit-dir-times --no-perms"

# rsync flags for ext4/f2fs (internal storage or Linux-native USB)
RSYNC_UNIX_FLAGS="-avz --progress"

# Log file location
LOG_FILE="$HOME/.ez_usb_transfer.log"

# Max log file size in KB before rotation (0 = unlimited)
LOG_MAX_SIZE_KB=5120

# Number of log files to keep (1 = current + 1 old)
LOG_ROTATE_COUNT=1
```

---

## `docs/TROUBLESHOOTING.md`

```markdown
# Troubleshooting

## "No USB drives detected"

| Cause | Fix |
|-------|-----|
| Drive not plugged in | Connect via OTG adapter |
| Drive formatted as MTP-only | Reformat as FAT32, exFAT, or NTFS |
| Android 13+ restriction | Grant "All files access" or use `termux-storage-get` |
| SELinux blocks access | Try the `/storage/<UUID>` path instead of `/mnt/media_rw/<UUID>` |

**Diagnose:**
```bash
cat /proc/mounts | grep -E "media_rw|/storage/[0-9A-Fa-f]"
ls -la /mnt/media_rw/
termux-usb -l
```

## "Permission denied" on write

- **Android 12+:** Grant "All files access" → Settings → Apps → Termux → Special app access
- **Still failing:** Try `termux-storage-get` and select the USB drive root
- **Samsung devices:** The `/storage/<UUID>` path often works when `/mnt/media_rw/<UUID>` doesn't

## rsync: "mkstemp failed"

FAT32/exFAT can't handle rsync's default temp-file strategy.

**Fix:** Ensure `--inplace` is in your rsync flags (it's the default in `transfer.conf`).

## rsync: "chmod: Operation not permitted"

FAT32 has no Unix permissions.

**Fix:** Ensure `--no-perms` is in your rsync flags (it's the default).

## Files are 0 bytes on destination

- Drive was unmounted mid-transfer
- exFAT journal corruption
- **Fix:** Safely unmount in Android before unplugging. Reformat if corruption persists.

## `~/storage/external-1` exists but is empty

This symlink points to Termux's **private** data directory on removable storage — not the drive root. Don't use it as a general USB path.

## Drive visible in Files app but not in `/proc/mounts`

Android 14/15 may expose drives only via MTP. Options:
1. Reformat as FAT32/exFAT (forces kernel mount)
2. Use `termux-storage-get` to grant scoped access
3. Use the staging fallback (option 6)

## Transfer works but is very slow

- Use a USB 3.0 OTG adapter (USB 2.0 caps at ~30 MB/s)
- Avoid NTFS (slower than exFAT on Android)
- Large numbers of small files are inherently slow; consider `tar` first:
  ```bash
  tar czf ~/storage/downloads/backup.tar.gz -C ~/ my_project
  ```
  Then transfer the single archive.
```

---

## Git Init & Push

```bash
# Initialize repo
mkdir -p ez_termux_usb_file_transer/{lib,scripts,config,docs}
cd ez_termux_usb_file_transer

# Create all files (paste content from above)
# ...

# Make executable
chmod +x ez_usb_transfer.sh scripts/*.sh lib/*.sh

# Git
git init
git add .
git commit -m "Initial release: EZ Termux → USB transfer tool"

# Push
git remote add origin https://github.com/bilbywilby/ez_termux_usb_file_transer.git
git push -u origin main
```

---

