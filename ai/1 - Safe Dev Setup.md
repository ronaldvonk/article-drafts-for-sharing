# Setting up a safe development environment (VS Code + dev containers)

I'm writing this course material on my personal PC at home, and that's recently become a problem.

There are _a lot_ of files on this PC I don't want an AI agent to see: Terabytes of photos (I'm also a photographer), all my bookkeeping documents, and loads of old project files for clients who would be get upset if their stuff got anywhere near an AI. The same PC is where I'm writing these lessons, building demo projects, and trying out Claude Code to stay up-to-date as a developer.

So I built a sandbox. A sealed environment, without access to the other files on my machine, where I can experiment and install things. It's not a new thing, but these days, it is what all developers should get used to! After you've set it up, there are many other benefits: Your dev environment is now more portable and easier to share with others (no more 'it works on my machine'), and it lessens the impact of supply chain attacks (such as hacker hijacking npm packages).

This in an extra lesson on how to build that box. You're going to want one too.

## Why?

Three years ago, if you told me about a tool that would read files you didn't open and run commands you didn't type, I'd have called it malware. The difference is that this kind of malware is now helpful, you installed it on purpose, and most of the time you're glad it's there.

Old tools (your editor, your terminal, your linter) did exactly what you told them. New tools (Claude Code, Cursor, Copilot Agent, whatever comes next) take initiative. They poke around. They guess. Sometimes brilliantly. Sometimes wrong.

You give the AI room to move, the AI saves you time. That's fine until "room to move" means the wrong folder. So lets put a fence around it.

There are actually two fences worth having. The first is version control. Git is what saves you when the AI confidently rewrites a file you didn't ask it to touch. We'll cover that pattern throughout the course; for now, just know that _every_ AI-assisted edit is something you should be able to throw away with `git checkout .` if you don't like it.

The second fence is the one we're building here: an isolated environment, so that even if everything inside it catches fire, your laptop is fine.

## What we're building

```
your-project/
└── .devcontainer/
    └── devcontainer.json   ← describes the environment
```

That's it. One file. The file tells your editor "for this project, build me a sealed Linux environment with these tools inside." Your editor reads it, builds the environment, and from then on, every time you open this project, you're working inside the box.

The tools we'll put in the box this week:

- Node.js (for everything frontend in this course)
- Claude Code (the AI agent)

Later in the course, when we start working on the backend, we'll add one more line and .NET 10 appears inside the same box. Same file. Same pattern. The container rebuilds and the new tools are there.

Once you've done it once, you do it for every project. The box is per-project, not per-machine.

## The layers underneath

A quick explanation before we start installing things, from bottom to top:

- **Windows.** Your operating system. We're keeping this clean. (It works on OSX and Linux too, just ignore the WSL part)
- **WSL2.** A real Linux kernel running alongside Windows. Not a VM in the old sense, more like a deeply integrated second OS.
- **Docker Engine.** A daemon (not a 😈) running inside WSL2 that manages containers.
- **Your dev container.** A sealed Linux environment for this project, built from the `devcontainer.json` file.
- **VS Code.** Your editor, running on Windows but connected to the dev container.
- **Claude Code (and Node, and whatever else).** Tools installed inside the container.

If some other tutorial mentions Docker Desktop, that's a packaging of WSL2 + Docker Engine + a GUI + Compose, sold to you as one product. We're not using it. The reason is partly practical (less resource intensive, no licensing stuff, fewer black boxes), but mostly pedagogical: I want you to see what Docker actually is. It becomes easier to fix problems when you understand them.

> 🎓 If a student asks "Can't I just use Docker Desktop," the honest answer is yes, that also works, and Docker Desktop is a perfectly good tool, but we're choosing this path because it teaches them the underlying machinery, and it uses less resources on your laptops.

### For Mac and Linux students

This guide is for Windows. If you're on Mac or Linux, the structure is the same (WSL2 doesn't exist on those, but the rest does) and the result is identical. TODO: Test on Linux and OSX and update.

## Prerequisites

A few things to check before we start:

- Windows 10 (version 2004 or newer) or Windows 11
- Administrator access on your machine
- Hardware virtualization enabled in your BIOS or UEFI (almost always on by default on a modern laptop)
- About 10 GB of disk space free for the Linux environment and a few containers

If you don't know whether virtualization is enabled, run Task Manager, go to the Performance tab, click CPU. Near the bottom right it says "Virtualization: Enabled" or "Disabled."

If it's disabled, you'll need to reboot into BIOS/UEFI and turn it on. The setting is called "Intel VT-x" or "Intel Virtualization Technology" on Intel CPUs. On AMD CPUs it's usually called "SVM" or "SVM Mode", occasionally "AMD-V". It lives somewhere under "Advanced", "CPU Configuration", or sometimes "Security". The full procedure (which key to press during boot, where to look if you can't find it, how to confirm afterwards) is in the troubleshooting section at the end of this document. Scroll down there now if your virtualization is off, since the rest of this document won't work without it.

You'll also need a free Anthropic account for Claude Code. We'll create it in the last step, not now.

## Step 1 - Install WSL2

> Open PowerShell (It's in the Windows Start menu) and run:

```powershell
wsl --install
```

This installs WSL2 and Ubuntu, the default Linux distribution. Restart your computer when prompted.

After the restart, Ubuntu finishes its first-time setup and asks you to create a username and password. **Make sure you write these down somewhere (put them in your password manager!)**. They're separate from your Windows credentials.

> Verify by opening PowerShell again and running:

```powershell
wsl --status
```

You should see `Default Version: 2` and `Default Distribution: Ubuntu`. If you do, you have a real Linux running on your machine. Pretty cool.

> Now open Ubuntu. Easiest way: Use the Start menu, search for "Ubuntu" and click it. (You can also type `wsl` in a PowerShell window, but then you'll land in whatever Windows folder that PowerShell was in, which is often `C:\Windows\System32`. Save yourself the confusion and use the Start menu.)

```
ronald@MY-COMPUTER-NAME:~$
```

The `~` at the end of the path is your Ubuntu home directory. That's where you want to be. If the path shows something else (for example `/mnt/c/Windows/System32`), you came from a PowerShell that was in a Windows folder. Run `cd ~` to go home.

Anything you do from here happens inside the Linux environment, not Windows. Now lets set up the dev container!

## Step 2 - Install Docker Engine inside WSL2

I'm going to help you out and give you a script. (Alternatively, you can check https://docs.docker.com/engine/install/ubuntu/ if you want to figure it out yourself)

> Inside your Ubuntu terminal, first make sure you're in your home directory, then open a new file with nano:

```bash
cd ~
nano setup-wsl2-docker.sh
```

This opens `nano`, a simple terminal-based editor. The filename shows at the top of the window; available commands run along the bottom (`^` means `Ctrl`).

> Copy the script below and paste it into the `nano` window.

Read it first. Don't worry. You don't have to be an expert at this, but shouldn't run scripts you haven't read, even when they're from your course materials.

```bash
#!/usr/bin/env bash
#
# setup-wsl2-docker.sh
#
# Installs Docker Engine inside a WSL2 Ubuntu environment.
# For INFWAD Web App Development.
#
# Run with: bash setup-wsl2-docker.sh
#
# This script does NOT install Docker Desktop. It installs Docker Engine
# directly inside your Linux distro. Lighter, cleaner, closer to what a real
# Linux server looks like.
#
# After this script finishes, you need to run `wsl --shutdown` from PowerShell
# and reopen Ubuntu before Docker will work.

set -euo pipefail

# ---------- Sanity checks ----------

if ! grep -qi microsoft /proc/version 2>/dev/null; then
  echo "ERROR: This script is for WSL2 Ubuntu. You're not in WSL." >&2
  echo "Open Ubuntu from your Start menu and run this script there." >&2
  exit 1
fi

if ! grep -qi ubuntu /etc/os-release 2>/dev/null; then
  echo "ERROR: This script is for Ubuntu. Your distribution is something else." >&2
  echo "If you're on a different distro, the steps are similar but the apt" >&2
  echo "commands won't work as-is. See the chapter for the manual version." >&2
  exit 1
fi

if [[ $EUID -eq 0 ]]; then
  echo "ERROR: Don't run this with sudo. Run as your normal user." >&2
  echo "The script will ask for sudo when it needs to." >&2
  exit 1
fi

echo "==> WSL2 Ubuntu detected. Good."
echo ""

# ---------- Update apt and install prerequisites ----------

echo "==> Updating apt package index..."
sudo apt-get update

echo "==> Installing prerequisites (ca-certificates, curl, gnupg)..."
sudo apt-get install -y ca-certificates curl gnupg

# ---------- Add Docker's official GPG key ----------

echo "==> Adding Docker's GPG key..."
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# ---------- Add Docker's apt repository ----------

echo "==> Adding Docker repository to apt sources..."
ARCH="$(dpkg --print-architecture)"
CODENAME="$(. /etc/os-release && echo "$VERSION_CODENAME")"
echo \
  "deb [arch=${ARCH} signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu ${CODENAME} stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

echo "==> Refreshing apt with the new repository..."
sudo apt-get update

# ---------- Install Docker Engine ----------

echo "==> Installing Docker Engine, CLI, containerd, Buildx, Compose..."
sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# ---------- Group membership ----------

echo "==> Adding your user ($USER) to the docker group..."
sudo usermod -aG docker "$USER"

# ---------- Enable systemd in WSL2 ----------

WSL_CONF="/etc/wsl.conf"
if [[ ! -f "$WSL_CONF" ]] || ! grep -q "^\[boot\]" "$WSL_CONF" 2>/dev/null; then
  echo "==> Enabling systemd in $WSL_CONF..."
  sudo tee -a "$WSL_CONF" > /dev/null <<EOF

[boot]
systemd=true
EOF
  echo "    Done. You'll need to restart WSL after this script."
else
  echo "==> $WSL_CONF already has a [boot] section. Skipping."
  echo "    If systemd isn't actually enabled, edit the file manually:"
  echo "      sudo nano $WSL_CONF"
  echo "    Make sure the [boot] section contains: systemd=true"
fi

# ---------- Done ----------

cat <<'EOF'

================================================================
Docker Engine installation complete.

NEXT STEPS (you must do these for Docker to work):

  1. Close this Ubuntu terminal.

  2. Open a Windows PowerShell and run:

       wsl --shutdown

  3. Wait a few seconds, then reopen Ubuntu.
     (Start menu, or just type `wsl` in any PowerShell.)

  4. Verify Docker is working:

       docker run hello-world

     If you see a welcome message from Docker, you're done.
     If you see "permission denied" or "cannot connect", repeat
     steps 1-3. The most common cause is forgetting `wsl --shutdown`.

================================================================
EOF
```

> Save the file: `Ctrl+O`, then `Enter` to confirm the filename.

> Exit `nano`: `Ctrl+X`.

> Now run it:

```bash
bash setup-wsl2-docker.sh
```

You'll be asked for your Ubuntu password (the one you set in Step 1, not your Windows one). The script updates `apt`, adds Docker's official repository, installs Docker Engine and the CLI tools, adds your user to the `docker` group, and turns on `systemd` if it wasn't on already.

> When the script finishes, close your Ubuntu terminal. Then from a Windows PowerShell, run:

```powershell
wsl --shutdown
```

This restarts WSL, so that everything you just did takes effect.

> Reopen Ubuntu (Start menu, remember?), and verify:

```bash
docker run hello-world
```

If you see a welcome message from Docker, the daemon is running, your user can talk to it, and the layers are all wired up. It works! Yay.

If it doesn't, first scroll down to the troubleshooting chapter.

### What the script just did

Worth a brief walkthrough, so you get a feeling for all this stuff.

Docker is published as a set of packages (`docker-ce`, `docker-ce-cli`, `containerd.io`, plus a couple of plugins) in Docker's own apt repository. To install from that repository, your system needs to trust it, which means importing Docker's GPG key. The script did all of that. None of it is mysterious; it's a standard Linux package install with the trust handshake at the start.

The `docker` group is a Linux convention: anyone in that group can talk to the Docker daemon without `sudo`. The script added you to it. Group changes only take effect on a new login, which is why the WSL shutdown was necessary.

`Systemd` is the service manager that starts and stops daemons like Docker. WSL2 didn't have it by default for a long time, and it's still opt-in via `/etc/wsl.conf`. The script enabled it. After the WSL restart, the Docker daemon starts automatically whenever you open an Ubuntu terminal. We're doing all this now to make it easier to use.

## Step 3 — Install VS Code

(Note: If you already have it, make sure it's updated!)

Download from [https://code.visualstudio.com](https://code.visualstudio.com) and install with default options. The Windows installer adds `code` to your PATH, so you can start VS Code from inside your WSL terminal.

If your Ubuntu terminal was open during the VS Code install, close and reopen it. WSL terminals don't see new entries in your Windows PATH until they restart.

> Verify from your Ubuntu terminal:

```bash
code --version
```

You should see a VS Code version number. (If you see "command not found", check the troubleshooting section down below.)

## Step 4 — Install the required VS Code extensions

Two extensions, both from Microsoft:

- **WSL** — lets VS Code edit files inside your WSL2 Ubuntu environment.
- **Dev Containers** — lets VS Code open a project inside a Docker container.

> In VS Code, press `Ctrl+Shift+X` to open the Extensions panel. Search for each name and install. Check that the publisher is "Microsoft" before clicking install — there are lookalikes from less-trusted authors.

## Step 5 — Create your first dev container

Now we put it all together.

> In your Ubuntu terminal, create a project folder and open it in VS Code:

```bash
mkdir -p ~/projects/safe-dev-demo
cd ~/projects/safe-dev-demo
code .
```

VS Code opens in a new window, connected to that folder inside WSL. Look at the bottom-left of the VS Code window: you should see "`>< WSL: Ubuntu`". That's how you know it's connected. If it says something else (or nothing), you opened VS Code the wrong way. Close it and try the `code .` command again from inside the Ubuntu terminal.

> Now (inside this new VS Code window) create a folder named `.devcontainer` in the project root (the dot in the folder name matters!), and inside it create a file named `devcontainer.json` with this content:

```json
{
  "name": "INFWAD Dev Container",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/node:1": {}
  },
  "mounts": [
    "source=claude-code-config-${devcontainerId},target=/home/vscode/.claude,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode",
  "onCreateCommand": "chown -R vscode:vscode /home/vscode/.claude",
  "postCreateCommand": "curl -fsSL https://claude.ai/install.sh | bash"
}
```

What this file means, line by line:

- **image**: Start from a basic Ubuntu container image maintained by Microsoft.
- **features**: Install Node.js into the container.
- **mounts**: Mount a Docker volume at `/home/vscode/.claude` to hold Claude Code's binary and login credentials, named after this specific container so it survives rebuilds.
- **containerEnv**: Tell Claude Code where to find its config inside the container.
- **remoteUser**: Run as a non-root user called `vscode` (better practice than running as root).
- **onCreateCommand**: After the container is built but before we use it, `chown` the mounted volume to the `vscode` user (the mount comes up root-owned by default).
- **postCreateCommand**: Finally, run Anthropic's official native installer to install Claude Code. The installer puts the binary at `~/.claude/bin/claude` and adds that to PATH via `.bashrc`.

A small note on why we don't use Anthropic's own `claude-code` dev container feature: that feature installs Claude Code via `npm install -g`, which Anthropic themselves have officially deprecated. The native installer (the `curl ... | bash` line above) is the maintained install path. Confusing, but true. If Anthropic updates their dev container feature to use the native installer, this whole block becomes a one-liner again.

When VS Code notices the new file, a popup appears in the bottom right: **"Reopen in Container."** Click it.

Note: The popup disappears after a few seconds. If you miss it, trigger the same thing manually: `Ctrl+Shift+P` → type "Reopen in Container".

Sometimes the popup doesn't appear on the first try, especially right after installing VS Code. Close VS Code, reopen it with `code .` from your Ubuntu terminal, and give it another moment. In my testing it took two restarts before the popup showed up consistently. Otherwise, check the troubleshooting section for other problems.

The first build takes a moment. It's downloading the base image, installing Node, installing Claude Code. Rebuilds are much faster because Docker caches the layers.

> When the container is ready, look at the bottom-left of the VS Code window again: you should see "`>< Dev Container: INFWAD Dev Container`".

Now try it out:

> Open a terminal inside VS Code (shortcut: `` Ctrl+` ``). The prompt now looks something like `vscode ➜/workspaces/safe-dev-demo $`. That's a different machine, a different user, a different filesystem.

You're inside the box!

> Confirm everything's installed:

```bash
node --version      # a Node version
claude --version    # a Claude Code version
```

If both respond, your container is good.

## Step 6 — Sign into Claude Code

> Inside the container's terminal:

```bash
claude
```

First time you run this, Claude Code shows a short onboarding wizard: pick a text style, pick a theme, pick an effort level. Step through with the arrow keys and Enter. None of these matter much, you can change them later.

After the wizard, you'll be asked to authenticate. Claude Code prints an authentication URL in the terminal and tries to open your Windows browser to it. Depending on your setup, one of three things happens:

- Your Windows browser opens directly to the sign-in page (cleanest case)
- The browser doesn't open, but the URL is visible in your terminal. Copy it, paste it into your Windows browser, sign in there. After signing in, the browser redirects to a callback URL. Claude Code in the terminal should pick up the callback automatically.
- The callback also doesn't reach Claude Code (port forwarding race condition). The browser sits on a "this site can't be reached" page after sign-in. Look at the URL bar: it'll be `http://localhost:PORT/?code=XXX`. Copy the entire URL and paste it back into the terminal where Claude is waiting.

Sign in with whatever Anthropic account you have, or create a free one. Once authenticated, your credentials live in the Docker volume we configured in Step 5, so this is a one-time thing per project.

> Try a first prompt to confirm everything works:

> Read the `devcontainer.json` in this project and explain what it does in three bullet points.

If Claude reads the file and gives you a sensible answer, your setup is complete.

It's working!

## What this protects you from, and what it doesn't

The container has its own filesystem. Claude (or anything else you run inside it) cannot see or write files outside it unless you explicitly mount them in. Your other stuff is invisible from inside the box. Good.

The container has its own installed software. Whatever Node packages, .NET versions, or weird CLI tools end up in there don't pollute your Windows installation. If you delete the container, all of that vanishes with it. Also good.

What the container does _not_ do: it doesn't make the AI inherently safe. Inside the container, Claude can still:

- Make network requests to anywhere.
- Install packages, run scripts, create infinite-loop processes that eat your CPU.
- Confidently rewrite your code in ways you didn't want.

The container limits the _blast radius_. It doesn't eliminate the explosion. That's what version control is for. If Claude rewrites half your project in a way you didn't expect:

```bash
git checkout .
```

Everything reverts to your last commit. That's your first safety net. The container is your second. Both are useful; the container alone is not enough.

Commit early, commit often, commit before any AI-assisted edit you're not 100% sure about.

## Looking ahead

You'll do this for every project in this course, and probably for every project after. Frontend project: same `devcontainer.json`, same tools inside. Group project (GameShelf): same file. Starlog along with us: same file.

When we start working on the backend, we add one line:

```json
"ghcr.io/devcontainers/features/dotnet:2": { "version": "10.0" }
```

That's it. Rebuild the container, .NET 10 is inside, you can run `dotnet new webapi` without ever installing .NET on Windows. When the course is over and you delete the project, .NET is gone too, keeping your OS nice and clean.

## Try this now

Three small tasks to confirm everything works. Do them inside the container terminal.

1. Inside `~/projects/safe-dev-demo`, create a file called `hello.js` containing `console.log("Hello from inside the box!");`. Run it with `node hello.js`. Confirm the message prints.

2. From the container terminal, run `ls /mnt/c` (you may need `sudo`). What do you see? What does that tell you about how isolated the container is from your Windows filesystem?

3. Ask Claude Code: "What files exist in this project, and what's the operating system this container is running?" Read the answer. Compare it to what you actually have on your Windows machine.

When you're done, close VS Code, go for a walk, come back, run `code .` from your Ubuntu terminal again, and click "Reopen in Container." The whole thing should come back without any of the setup work. Woohoo!

## Troubleshooting

**"WSL2 requires an update."** Run `wsl --update` in PowerShell.

**Virtualization is disabled.** Reboot into your BIOS or UEFI. The key to press during boot is usually `F2`, `F10`, `Del`, or `Esc`, depending on your laptop. (If you don't know, search for "[laptop model] BIOS key".)

Once you're in, you're looking for the setting that turns on hardware virtualization. The name depends on your CPU:

- Intel CPUs: "Intel VT-x", "Intel Virtualization Technology", or sometimes just "Virtualization Technology".
- AMD CPUs: "SVM" or "SVM Mode" (the most common name and the least obvious one). Occasionally "AMD-V" or just "Virtualization".

The setting is usually under "Advanced", "CPU Configuration", or "Security". Sometimes it's under "Overclocking", which is annoying but real. If your BIOS has a search function, search for "virtual" or "svm".

If you really can't find it, search the internet for "[laptop model] enable virtualization". Someone has documented every laptop on earth, often with screenshots.

Enable the setting, save, and exit. Your laptop reboots back into Windows. Confirm in Task Manager (Performance → CPU → Virtualization) that it now says "Enabled".

**`docker run hello-world` says "permission denied" or "cannot connect to the Docker daemon".** You skipped the `wsl --shutdown` step after running the install script. Close all Ubuntu windows, run `wsl --shutdown` from PowerShell, reopen Ubuntu, try again.

**`docker` is not found.** The install script probably failed somewhere. Run it again and watch the output for errors. The script is loud about failures (it stops on the first one) so the error should be near the end.

**`code .` is not recognized inside WSL.** Close your Ubuntu terminal and reopen it. WSL terminals don't see new entries in your Windows PATH until they restart. If that doesn't help, run `wsl --shutdown` in PowerShell and reopen Ubuntu.

**VS Code opened, but the bottom-left doesn't show "WSL: Ubuntu".** You opened VS Code from Windows directly (Start menu, desktop icon). Close it. From the Ubuntu terminal, run `code .` inside your project folder.

**The "Reopen in Container" popup never appears.** Check that the Dev Containers extension is installed (Microsoft, not a lookalike). Then `Ctrl+Shift+P` and run "Dev Containers: Reopen in Container" manually.

**The `postCreateCommand` fails with "Permission denied" trying to write to `/home/vscode/.claude/`.** The `onCreateCommand` that `chown`s the volume didn't run, or your `devcontainer.json` is missing it. Check that the file matches the version in Step 5. If you're stuck mid-build, rebuild from scratch (Command Palette → "Dev Containers: Rebuild Container Without Cache"). If that doesn't help, delete the volume entirely and rebuild (see "Something else broken" below).

**`claude --version` works but `claude` itself hangs silently.** You're probably running an old or broken Claude Code install. The chapter's `devcontainer.json` uses the native installer (`curl -fsSL https://claude.ai/install.sh | bash`) because Anthropic deprecated the npm install path. If you somehow ended up with the npm-installed version, swap it: from inside the container, run `sudo npm uninstall -g @anthropic-ai/claude-code`, then re-run the curl install command. Open a new terminal afterwards so PATH refreshes.

**Claude Code auth never completes inside the container.** Look at the terminal carefully for the auth URL. Sometimes the browser doesn't open automatically and you need to copy the URL to your Windows browser manually. After signing in, the browser redirects to a localhost URL that "fails to load"; the failed URL's address bar contains the `?code=XXX` parameter you need to paste back into the terminal. This is a known issue with OAuth callbacks inside containers.

**Container is slow.** You're probably working from a Windows folder instead of inside WSL. Always create projects inside `~/projects/` in your Linux home, never in `C:\Users\…`. The file system performance across the Windows/WSL boundary is awful.

**Something else broken in the container.** Delete the container and start over (Command Palette → "Dev Containers: Rebuild Container Without Cache"). The `devcontainer.json` defines everything, so a rebuild is cheap. If a rebuild isn't enough and you need a fully clean state, you have to remove the volume too. The volume is in use as long as the container exists, so you have to stop and remove the container first. From an Ubuntu terminal (_outside_ the container!):

```bash
docker ps -a                         # find the dev container (running or stopped)
docker stop <container-id-or-name>   # if it's still running
docker rm <container-id-or-name>     # remove it
docker volume ls | grep claude-code  # find the volume
docker volume rm <volume-name>       # remove it
```

Then reopen the project in VS Code and let it rebuild. Note: removing the volume means signing into Claude again on the first run.

## What comes next

Once this works, you have a clean, reproducible, isolated environment for every project in this course, and a pattern you can take to any project after!

The next document (when I get around to writing it) covers Claude Code's permission system, how to write deny rules in `settings.json` for stricter projects, and an introduction to MCP, the protocol for extending Claude with custom tools. For now: get this setup working, try a few simple prompts to get a feel for Claude Code, and get comfortable with the rhythm of _open project → Reopen in Container → work → close_.

If something in this document is wrong, unclear, or out of date, tell me. The script and the `devcontainer.json` will both drift as Docker and Anthropic update things, and I'd rather hear about it from you than have you stuck on a Sunday night before a deadline.
