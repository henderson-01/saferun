# saferun 🛡️
A zero-trust, disposable `Docker sandbox` designed for **macOS and Linux** systems, for exploring unfamiliar GitHub repositories and running AI coding agents safely.

---

## Why saferun?
When contributing to new open-source projects, cloning repos locally and letting an AI coding agent (like OpenCode) execute build scripts or test suites can sometimes introduce unexpected risks. A rogue script or accidental command can break your host operating system.

`saferun` solves this by giving you an instant, isolated containerized workspace. It provides:

- Absolute Isolation: Any untrusted code execution, rogue `setup.py` scripts, or accidental destructive commands run strictly inside a throwaway Linux container. They cannot access your host OS binaries, system Python, or personal files outside the project.

- 🔒 Optional Offline Mode: Includes a dedicated `saferun-offline` mode (`--network none`) to completely block external network requests, stopping data exfiltration and rogue installation callbacks dead in their tracks.

- Headless Environment: Optimized purely for `CLI development`, web servers, test runners, and AI agents. Graphical UI (GUI) popups are intentionally disabled.

- Zero Permission Headaches: By dynamically passing your user ID to Docker, any files or virtual environments generated inside the sandbox belong to you, not `root`. Your IDE (like `PyCharm`) can save and modify files without "Permission Denied" errors.

- Lightning Fast: Uses a pre-built local image and a dedicated Docker volume for `uv`. Startup takes milliseconds, and package downloads are cached permanently between sessions.

- Persistent AI Context: Safely maps your `OpenCode configuration` (`agents.md` and login sessions) into the container so your AI agent always knows your preferences without polluting your system environment.

- Clean Exit: When you exit, the environment is destroyed instantly, leaving your host system completely clean, while your modified project files remain safely saved locally and ready to push to GitHub.

---

## Prerequisites
Before setting up `saferun`, it is recommended to install and configure **OpenCode** globally on your host machine first:

1. Install OpenCode on your host system following its standard installation guide.
2. Run OpenCode once on your machine to complete initial setup and log in.

> **Why?** `saferun` seamlessly mounts your host's OpenCode configuration into the container. Setting it up on your main system first ensures your AI preferences, API keys, and login sessions persist automatically inside the sandbox without asking you to re-authenticate every time.

---

## Setup Instructions
Choose the setup instructions below depending on your operating system (Mac defaults to `zsh`, while Ubuntu defaults to `bash`).

### Step 1: Create the Permanent Dockerfile
Instead of installing tools from scratch every time, we build a lightweight base image once.

Open your standard terminal and run these commands to create a configuration folder and open a new file:

```Bash
mkdir -p ~/.saferun
```
Then Run:
```Bash
nano ~/.saferun/Dockerfile 
```

Paste the following blueprint inside. (Note: We intentionally omit `Git` here because you should use Git on your host machine via PyCharm).

```Dockerfile
FROM python:3.12-slim

# Install curl and essential build tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install uv globally
RUN pip install --no-cache-dir uv

# Install OpenCode, copy the binary, and clean up installer leftovers
RUN curl -fsSL https://opencode.ai/install | bash && \
    find /root -name "opencode" -type f -exec cp {} /usr/local/bin/ \; && \
    rm -rf /root/.cache /root/.local /tmp/*

# Pre-create paths and fix permissions for dynamic non-root UIDs
RUN mkdir -p /uv-cache /app-data /tmp/.config && chmod 777 /uv-cache /app-data /tmp/.config

# Fix the "I have no name!" prompt for dynamic UIDs
RUN echo 'export PS1="Docker Terminal:\w\$ "' >> /etc/bash.bashrc

WORKDIR /app
CMD ["bash"]

```
Save and exit your editor (in nano, press `Ctrl+O`, `Enter`, then `Ctrl+X`).

> [!TIP] 
> If you ever need a different Python version (e.g., Python 3.11), simply change `FROM python:3.12-slim` in your Dockerfile, save the file, and re-run the build with the clean-cache command:
> 
> `docker build --no-cache -t saferun-base ~/.saferun/`

---

### Step 2: Build the Local Sandbox Image
Run the following command to package the `Dockerfile` into a ready-to-use image on your machine. You only ever need to do this once.

```Bash
docker build -t saferun-base ~/.saferun/
```

---

### Step 3: Add the Alias to Your Shell Profile
Now, tie it all to the simple saferun command. Open your shell's configuration file:

> [!IMPORTANT]
> **First-Time Setup Note:** If you have not installed or launched `OpenCode` on your host machine yet, run this command **before** using `saferun`:
> ```bash
> mkdir -p ~/.config/opencode ~/.local/share/opencode
> ```
> **Why?** If these host directories do not exist when Docker starts, Docker will auto-create them on your machine with `root` ownership. Pre-creating them manually ensures your normal user account owns the folders, preventing permission errors when saving OpenCode settings.

#### For Mac (zsh):
```Bash
nano ~/.zshrc
```

#### For Ubuntu (bash):
```Bash
nano ~/.bashrc
```

Scroll to the very bottom of the file and paste this optimized alias:

```Bash 
alias saferun='docker run --rm -it -u "$(id -u):$(id -g)" -e HOME=/tmp -e UV_CACHE_DIR=/uv-cache -e XDG_DATA_HOME=/app-data -v "$(pwd):/app" -v uv-cache:/uv-cache -v ~/.config/opencode:/tmp/.config/opencode -v ~/.local/share/opencode:/app-data/opencode -w /app saferun-base'
```

### What this command actually does:

- `--rm`: Destroys the container instantly when you type exit.

- `-u "$(id -u):$(id -g)"`: Tells the container to run as you instead of the root user, protecting your host file permissions.

- `-v "$(pwd):/app"`: Live-mounts the project folder you are currently sitting in.

- `-v uv-cache...`: Mounts an isolated Docker volume specifically to cache Python packages, saving you download time without touching your host OS cache.

- `-v ~/.config/opencode...`: Safely passes your OpenCode login and agents.md preferences into the sandbox.

Save and exit your editor.

---

### Step 4: Reload Your Terminal Settings
Apply the changes immediately by reloading your shell profile:

#### For Mac:
```Bash
source ~/.zshrc
```

#### For Ubuntu:
```Bash
source ~/.bashrc
```

---

## Daily Workflow
Now that your system recognizes saferun, your development workflow is effortless and safe:

1. Clone & Open: Clone any project from GitHub to your local machine as usual, and open it in your preferred editor (e.g., PyCharm).

2. Open the Terminal: Navigate to your IDE's built-in terminal tab. Ensure you are at the root of the project folder.

3. Enter the Sandbox: Type the alias and press Enter:
```Bash
saferun
```

4. Code & Execute Safely: You are now inside the isolated Linux environment.

- Edit files normally in your PyCharm UI (changes sync instantly).

- Run untrusted code or test suites without fear.

- Let uv manage your virtual environments (uv run pytest) automatically.

- Ask opencode to generate or refactor code.

> [!NOTE] 
> Continue to use Git through your host machine (either via PyCharm's UI or a separate regular terminal tab) since the sandbox is intentionally isolated from your global Git credentials.

5. Exit & Clean Up: When you are finished, simply type:
```Bash
exit
```

The execution sandbox vanishes into thin air, leaving your code edits safely saved and ready to commit.

---

## 🔒 Advanced Security: Going Offline (saferun-offline)

As noted by security feedback, the highest risk when running untrusted GitHub repositories isn't usually file system damage—it's **rogue installation scripts or setup callbacks phoning home** to exfiltrate data or download payloads.

To counter this, `saferun` supports a fully offline execution mode.

### How to use it:
Add a secondary alias to your shell profile (`~/.bashrc` or `~/.zshrc`) alongside your main command:

```Bash
alias saferun-offline='docker run --rm -it --network none -u "$(id -u):$(id -g)" -e HOME=/tmp -e UV_CACHE_DIR=/uv-cache -e XDG_DATA_HOME=/app-data -v "$(pwd):/app" -v uv-cache:/uv-cache -v ~/.config/opencode:/tmp/.config/opencode -v ~/.local/share/opencode:/app-data/opencode -w /app saferun-base'
```

---

## The Recommended Hybrid Workflow:
First Boot (`saferun`): Run your standard container with network access enabled to let uv download dependencies and packages. Then type exit.

Execution (`saferun-offline`): Spin up your container using the offline alias. Any malicious script attempting a network callback, external fetch, or telemetry exfiltration will instantly fail.

---

## Why Use Offline Mode?
- Zero Data Exfiltration: Completely blocks malicious scripts, hidden telemetry, or rogue installation callbacks from sending your local data, project files, or environment variables to external servers.

- Smart Hybrid Workflow: Separates the initial package download phase from the execution phase, ensuring you never run into installation errors while keeping your test runs entirely locked down.

---

## License

This project is licensed under the [MIT License](LICENSE).
