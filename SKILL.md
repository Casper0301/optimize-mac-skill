---
name: optimize-mac
description: >
  Optimize MacBook performance by auditing and fixing memory hogs, runaway processes,
  swap pressure, DNS, animations, startup items, and system settings. Installs a
  persistent memory watchdog that auto-kills runaway Chrome tabs. Safe, reversible,
  non-destructive. Use when user says "optimize my Mac", "Mac is slow", "speed up Mac",
  "performance issues", "memory issues", "Mac running slow", "fix my Mac speed",
  "system optimization", or "optimize MacBook".
---

# Optimize My MacBook for Speed

Full-system performance audit and optimization for macOS. Diagnoses memory hogs,
swap pressure, animation overhead, DNS latency, and runaway processes — then fixes
what it safely can and reports on the rest.

## Safety Rules

- **NEVER run `memory_pressure`** — it allocates massive memory to stress-test the system
  and will spike swap usage dramatically. It is a developer stress-testing tool, not a
  cleanup tool.
- **NEVER run `sudo purge`** as a performance fix — it flushes disk cache and forces
  re-reads from disk, making the system temporarily slower.
- **NEVER kill system processes** — Finder, Dock, WindowServer, loginwindow,
  SystemUIServer, kernel_task, launchd, and all processes owned by root are off-limits.
- **NEVER kill non-Chrome user apps** without explicit user permission.
- **Only kill Chrome Helper (Renderer) processes** that exceed the memory threshold —
  these are individual tabs, not the browser itself.
- **All changes must be reversible.** Document what was changed so the user can undo it.
- Let macOS manage its own memory. Don't try to outsmart the VM subsystem.

## Execution Flow

Run these phases in order. Report findings after each phase before proceeding.

### Phase 1: System Snapshot

Gather the current state. Run these commands and present findings in a table:

```bash
# Total RAM
sysctl -n hw.memsize | awk '{printf "%.0f GB", $1/1024/1024/1024}'

# Top 15 processes by memory
top -l 1 -o mem -n 15 -stats pid,mem,cpu,command

# Memory breakdown
vm_stat

# Swap usage
sysctl vm.swapusage

# Process count
ps aux | wc -l

# Chrome tab count
ps aux | grep "Chrome Helper (Renderer)" | grep -v grep | wc -l

# Login items
osascript -e 'tell application "System Events" to get the name of every login item'

# DNS servers
networksetup -getdnsservers Wi-Fi

# Disk space
df -h /
```

Present a summary table:
| Metric | Value | Status |
|--------|-------|--------|
| Total RAM | X GB | — |
| Active memory demand | X GB | OK/High |
| Swap usage | X GB | OK/Warning/Critical |
| Chrome tabs | X | OK/High |
| Process count | X | OK/High |
| DNS | ISP/Cloudflare/Google | OK/Slow |

### Phase 2: Kill Runaway Processes

Check for any process using excessive memory:

```bash
# Find processes using more than 4GB
top -l 1 -o mem -n 20 -stats pid,mem,command
```

- **Chrome Helper (Renderer) > 6 GB**: Kill immediately (these are leaked tabs)
- **Any other process > 4 GB**: Report to user, ask before killing
- **Never kill**: System processes, Claude, Wispr Flow, or anything the user has flagged as important

### Phase 3: DNS Optimization

Check current DNS and switch to Cloudflare if using ISP defaults:

```bash
# Check current DNS
networksetup -getdnsservers Wi-Fi

# If ISP DNS, switch to Cloudflare + Google fallback
networksetup -setdnsservers Wi-Fi 1.1.1.1 1.0.0.1 8.8.8.8 8.8.4.4

# Flush DNS cache
dscacheutil -flushcache
```

Skip if already using Cloudflare or Google DNS.

### Phase 4: Animation & Visual Tweaks

Check and apply these performance settings (all reversible):

```bash
# Reduce Motion (via System Settings > Accessibility > Motion)
# Reduce Transparency (via System Settings > Accessibility > Display)
# These require System Settings UI or computer-use — cannot be set via defaults write

# These CAN be set via defaults:
defaults write NSGlobalDomain NSAutomaticWindowAnimationsEnabled -bool false
defaults write com.apple.finder DisableAllAnimations -bool true
defaults write com.apple.dock launchanim -bool false
defaults write com.apple.dock expose-animation-duration -float 0.1
defaults write com.apple.dock autohide-delay -float 0
defaults write com.apple.dock autohide-time-modifier -float 0.3
defaults write NSGlobalDomain NSWindowResizeTime -float 0.1

# Restart Dock and Finder to apply
killall Dock
killall Finder
```

For Reduce Motion and Reduce Transparency: use computer-use MCP to toggle them in
System Settings > Accessibility > Motion and Display, since `defaults write
com.apple.universalaccess` requires elevated permissions.

### Phase 5: Install Memory Watchdog

Check if the watchdog is already installed:

```bash
launchctl print gui/$(id -u)/com.casper.memory-watchdog 2>/dev/null
```

If not installed, create two files:

**Script** at `~/Scripts/memory-watchdog.sh`:
- Runs every 5 minutes via launchd
- Kills Chrome Helper (Renderer) processes using > 6 GB RAM
- Sends macOS notification when it kills a process
- Warns (notification only) about non-Chrome processes > 4 GB
- Protects system processes (Finder, Dock, WindowServer, Claude, etc.)
- Logs all actions to `~/Scripts/logs/memory-watchdog.log`
- Auto-trims log file when it exceeds 1000 lines
- **NEVER uses `memory_pressure` command**

**LaunchAgent** at `~/Library/LaunchAgents/com.casper.memory-watchdog.plist`:
- StartInterval: 300 (every 5 minutes)
- RunAtLoad: true
- Logs stdout/stderr to `~/Scripts/logs/`

Load with:
```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.casper.memory-watchdog.plist
```

### Phase 6: Audit Login Items

List all login items and launch agents. Flag anything that:
- Uses > 500 MB RAM
- Is an updater/helper that doesn't need to run constantly
- Is a duplicate (e.g., multiple Adobe updaters)

Present findings and ask the user which ones to remove. Never remove without asking.

### Phase 7: Summary Report

Present a final report:

```
OPTIMIZATION REPORT
===================
[x] Killed X runaway processes (reclaimed ~X GB)
[x] DNS switched to Cloudflare 1.1.1.1
[x] Animations disabled (7 settings)
[x] Reduce Motion enabled
[x] Reduce Transparency enabled
[x] Memory watchdog installed (runs every 5 min)
[ ] Login items reviewed (X flagged for user review)

Memory: X GB active / X GB total
Swap: X GB (was X GB)
Chrome tabs: X

RECOMMENDATIONS:
- Enable Chrome Memory Saver at chrome://settings/performance
- Restart [app] periodically (memory leak detected: X GB)
- Consider closing X+ Chrome tabs to reduce baseline memory
```

## Reversal Instructions

If the user wants to undo any changes:

```bash
# Restore ISP DNS
networksetup -setdnsservers Wi-Fi empty

# Re-enable animations
defaults delete NSGlobalDomain NSAutomaticWindowAnimationsEnabled
defaults delete com.apple.finder DisableAllAnimations
defaults delete com.apple.dock launchanim
defaults delete com.apple.dock expose-animation-duration
defaults delete com.apple.dock autohide-delay
defaults delete com.apple.dock autohide-time-modifier
defaults delete NSGlobalDomain NSWindowResizeTime
killall Dock; killall Finder

# Remove watchdog
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.casper.memory-watchdog.plist
rm ~/Library/LaunchAgents/com.casper.memory-watchdog.plist
rm ~/Scripts/memory-watchdog.sh

# Reduce Motion / Transparency: toggle off in System Settings > Accessibility
```
