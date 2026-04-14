# Optimize My MacBook for Speed

A Claude Code skill that audits and optimizes your Mac's performance. Born from a real debugging session where a single Chrome tab was eating 15 GB of RAM.

## What it does

1. **System Snapshot** — audits memory, swap, processes, DNS, and startup items
2. **Kills Runaway Processes** — safely terminates Chrome tabs leaking memory (>6 GB)
3. **DNS Optimization** — switches from slow ISP DNS to Cloudflare 1.1.1.1
4. **Animation Tweaks** — disables 7 macOS animations for instant UI response
5. **Memory Watchdog** — installs a persistent background monitor that auto-kills leaked Chrome tabs and warns about memory hogs
6. **Login Item Audit** — identifies unnecessary startup apps consuming resources
7. **Full Report** — presents before/after metrics with reversal instructions

## Safety

- Never kills system processes or non-Chrome apps without asking
- All changes are reversible with documented undo commands
- Never uses `memory_pressure` (a stress-testing tool that makes things worse)
- Lets macOS manage its own VM subsystem — only intervenes for clear runaways

## Install

```bash
git clone https://github.com/Casper0301/optimize-mac-skill.git ~/.claude/skills/optimize-mac
```

Then restart Claude Code and say "optimize my Mac" or run `/optimize-mac`.

## Requirements

- macOS (any version with Apple Silicon or Intel)
- Claude Code with Max subscription

## What you get

- Full system performance audit
- Automatic Chrome memory leak cleanup
- Persistent watchdog (runs every 5 minutes via launchd)
- DNS optimization (Cloudflare 1.1.1.1)
- 7 animation performance tweaks
- Startup item audit
- Before/after report with undo instructions

## License

MIT
