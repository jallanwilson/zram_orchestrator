#!/bin/sh
# ==============================================================================
#  PROTOCOL: WILSON | ZRAM ORCHESTRATOR
#  CODENAME: "Neural Scrambler"
#  VERSION:  5.0 (Adaptive + Anti-Forensic)
#  AUTHOR:   Skynet2029
# ==============================================================================
#
#  LICENSE: MIT License
#
#  Copyright (c) 2026 Skynet2029
#
#  Permission is hereby granted, free of charge, to any person obtaining a copy
#  of this software and associated documentation files (the "Software"), to deal
#  in the Software without restriction, including without limitation the rights
#  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
#  copies of the Software, and to permit persons to whom the Software is
#  furnished to do so, subject to the following conditions:
#
#  The above copyright notice and this permission notice shall be included in all
#  copies or substantial portions of the Software.
#
#  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
#  AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#  LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
#  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
#  SOFTWARE.
# ==============================================================================

# --- CONFIGURATION ---
BASE_MOUNT="/mnt/zram_target"

# Portable Colors
if [ -t 1 ]; then
    RED='\033[0;31m'; GREEN='\033[0;32m'; BLUE='\033[0;34m'; YELLOW='\033[1;33m'; NC='\033[0m'
else
    RED=''; GREEN=''; BLUE=''; YELLOW=''; NC=''
fi

# Root Check
if [ "$(id -u)" -ne 0 ]; then echo "${RED}[!] FATAL: Run as root.${NC}"; exit 1; fi
if ! command -v cryptsetup >/dev/null 2>&1; then echo "${RED}[!] MISSING: cryptsetup${NC}"; exit 1; fi

# --- HELPERS ---
show_header() {
    clear
    echo "${GREEN}"
    echo "==================================================="
    echo "   PROTOCOL: WILSON | ZRAM ORCHESTRATOR (v5.0)     "
    echo "==================================================="
    echo "${NC}"
}

get_input() {
    printf "    > %s: " "$1"
    read $2
}

list_targets() {
    echo "---------------------------------------------------"
    echo "ACTIVE INFRASTRUCTURE:"
    lsblk -o NAME,SIZE,TYPE,MOUNTPOINT | grep -E "zram|crypt"
    echo "---------------------------------------------------"
}

spawn_device() {
    # Kernel Hot-Add Force Fabrication
    if [ -f "/sys/class/zram-control/hot_add" ]; then
        ID=$(cat /sys/class/zram-control/hot_add)
        echo "/dev/zram${ID}"
    else
        zramctl --find
    fi
}

# --- TUNING ENGINE (Adaptive RAM Logic) ---
configure_zram_device() {
    DEV_NAME=$1
    DEV_BASE=${DEV_NAME#"/dev/"}

    if [ ! -d "/sys/block/$DEV_BASE" ]; then sleep 1; fi

    # [STRICT COMPLIANCE CHECK]
    # Calculate Safe RAM usage (50% of Physical)
    if [ -r /proc/meminfo ]; then
        # Capture Total Mem in kB
        TOTAL_MEM_KB=$(awk '/MemTotal/ {print $2}' /proc/meminfo)

        # POSIX Arithmetic Expansion to convert to GB (integer math)
        TOTAL_MEM_GB=$((TOTAL_MEM_KB / 1024 / 1024))

        # Safety Logic: 50% of Total RAM
        SAFE_SIZE_GB=$((TOTAL_MEM_GB / 2))

        # Fallback for low-memory environments (< 2GB RAM results in 0GB)
        if [ "$SAFE_SIZE_GB" -eq 0 ]; then
            SAFE_DEFAULT="512M"
        else
            SAFE_DEFAULT="${SAFE_SIZE_GB}G"
        fi

        echo "${YELLOW}[i] HOST KERNEL REPORTS: ${TOTAL_MEM_GB} GB Physical RAM.${NC}"
    else
        # Fallback if /proc is not mounted (The Void scenario)
        SAFE_DEFAULT="2G"
        echo "${RED}[!] WARNING: /proc/meminfo unavailable. Defaulting to ${SAFE_DEFAULT}.${NC}"
    fi

    echo "---------------------------------------------------"

    # Prompt for Size
    get_input "Target Size (Default: $SAFE_DEFAULT)" INPUT_SIZE
    TARGET_SIZE=${INPUT_SIZE:-$SAFE_DEFAULT}

    echo "${YELLOW}[i] KERNEL CAPABILITIES:${NC}"
    if [ -f "/sys/block/$DEV_BASE/comp_algorithm" ]; then
        cat "/sys/block/$DEV_BASE/comp_algorithm"
    fi

    echo "---------------------------------------------------"
    get_input "Compression Algo (default: zstd)" INPUT_ALGO
    ALGO=${INPUT_ALGO:-zstd}

    get_input "Compression Level (Enter to skip)" INPUT_LEVEL
    get_input "Path to Pre-trained Dictionary (Enter to skip)" INPUT_DICT

    echo "$ALGO" > "/sys/block/$DEV_BASE/comp_algorithm" 2>/dev/null

    PARAM_STR="algo=$ALGO"
    if [ -n "$INPUT_LEVEL" ]; then PARAM_STR="$PARAM_STR level=$INPUT_LEVEL"; fi

    if [ -n "$INPUT_DICT" ]; then
        if [ -f "$INPUT_DICT" ]; then
            PARAM_STR="$PARAM_STR dict=$INPUT_DICT"
            echo "[+] Dictionary Loaded: $INPUT_DICT"
        else
            echo "${RED}[!] Dictionary file not found. Skipping.${NC}"
        fi
    fi

    if [ -n "$INPUT_LEVEL" ] || [ -n "$INPUT_DICT" ]; then
        echo "[+] Injecting Params: $PARAM_STR"
        if ! echo "$PARAM_STR" > "/sys/block/$DEV_BASE/algorithm_params"; then
             echo "${RED}[!] Parameter Injection Failed. (Kernel incompatible?)${NC}"
        fi
    fi

    echo "$TARGET_SIZE" > "/sys/block/$DEV_BASE/disksize"
}

# --- MODULES ---

module_create() {
    echo "[+] Initializing Creation Protocol..."
    if ! grep -q zram /proc/modules; then modprobe zram; fi

    ZRAM_DEV=$(spawn_device)
    if [ -z "$ZRAM_DEV" ]; then echo "${RED}[!] Spawn failed.${NC}"; return; fi

    configure_zram_device "$ZRAM_DEV"

    DEV_NUM=$(echo "$ZRAM_DEV" | grep -o '[0-9]*$')
    MOUNT_POINT="${BASE_MOUNT}_${DEV_NUM}"
    if [ ! -d "$MOUNT_POINT" ]; then mkdir -p "$MOUNT_POINT"; fi

    echo "[+] Formatting (ext4 optimized)..."
    mkfs.ext4 -O ^has_journal -E discard -m 0 "$ZRAM_DEV" > /dev/null
    mount -o discard,noatime,nodiratime "$ZRAM_DEV" "$MOUNT_POINT"
    chmod 1777 "$MOUNT_POINT"

    echo "${GREEN}[+] READY: $ZRAM_DEV -> $MOUNT_POINT${NC}"
    get_input "Press Enter" DUMMY
}

module_secure() {
    echo "${BLUE}[+] BLACK SITE PROTOCOL (Encrypted + Tuned)${NC}"
    if ! grep -q zram /proc/modules; then modprobe zram; fi

    ZRAM_DEV=$(spawn_device)
    if [ -z "$ZRAM_DEV" ]; then echo "${RED}[!] Spawn failed.${NC}"; return; fi

    configure_zram_device "$ZRAM_DEV"

    DEV_NUM=$(echo "$ZRAM_DEV" | grep -o '[0-9]*$')
    MAPPER_NAME="wilson_crypt_${DEV_NUM}"
    MOUNT_POINT="${BASE_MOUNT}_secure_${DEV_NUM}"

    echo "[+] Generating Ephemeral Key (PRNG)..."
    head -c 64 /dev/urandom | cryptsetup open --type plain "$ZRAM_DEV" "$MAPPER_NAME" --key-file - --hash sha256 --key-size 512

    echo "[+] Formatting Encrypted Container..."
    mkfs.ext4 -O ^has_journal -E discard -m 0 "/dev/mapper/$MAPPER_NAME" > /dev/null

    if [ ! -d "$MOUNT_POINT" ]; then mkdir -p "$MOUNT_POINT"; fi
    mount -o discard,noatime,nodiratime "/dev/mapper/$MAPPER_NAME" "$MOUNT_POINT"
    chmod 1777 "$MOUNT_POINT"

    echo "${BLUE}[+] SECURE SITE: $ZRAM_DEV -> ENCRYPT -> $MOUNT_POINT${NC}"
    get_input "Press Enter" DUMMY
}

module_colonize() {
    list_targets
    get_input "Enter Target ID (e.g., 3)" TARGET_ID
    ID=$(echo "$TARGET_ID" | grep -o '[0-9]*$')

    MAPPER_NAME="wilson_crypt_${ID}"
    MOUNT_POINT=$(lsblk -n -o MOUNTPOINT "/dev/mapper/$MAPPER_NAME" 2>/dev/null)
    if [ -z "$MOUNT_POINT" ]; then MOUNT_POINT=$(lsblk -n -o MOUNTPOINT "/dev/zram${ID}" 2>/dev/null); fi
    if [ -z "$MOUNT_POINT" ]; then echo "${RED}[!] Cluster $ID is not mounted.${NC}"; return; fi

    echo "[+] INITIALIZING THE VOID: $MOUNT_POINT"

    # Infrastructure
    mkdir -p "$MOUNT_POINT/bin" "$MOUNT_POINT/sbin" "$MOUNT_POINT/lib" "$MOUNT_POINT/lib64"
    mkdir -p "$MOUNT_POINT/usr/bin" "$MOUNT_POINT/usr/sbin" "$MOUNT_POINT/usr/include"
    mkdir -p "$MOUNT_POINT/proc" "$MOUNT_POINT/sys" "$MOUNT_POINT/tmp" "$MOUNT_POINT/etc" "$MOUNT_POINT/workspace"

    # Parasitic Binds
    mount --bind /bin "$MOUNT_POINT/bin"
    mount --bind /sbin "$MOUNT_POINT/sbin"
    mount --bind /lib "$MOUNT_POINT/lib"
    mount --bind /lib64 "$MOUNT_POINT/lib64"
    mount --bind /usr/bin "$MOUNT_POINT/usr/bin"
    mount --bind /usr/sbin "$MOUNT_POINT/usr/sbin"
    mount --bind /usr/include "$MOUNT_POINT/usr/include"

    # Sensory Injection
    if [ -f /etc/resolv.conf ]; then touch "$MOUNT_POINT/etc/resolv.conf"; mount --bind /etc/resolv.conf "$MOUNT_POINT/etc/resolv.conf"; fi
    if [ -d /etc/ssl ]; then mkdir -p "$MOUNT_POINT/etc/ssl"; mount --bind /etc/ssl "$MOUNT_POINT/etc/ssl"; fi
    mount -t sysfs sysfs "$MOUNT_POINT/sys"

    # Config
    RC_FILE="$MOUNT_POINT/.wilson_rc"
    cat <<EOF > "$RC_FILE"
export PS1="\[\033[1;31m\][VOID:zram${ID}]\[\033[0m\] \w > "
export PATH=/bin:/usr/bin:/sbin:/usr/sbin
alias ll='ls -l'
echo "---------------------------------------------------"
echo " YOU ARE NOW PID 1. HOST PROCESSES ARE INVISIBLE."
echo " NETWORK: RESOLVED. COMPILER: READY."
echo "---------------------------------------------------"
EOF

    # Isolation
    echo "${RED}[+] SEVERING HOST TETHER...${NC}"
    if command -v unshare >/dev/null 2>&1; then
        unshare -m -p -f --mount-proc="$MOUNT_POINT/proc" chroot "$MOUNT_POINT" /bin/bash --rcfile /.wilson_rc
    else
        chroot "$MOUNT_POINT" /bin/bash --rcfile /.wilson_rc
    fi

    # Cleanup
    echo "${YELLOW}[+] CONNECTION SEVERED. CLEANING BINDS...${NC}"
    umount "$MOUNT_POINT/proc" 2>/dev/null
    umount "$MOUNT_POINT/sys" 2>/dev/null
    umount "$MOUNT_POINT/etc/resolv.conf" 2>/dev/null
    umount "$MOUNT_POINT/etc/ssl" 2>/dev/null
    umount "$MOUNT_POINT/usr/include" 2>/dev/null
    umount "$MOUNT_POINT/usr/sbin" 2>/dev/null
    umount "$MOUNT_POINT/usr/bin" 2>/dev/null
    umount "$MOUNT_POINT/lib64" 2>/dev/null
    umount "$MOUNT_POINT/lib" 2>/dev/null
    umount "$MOUNT_POINT/sbin" 2>/dev/null
    umount "$MOUNT_POINT/bin" 2>/dev/null
    get_input "Press Enter" DUMMY
}

# --- MODULE KILL (ANTI-FORENSIC) ---
module_kill() {
    list_targets
    echo "${RED}[!] WARNING: zram0 is usually SWAP.${NC}"
    get_input "Enter Target ID (e.g., 3)" TARGET_ID
    ID=$(echo "$TARGET_ID" | grep -o '[0-9]*$')
    if [ -z "$ID" ]; then return; fi

    ZRAM_DEV="/dev/zram${ID}"
    MAPPER_NAME="wilson_crypt_${ID}"

    MOUNT_A=$(lsblk -n -o MOUNTPOINT "$ZRAM_DEV" 2>/dev/null)
    MOUNT_B=$(lsblk -n -o MOUNTPOINT "/dev/mapper/$MAPPER_NAME" 2>/dev/null)

    if [ -n "$MOUNT_B" ]; then umount "$MOUNT_B"; rmdir "$MOUNT_B" 2>/dev/null; fi
    if [ -n "$MOUNT_A" ]; then umount "$MOUNT_A"; rmdir "$MOUNT_A" 2>/dev/null; fi

    # [NEURAL PATHWAY SCRAMBLER]
    if [ -b "/dev/mapper/$MAPPER_NAME" ]; then
        echo "${RED}[!] SCRAMBLING NEURAL PATHWAYS...${NC}"
        # One pass of random data to the OPEN encrypted container
        # This destroys the plaintext structure inside before the key is dropped.
        dd if=/dev/urandom of="/dev/mapper/$MAPPER_NAME" bs=1M count=10 >/dev/null 2>&1

        echo "[+] Closing Encrypted Gate..."
        cryptsetup close "$MAPPER_NAME"
    fi

    if [ -b "$ZRAM_DEV" ]; then
        IS_SWAP=$(lsblk -no FSTYPE "$ZRAM_DEV" | grep "swap")
        if [ -n "$IS_SWAP" ]; then
            echo "${RED}[!] ABORT: Is Swap.${NC}"
        else
            if echo "$ID" > /sys/class/zram-control/hot_remove 2>/dev/null; then
                echo "${GREEN}[+] DEVICE DESTROYED (Hot Removed).${NC}"
            else
                zramctl --reset "$ZRAM_DEV"
                echo "${GREEN}[+] DEVICE RESET.${NC}"
            fi
        fi
    fi
    get_input "Press Enter" DUMMY
}

module_bench() {
    list_targets
    get_input "Enter Mount Point" TARGET_MOUNT
    if [ ! -d "$TARGET_MOUNT" ]; then echo "${RED}[!] Invalid.${NC}"; return; fi
    if command -v fio >/dev/null 2>&1; then
        echo "[+] STRESS TEST..."
        fio --name=wilson_stress --filename="$TARGET_MOUNT/stress_test" --ioengine=libaio --rw=randwrite --bs=4k --size=256M --numjobs=4 --iodepth=16 --group_reporting --direct=1
        rm "$TARGET_MOUNT/stress_test"
    fi
    get_input "Press Enter" DUMMY
}

module_watch() {
    list_targets
    get_input "Enter ZRAM ID (e.g., 3)" TARGET_ID
    ID=$(echo "$TARGET_ID" | grep -o '[0-9]*$')
    STATS_PATH="/sys/block/zram${ID}/mm_stat"
    if [ ! -f "$STATS_PATH" ]; then echo "${RED}[!] Invalid ID${NC}"; return; fi
    trap 'return' INT
    while true; do
        read ORIG COMP REST < "$STATS_PATH"
        if [ "$COMP" -eq 0 ]; then R="0.00"; else R=$(awk "BEGIN {printf \"%.2f\", $ORIG / $COMP}"); fi
        printf "RATIO (zram%s): %s\r" "$ID" "$R"
        sleep 1
    done
    trap - INT
}

# --- MAIN MENU ---
while true; do
    show_header
    echo "1. [CREATE]  Standard Reactor (Hotplug + Dict)"
    echo "2. [SECURE]  Black Site (Hotplug + Encrypt)"
    echo "3. [COLONIZE] The Void (Namespace Isolation)"
    echo "4. [STRESS]  Benchmark I/O"
    echo "5. [WATCH]   Monitor Compression"
    echo "6. [PURGE]   Neutralize Target"
    echo "7. [EXIT]    Disconnect"
    echo "---------------------------------------------------"
    get_input "Select Protocol [1-7]" OPTION

    case $OPTION in
        1) module_create ;;
        2) module_secure ;;
        3) module_colonize ;;
        4) module_bench ;;
        5) module_watch ;;
        6) module_kill ;;
        7) echo "[+] Session Terminated."; exit 0 ;;
        *) sleep 1 ;;
    esac
done
