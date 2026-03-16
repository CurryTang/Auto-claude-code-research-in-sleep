---
name: gpu-status
description: Show GPU availability across all saved SSH servers. Use when user says "check GPUs", "which GPUs are free", "gpu status", "GPU 状态", or needs to know where to run experiments.
argument-hint: [optional: specific server alias]
allowed-tools: Bash(ssh *), Bash(echo *), Bash(cat *), Read, Grep, Glob
---

# GPU Status

Check GPU availability: $ARGUMENTS

## Workflow

### Step 1: Discover Servers

Read the project's `CLAUDE.md` (and any `AGENTS.md`) to find all configured SSH servers.
Look for patterns like:
- `ssh <alias>` or `SSH: ssh <alias>`
- Server sections with GPU info
- Remote server blocks

Also check `~/.ssh/config` for known hosts if no servers are found in project files:
```bash
cat ~/.ssh/config | grep -i "^Host " | grep -v "\*"
```

If `$ARGUMENTS` specifies a server, check only that server.

### Step 2: Query Each Server

For each discovered server, run `nvidia-smi` via SSH:

```bash
ssh <server> "nvidia-smi --query-gpu=index,name,memory.used,memory.total,utilization.gpu,temperature.gpu --format=csv,noheader" 2>&1
```

If `nvidia-smi` is not found, try:
```bash
ssh <server> "/usr/bin/nvidia-smi --query-gpu=index,name,memory.used,memory.total,utilization.gpu,temperature.gpu --format=csv,noheader" 2>&1
```

If the server is unreachable or has no GPUs, note it and move on.

### Step 3: Determine Availability

A GPU is considered **free** if:
- `memory.used < 500 MiB` — completely idle
- `memory.used < 2000 MiB` AND `utilization.gpu < 10%` — likely idle (small framework overhead)

A GPU is **partially available** if:
- Used memory is less than 50% of total memory

A GPU is **busy** if:
- Used memory exceeds 50% of total memory

### Step 4: Present Results

Display a summary table per server:

```
## <server-alias>  (e.g. my-gpu-server)

| GPU | Model       | VRAM Used   | VRAM Total | Util % | Temp | Status    |
|-----|-------------|-------------|------------|--------|------|-----------|
| 0   | A100-80GB   | 234 MiB     | 81920 MiB  | 0%     | 32C  | Free      |
| 1   | A100-80GB   | 45321 MiB   | 81920 MiB  | 87%    | 71C  | Busy      |
| 2   | A100-80GB   | 1200 MiB    | 81920 MiB  | 0%     | 35C  | Free      |
| 3   | A100-80GB   | 40000 MiB   | 81920 MiB  | 45%    | 58C  | Partial   |
```

Then a quick summary:
```
### Summary
- my-gpu-server: 2 free, 1 partial, 1 busy (out of 4 GPUs)
- lab-server:    8 free, 0 partial, 0 busy (out of 8 GPUs)
Total free GPUs across all servers: 10
```

### Step 5: Recommend

Based on availability:
- If user mentioned a specific workload, suggest which server + GPU(s) to use
- Prefer servers with the most contiguous free GPUs for multi-GPU jobs
- Prefer GPUs with lower temperature (cooler = less thermal throttling risk)
- Note any servers that were unreachable

## Key Rules

- Never modify anything on the servers — this skill is read-only
- If SSH times out after 10 seconds, mark server as unreachable and continue
- Run SSH commands with a timeout: `ssh -o ConnectTimeout=10 <server> ...`
- Query all servers in parallel when possible (use multiple tool calls)
- If no servers are configured anywhere, tell the user to add server info to their CLAUDE.md
