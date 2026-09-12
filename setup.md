# Prerequisites: Clean Machine Setup Commands

Everything below gets a completely fresh machine ready for all four labs (Git/PR, CI/CD, Containerization, MLOps). Pick your OS section. Verification commands are at the end of each section — run them before the session starts.

**Python version note:** none of the labs use a 3.12-specific feature, so **Python 3.10 or newer is fine**. If your machine already has 3.10 or 3.11 (common on Ubuntu 22.04), don't bother installing 3.12 — just use what's there. Commands below install 3.12 where it's easy (macOS, Windows) and treat it as optional on Linux.

---

## macOS

Uses [Homebrew](https://brew.sh) as the package manager.

```bash
# Homebrew (skip if already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Git
brew install git

# GitHub CLI (optional but makes PR creation from the terminal easy)
brew install gh

# Docker Desktop
brew install --cask docker
open /Applications/Docker.app   # launch it once so the daemon starts

# Node.js 20 (via nvm, so you can switch versions later if needed)
brew install nvm
mkdir -p ~/.nvm
echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.zshrc
echo '[ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"' >> ~/.zshrc
source ~/.zshrc
nvm install 20
nvm use 20

# Python 3.12
brew install python@3.12
```

**Verify:**
```bash
git --version
gh --version
docker run hello-world
node --version      # should print v20.x
python3.12 --version
```

---

## Windows (PowerShell, using winget)

Docker on Windows requires **WSL2** — install that first.

```powershell
# WSL2 (requires a restart after this)
wsl --install

# --- after restart, continue below ---

# Git
winget install --id Git.Git -e

# GitHub CLI (optional)
winget install --id GitHub.cli -e

# Docker Desktop (uses the WSL2 backend automatically)
winget install --id Docker.DockerDesktop -e
# Launch Docker Desktop once from the Start menu so the engine starts

# Node.js 20
winget install --id OpenJS.NodeJS.LTS -e

# Python 3.12
winget install --id Python.Python.3.12 -e
```

**Verify (new PowerShell window, after Docker Desktop has fully started):**
```powershell
git --version
gh --version
docker run hello-world
node --version
python --version
```

---

## Linux (Ubuntu/Debian)

Ubuntu's default repos ship an old Node.js (v12 on 22.04) and cap Python at 3.10/3.11 depending on release — the commands below account for both.

```bash
sudo apt-get update

# Git
sudo apt-get install -y git

# GitHub CLI (optional)
type -p curl >/dev/null || sudo apt-get install curl -y
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt-get update
sudo apt-get install -y gh

# Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker   # or log out/in so the group change takes effect

# Node.js 20 — remove the old distro package FIRST, or NodeSource's
# install can end up shadowed by the ancient default (v12 on Ubuntu 22.04)
sudo apt-get remove -y nodejs libnode-dev npm
sudo apt-get autoremove -y
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Python — use what's already installed if it's 3.10+ (check first: python3 --version)
python3 --version

# Only if you specifically need 3.12 and it's not available in the default repos:
sudo apt-get install -y software-properties-common
sudo add-apt-repository -y ppa:deadsnakes/ppa
sudo apt-get update
sudo apt-get install -y python3.12 python3.12-venv python3.12-dev
```

**Verify:**
```bash
git --version
gh --version
docker run hello-world
node --version        # should now print v20.x, not v12.x
python3 --version     # 3.10+ is fine; use python3.12 --version if you installed it above
```

---

## After Docker Is Confirmed Working — Pull the Base Images Ahead of Time

Do this once on the same network you'll be teaching on, so participants aren't all pulling multi-hundred-MB images simultaneously during the session:

```bash
docker pull node:20-slim
docker pull python:3.12-slim
docker pull postgres:16
docker pull redis:7
```

(The GPU-specific `nvidia/cuda:...` image from the MLOps lab's walkthrough section is optional to pre-pull — most participants won't run it live.)

---

## Python Packages Used Across the Labs (install inside each lab's folder, not globally)

These are already listed in each lab's `requirements.txt`, but for reference, the full set across all labs is:

```
scikit-learn==1.5.0
joblib==1.4.2
flask==3.0.3
schedule==1.2.1
```

Install with:
```bash
python3 -m venv venv            # or python3.12 -m venv venv if you installed 3.12
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## Node Packages Used Across the Labs

```
express@^4.19.0
jest (installed as a dev dependency during the CI/CD lab)
```

No global npm installs are required — each lab's `npm install` step handles it locally.

---

## One-Line Sanity Check (run this last, on every machine)

```bash
git --version && docker run --rm hello-world && node --version && python3 --version
```

If all four print version/success output without errors, the machine is ready for the full session.

