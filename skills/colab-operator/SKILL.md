---
name: colab-operator
description: Operate Google Colab environments via the colab CLI. Provision GPU/TPU sessions, run Python/shell code on the VM, sync files, and check authentication.
---

# Colab Session Operator

Operate Google Colab environments via the `colab` CLI: provision GPU/TPU sessions, run Python/shell on the VM, sync files, and capture work.

## When to activate
- Creating or managing TPU/GPU sessions.
- Running Python or shell on a remote Colab VM.
- Syncing files between local and remote.
- Automating environment setup (packages, auth, Drive).

## Prerequisites Check & Installation
Before executing any `colab` commands, verify if the CLI tool is installed:
1. Run a quick check: `colab --version`
2. If the tool is missing, install it:
   ```bash
   uv tool install google-colab-cli
   # Or fallback to pip:
   pip install google-colab-cli
   ```

## Authentication Setup & Verification
Colab API requests will fail if the CLI has incorrect authentication scopes.
1. **Verification**: Run the hidden debug command `colab whoami` to inspect the active identity and OAuth scopes.
2. **Re-authenticating ADC (Recommended)**: If scopes are missing or a 403 error is encountered, run:
   ```bash
   gcloud auth application-default login \
     --scopes=openid,https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/userinfo.email,https://www.googleapis.com/auth/colaboratory
   ```
   *Note: All four scopes are strictly required.*

## Core Workflows & Command Reference

### 1. Provisioning a Session
Always name your session explicitly to prevent naming conflicts:
```bash
# Provision a CPU-based VM
colab --auth=adc new -s my-session

# Provision a GPU or TPU VM (T4, L4, A100, H100, v5e1, v6e1)
colab --auth=adc new -s my-session --gpu L4
```

### 2. Execution
To run code programmatically without hanging:
* **Avoid interactive REPLs** (`colab repl` or `colab console`) unless specifically requested by the user, as they require a TTY.
* **Preferred execution**: Send code via piped stdin:
  ```bash
  echo "import torch; print(torch.cuda.is_available())" | colab --auth=adc exec -s my-session
  ```
* **Running local scripts**:
  ```bash
  colab --auth=adc exec -s my-session -f train.py
  ```

### 3. File Syncing
```bash
# List files in the workspace (/content is the default working directory)
colab --auth=adc ls -s my-session

# Upload a local file to the VM
colab --auth=adc upload -s my-session local_data.csv /content/data.csv

# Download a file from the VM to local path
colab --auth=adc download -s my-session /content/checkpoint.bin ./checkpoint.bin
```

### 4. Cleanup
Always release VMs when execution is complete to stop consuming compute units:
```bash
colab --auth=adc stop -s my-session
```

### 5. Handling Long-Running Processes
For processes that take hours to complete (e.g., application servers, migrations):

#### Background Execution with tmux (Recommended/High Priority)
Using a detached `tmux` session is the most robust way to run background processes. It allows process state preservation and session listing:

1. **Start a detached tmux session running your command:**
   ```bash
   echo "tmux new-session -d -s migration 'python migrate.py > migrate.log 2>&1'" | colab --auth=adc exec -s my-session
   ```
2. **List running tmux sessions to verify:**
   ```bash
   echo "tmux ls" | colab --auth=adc exec -s my-session
   ```
3. **Send commands or keys to the session (if needed):**
   ```bash
   echo "tmux send-keys -t migration 'ls' C-m" | colab --auth=adc exec -s my-session
   ```

#### Alternative: Background Execution with nohup (Fallback)
If `tmux` is not preferred or available, use `nohup` or `&` via piped input:
```bash
echo "nohup python migrate.py > migrate.log 2>&1 &" | colab --auth=adc exec -s my-session
```

#### Keep-Alive Loop (Local)
To prevent the Colab VM from shutting down due to the **Idle Timeout** (~20 mins), run a periodic poll loop on your local machine:
```bash
while true; do
  echo "tail -n 10 migrate.log" | colab --auth=adc exec -s my-session
  sleep 300
done
```

#### Session Constraints and Limits
*   **Free Tier & Pro ($9.99/mo):** Do not support background execution. If you close your local keep-alive loop, the VM will be pruned. Runtimes have a hard cap of 12 hours max.
*   **Pro+ ($49.99/mo):** Supports native background execution up to 24 hours. No local keep-alive loop is needed.
*   **Alternative (Modal):** If fully unattended headless execution is required, consider deploying to Modal (using `modal deploy modal_app.py`). Modal scales resources dynamically and runs completely independent of your local machine.

