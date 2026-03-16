---
name: gpu-status
description: Show GPU availability across all saved SSH servers for the current ARIS project. Use when user says "check GPUs", "which GPUs are free", "gpu status", "GPU 状态", or needs to know where to run experiments.
argument-hint: [optional: project name or server name to filter]
allowed-tools: Bash(ssh *), Bash(echo *), Bash(curl *), Read, Grep, Glob
---

# GPU Status

Check GPU availability: $ARGUMENTS

## Workflow

### Step 1: Discover Servers from ARIS Project Targets

Each ARIS project has saved SSH server targets via the Auto Researcher API. Query the API to get all projects and their targets.

**List all ARIS projects:**
```bash
curl -s http://localhost:3000/api/aris/projects | python3 -c "
import sys, json
data = json.load(sys.stdin)
for p in data.get('projects', []):
    print(f\"{p['id']}  {p['name']}\")
"
```

If `$ARGUMENTS` names a specific project, filter to that project only.

**Get targets for each project (or the specified one):**
```bash
curl -s http://localhost:3000/api/aris/projects/<projectId>/targets | python3 -c "
import sys, json
data = json.load(sys.stdin)
for t in data.get('targets', []):
    print(f\"{t['sshServerName']}  sshServerId={t['sshServerId']}  path={t['remoteProjectPath']}\")
"
```

**Get SSH connection details for each server:**
```bash
curl -s http://localhost:3000/api/ssh-servers | python3 -c "
import sys, json
data = json.load(sys.stdin)
for s in data.get('servers', data if isinstance(data, list) else []):
    sid = s.get('id', '')
    name = s.get('name', '')
    host = s.get('host', '')
    user = s.get('user', '')
    port = s.get('port', 22)
    jump = s.get('proxyJump', '') or s.get('proxy_jump', '')
    key  = s.get('sshKeyPath', '') or s.get('ssh_key_path', '~/.ssh/id_rsa')
    print(f'{sid}|{name}|{user}@{host}:{port}|jump={jump}|key={key}')
"
```

Build the SSH command for each server. If `proxyJump` is set, use `-J <proxyJump>`. If `sshKeyPath` is set, use `-i <sshKeyPath>`.

### Step 2: Query Each Server

For each target's SSH server, run `nvidia-smi` via SSH:

```bash
ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no \
    [-J <proxyJump>] [-i <sshKeyPath>] [-p <port>] \
    <user>@<host> \
    "nvidia-smi --query-gpu=index,name,memory.used,memory.total,utilization.gpu,temperature.gpu --format=csv,noheader" 2>&1
```

If `nvidia-smi` is not found, try the full path `/usr/bin/nvidia-smi`.

If the server is unreachable or has no GPUs, note it and move on to the next.

Query all servers in **parallel** when possible (use multiple tool calls).

### Step 3: Determine Availability

A GPU is considered **free** if:
- `memory.used < 500 MiB` — completely idle
- `memory.used < 2000 MiB` AND `utilization.gpu < 10%` — likely idle (small framework overhead)

A GPU is **partially available** if:
- Used memory is less than 50% of total memory

A GPU is **busy** if:
- Used memory exceeds 50% of total memory

### Step 4: Present Results

Group results by ARIS project, then by server. Display a table per server:

```
## Project: AutoRDL

### papermachine  (chenzh85@papermachine.egr.msu.edu via scully)

| GPU | Model       | VRAM Used   | VRAM Total | Util % | Temp | Status    |
|-----|-------------|-------------|------------|--------|------|-----------|
| 0   | RTX A5000   | 234 MiB     | 24564 MiB  | 0%     | 32C  | Free      |
| 1   | RTX A5000   | 22100 MiB   | 24564 MiB  | 87%    | 71C  | Busy      |
| ...                                                                       |

### lab-server  (user@lab.cs.university.edu)

| GPU | Model       | VRAM Used   | VRAM Total | Util % | Temp | Status    |
|-----|-------------|-------------|------------|--------|------|-----------|
| ...                                                                       |
```

Then a cross-project summary:

```
### Summary
AutoRDL (6 servers):
  - papermachine: 5 free, 1 partial, 2 busy (8 GPUs)
  - lab-server:   4 free, 0 partial, 0 busy (4 GPUs)
  - ...
Total free GPUs: 28
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
- If no ARIS projects or targets exist, tell the user to create a project and add SSH server targets in the Auto Researcher ARIS workspace
- Deduplicate servers: if multiple projects share the same SSH server, only query it once but show the results under each project
