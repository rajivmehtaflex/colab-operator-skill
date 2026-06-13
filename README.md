# Google Colab CLI Skill (`colab-operator`)

This repository contains the `colab-operator` skill and plugin configuration designed for Antigravity agent environments. It enables agents to provision, manage, and execute commands on Google Colab remote VMs using the `google-colab-cli`.

## Repository Structure

- `plugin.json`: Plugin configuration registering the skill with Antigravity.
- `skills/colab-operator/SKILL.md`: The skill definition file containing instructions and workflows for the agent.

---

## How to Consume This Skill

To consume this skill, you need to register it as a custom plugin in your Antigravity agent workspace environment.

### Option 1: Direct Clone / Copy (Recommended)
1. Navigate to your Antigravity configuration plugins directory (typically `~/.gemini/antigravity/` or your project's custom plugins path).
2. Clone this repository:
   ```bash
   git clone https://github.com/rajivmehtaflex/colab-operator-skill.git google-colab-cli
   ```
3. Alternatively, copy the `plugin.json` and `skills/` directory structure directly into a custom plugin folder in your workspace.

### Option 2: Add as Git Submodule
If you manage your workspace plugins within a parent Git repository, add this as a submodule:
```bash
git submodule add https://github.com/rajivmehtaflex/colab-operator-skill.git plugins/google-colab-cli
```

### Reloading the Plugin
After adding the files, reload plugins to activate the skill:
- **In CLI/Terminal:** Run the reload command:
  ```bash
  antigravity plugin reload
  ```
- **In Web UI/Desktop:** Restart your Antigravity agent session or trigger a workspace sync.

Once activated, your Antigravity agent will automatically detect and execute commands via the `colab-operator` skill when managing Colab resources.

---

## Setup & Prerequisites (Observations)

To ensure the agent can execute these tasks successfully, verify the following prerequisites are met on the host machine where the agent runs.

### 1. Install Google Colab CLI
The command `colab` must be accessible in the agent's path.
- **Using uv (Preferred):**
  ```bash
  uv tool install google-colab-cli
  ```
- **Using pip:**
  ```bash
  pip install google-colab-cli
  ```

### 2. Configure OAuth Credentials (Critical)
The CLI requires specific OAuth scopes to interact with the Colab API. Standard authentication will fail with `403 Forbidden` errors. Run the following command to log in with all required scopes:

```bash
gcloud auth application-default login \
  --scopes=openid,https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/userinfo.email,https://www.googleapis.com/auth/colaboratory
```

You can verify the current authenticated user and active scopes at any time by running:
```bash
colab whoami
```

---

## Best Practices & Key Discoveries

### Non-Interactive Execution
Antigravity agents should **never** run interactive REPL commands like `colab repl` or `colab console` as they require an interactive TTY. Always run commands using piped stdin:
```bash
echo "import torch; print(torch.cuda.is_available())" | colab --auth=adc exec -s my-session
```

### Managing Long-Running Processes with `tmux`
For long-running tasks (e.g., training runs, database migrations) that take hours, do not rely on raw execution commands as they will hang the agent.
- **Start a detached tmux session:**
  ```bash
  echo "tmux new-session -d -s migration 'python migrate.py > migrate.log 2>&1'" | colab --auth=adc exec -s my-session
  ```
- **List active sessions:**
  ```bash
  echo "tmux ls" | colab --auth=adc exec -s my-session
  ```
- **Interact / Send input to session:**
  ```bash
  echo "tmux send-keys -t migration 'ls' C-m" | colab --auth=adc exec -s my-session
  ```

### VM Keep-Alive vs. Subscription Tiers
Google Colab VMs will terminate if left idle (typically ~20 minutes). 
- **Free Tier / Colab Pro ($10/mo):** Do not support background execution when the local browser or connection is disconnected. To keep the VM alive, the agent must run a local keep-alive poll loop:
  ```bash
  while true; do
    echo "tail -n 10 migrate.log" | colab --auth=adc exec -s my-session
    sleep 300
  done
  ```
- **Colab Pro+ ($50/mo):** Supports native background execution (up to 24 hours). No keep-alive loop is necessary.
- **Serverless Alternative (Modal):** If headless unattended execution is required, consider deploying tasks directly via **Modal** (`modal_app.py`) instead, which bypasses local session keep-alives and scales compute dynamically.
