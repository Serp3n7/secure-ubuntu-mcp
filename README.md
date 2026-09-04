<div align="center">

# 🔒 Secure Ubuntu MCP Server

**Hardened Model Context Protocol server · safe, controlled access to Ubuntu system operations**

`mcp` · `python` · `security-first`

[![Python 3.9+](https://img.shields.io/badge/Python-3.9%2B-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/downloads/)
[![MCP](https://img.shields.io/badge/MCP-1.0+-0A3A5F?style=flat-square&logo=modelcontextprotocol&logoColor=white)](https://modelcontextprotocol.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)
[![Security Focused](https://img.shields.io/badge/Security-Focused-3DDC84?style=flat-square)](#-security-features)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-18.04%2B-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com/)

---

Give your AI assistant **safe, audited, policy-controlled** access to an Ubuntu machine — through the power of the standard [Model Context Protocol](https://modelcontextprotocol.io/).

```
┌───────────────────────┐      ┌───────────────────────────┐      ┌─────────────────────┐
│  AI Assistant (Claude │ ───▶ │  ubuntu-mcp-server        │ ───▶ │  Ubuntu System      │
│  Desktop, custom      │ MCP  │  stdio/stdin-stream       │ safe │  files · commands ·  │
│  clients)             │      │  SecurityChecker │Audit   │      │  packages · system   │
└───────────────────────┘      └───────────────────────────┘      └─────────────────────┘
```

Never expose a raw shell. The server validates **every path, command, and file** against a configurable policy before it touches your system — and logs everything it does.

</div>

---

## 📑 Table of Contents

- [✨ Key Features](#-key-features)
- [🚀 Quick Start](#-quick-start)
- [⚙️ CLI Reference](#️-cli-reference)
- [🤖 Claude Desktop Integration](#-claude-desktop-integration)
- [🛡️ Security Policies](#️-security-policies)
- [🔍 Available Tools](#-available-tools)
- [🔒 Security Features](#-security-features)
- [🧪 Testing](#-testing)
- [📦 Installation Scripts](#-installation-scripts)
- [🛠️ Development](#️-development)
- [🔧 Troubleshooting](#-troubleshooting)
- [📄 License & Disclosure](#-license--disclosure)

---

## ✨ Key Features

| Area | What you get |
|------|--------------|
| 🛡️ **Security-first** | Symlink-aware path resolution, `shlex` command parsing, deny-by-default policies, sanitized environment |
| 🧱 **Defense in depth** | Multiple independent validation layers — a single bypass never grants system access |
| 📜 **Full audit trail** | Every command, file access, and violation logged with user attribution |
| 📦 **Safe package ops** | APT search & list only — no accidental installs unless you widen the policy |
| 🧪 **Proven** | Built-in security test suite (`--security-test`) exercises real attack vectors |
| 🪶 **Zero extra deps** | Core security is pure-Python; only `mcp` + `psutil` required |

---

## 🚀 Quick Start

```bash
# 1. Clone & enter
git clone <your-repo-url> && cd ubuntu_mcp_server

# 2. Interactive setup (venv + deps + tests + Claude config) — recommended
python3 setup.py

# --- or, manually ---
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 3. Verify
python main.py --test
python main.py --security-test
```

**Run it:**

```bash
python main.py --policy secure   # 🔐 production default
python main.py --policy dev      # 🧪 more permissive, for development
```

---

## ⚙️ CLI Reference

```
usage: main.py [options]

--policy secure|dev    Security policy to use                          [default: secure]
--test                 Run functionality tests
--security-test        Run the comprehensive security validation suite
--log-level LEVEL      Logging level: INFO, DEBUG, WARNING, ERROR      [default: INFO]
```

---

## 🤖 Claude Desktop Integration

### 1. Install Claude Desktop (Linux)

Officially, Claude Desktop targets macOS/Windows — the community-published Debian package by [@aaddrick](https://github.com/aaddrick) fills the gap:

```bash
wget https://github.com/aaddrick/claude-desktop-debian/releases/latest/download/claude-desktop_latest_amd64.deb
sudo dpkg -i claude-desktop_latest_amd64.deb
sudo apt-get install -f
```

### 2. Register the server

Edit `~/.config/Claude/claude_desktop_config.json` — **absolute paths required** (the `~` shorthand is not expanded):

```json
{
  "mcpServers": {
    "secure-ubuntu": {
      "command": "/path/to/ubuntu_mcp_server/.venv/bin/python3",
      "args": ["/path/to/ubuntu_mcp_server/main.py", "--policy", "secure"],
      "env": { "MCP_LOG_LEVEL": "INFO" }
    }
  }
}
```

> 💡 `setup.py` merges this block for you automatically. Alternatively point `command` at the bundled [`run_mcp_server.sh`](run_mcp_server.sh) launcher, which handles venv activation.

### 3. Verify

Restart Claude Desktop → confirm **"secure-ubuntu"** appears as a connected server → try:

> *"Check my system status and disk space"*

### Other MCP clients

The server speaks standard MCP over stdio — any compatible client works. See [`test_client.py`](test_client.py) for a reference client:

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def example():
    params = StdioServerParameters(command="python3", args=["main.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            # ... call tools ...

asyncio.run(example())
```

---

## 🛡️ Security Policies

Two ships-with-the-box policies plus a custom-policy API. **Start with `secure`, widen deliberately.**

### 🔐 Secure Policy — *default, production*

| Control | Value |
|---------|-------|
| Allowed paths | `~/`, `/tmp`, `/var/tmp` |
| Forbidden paths | `/etc`, `/root`, `/boot`, `/sys`, `/proc`, `/dev`, `/var/log`, `/var/lib`, `/usr`, `/sbin`, `/bin` |
| Command mode | **Whitelist** (deny-by-default) |
| Allowed commands | `ls cat echo pwd whoami date uname` · `grep head tail wc sort uniq cut` · `find which file stat du df` · `apt` (list/search only) |
| Forbidden commands | `rm rmdir dd mkfs fdisk cfdisk` · `shutdown reboot halt init systemctl service` · `mount umount chmod chown chgrp` · `su sudo passwd useradd userdel usermod` · `crontab at batch nohup pkill kill` |
| File size / output | 1 MB / 256 KB |
| Timeout / listing | 15 s / 100 items |
| Sudo · shell | ❌ Disabled · ❌ Disabled (direct `exec`, no shell) |

### 🧪 Development Policy — *less restrictive*

| Control | Value |
|---------|-------|
| Allowed paths | Secure + `/opt`, `/usr/local` |
| Command mode | **Denylist** (whitelist off — most tools allowed) |
| File size / output | 10 MB / 1 MB |
| Timeout / listing | 60 s / 500 items |
| Sudo | ❌ Still disabled |

### ✍️ Custom Policies

```python
from main import SecurityPolicy, SecureUbuntuController

policy = SecurityPolicy(
    allowed_paths=["/your/custom/paths"],
    forbidden_paths=["/sensitive/areas"],
    allowed_commands=["safe", "commands"],
    forbidden_commands=["dangerous", "commands"],
    command_whitelist_mode=True,   # True = deny-by-default
    max_command_timeout=30,
    allow_sudo=False,              # use with extreme caution
    audit_actions=True,
    resolve_symlinks=True,         # always on for security
    use_path_cache=False,          # off by default (TOCTOU-safe)
    use_shell_exec=False,          # off by default
)

controller = SecureUbuntuController(policy)
```

### ⚙️ Configuration

`config.py` provides dataclass-based loading/saving (`ConfigManager`, `ServerConfig`, `SecurityConfig`) with a default file at `~/.config/ubuntu-mcp/config.json`, created on first run. A ready-to-edit example ships in the repo as `config.json`.

```json
{
  "server": {
    "name": "ubuntu-controller",
    "version": "1.0.0",
    "description": "MCP Server for Ubuntu System Control",
    "log_level": "INFO"
  },
  "security": {
    "policy_name": "safe",
    "allowed_paths": ["~/", "/tmp", "/var/tmp"],
    "forbidden_paths": ["/etc/passwd", "/etc/shadow", "/root", "/boot", "/sys", "/proc"],
    "allowed_commands": ["ls", "cat", "echo", "pwd", "whoami", "date", "grep", "find", "which", "file", "head", "tail", "apt", "git", "python3", "pip3"],
    "forbidden_commands": ["rm", "rmdir", "dd", "mkfs", "shutdown", "reboot", "mount", "umount", "chmod", "chown"],
    "max_command_timeout": 30,
    "allow_sudo": false
  }
}
```

> ℹ️ Runtime policy is chosen via `--policy`; `config.py` is the programmatic / embedded-config path.

---

## 🔍 Available Tools

| Tool | Signature | Notes |
|------|-----------|-------|
| 📁 `list_directory` | `(path)` | Metadata: size, perms `-rw-r--r--`, owner, group, mtime, symlink flag |
| 📄 `read_file` | `(file_path)` | UTF-8, size-validated, encoding-tolerant |
| ✍️ `write_file` | `(file_path, content, create_dirs=False)` | Atomic temp+rename, timestamped `.backup.*` of prior file |
| ⌨️ `execute_command` | `(command, working_dir=None)` | Direct exec, sanitized PATH/env, timeout + output caps |
| 🖥️ `get_system_info` | `()` | OS, memory, disk usage, user, hostname, arch |
| 🔎 `search_packages` | `(query)` | `apt search`, input format-validated |
| 📦 `install_package` | `(package_name)` | `apt list --installed` check — **never installs** |

---

## 🔒 Security Features

### ✅ Path traversal — blocked

```
../../../etc/passwd        → SecurityViolation
/etc/passwd                → SecurityViolation
/tmp/../etc/passwd         → SecurityViolation
symlink_to_/etc/passwd     → SecurityViolation  (symlinks fully resolved)
```

Every path is canonicalized with symlink resolution, then matched against allowlist + denylist **before** access. Access to the server's own files and `system_critical_paths` is always refused.

### ✅ Command injection — blocked

```
echo hello; rm -rf /     → SecurityViolation
echo `cat /etc/passwd`   → SecurityViolation
echo $(whoami)           → SecurityViolation
ls | rm -rf /            → SecurityViolation
```

Parsed with `shlex` (no shell interpretation), the executable is resolved to a real absolute path (defeats `PATH` injection), checked against the whitelist/blacklist, then run **directly via `subprocess.exec`** with a sanitized environment (`LD_PRELOAD`, `LD_LIBRARY_PATH`, `IFS` stripped).

### ✅ Resource exhaustion — mitigated

| Threat | Defense |
|--------|---------|
| Oversized file reads | Hard size limit before read |
| Hanging commands | Per-process-group timeout, `killpg` on expiry |
| Output flooding | stdout/stderr capped + truncation marker |
| Directory enumeration | Item-count cap with `[TRUNCATED]` marker |

### 📜 Audit trail

Everything lands in `/tmp/ubuntu_mcp_audit.log`:

```
COMMAND_ATTEMPT: user=serp3n7 cmd='ls -la' cwd=default
FILE_READ:       user=serp3n7 path='/home/serp3n7/foo' status=SUCCESS
SECURITY_VIOLATION: user=serp3n7 violation='COMMAND_BLOCKED' details='Command not in whitelist: nmap'
```

---

## 🧪 Testing

```bash
./run_tests.sh          # full suite: controller, MCP client, policies, files, packages
python main.py --test   # functionality tests
python main.py --security-test   # attack-surface validation
python test_client.py             # end-to-end MCP protocol client
python test_client.py --simple    # controller-level smoke test
```

The `--security-test` suite validates real vectors — symlink attacks, path traversal, server self-protection, injection patterns, forbidden commands, and size limits — and exits non-zero on any bypass.

---

## 📦 Installation Scripts

| Script | Use | Notes |
|--------|-----|-------|
| [`setup.py`](setup.py) | Interactive dev setup | venv → deps → `config.example.json` → tests → optional Claude Desktop config |
| [`install.py`](install.py) | System install (root) | Installs to `/opt/ubuntu-mcp`, creates an `ubuntu-mcp` systemd service |
| [`run_mcp_server.sh`](run_mcp_server.sh) | Claude Desktop launcher | Activates `.venv`, then `exec`s `main.py` |
| [`check_disk_space.py`](check_disk_space.py) | Disk-space demo | Prints a usage bar using `get_system_info` |

```bash
# systemd service install
sudo python3 install.py
sudo systemctl enable --now ubuntu-mcp
```

---

## 🛠️ Development

### Add a tool

```python
from main import create_secure_policy, SecureUbuntuController

controller = SecureUbuntuController(create_secure_policy())

@mcp.tool("your_tool_name")
async def your_tool(param: str) -> str:
    try:
        result = controller.safe_operation(param)
        return json.dumps(result, indent=2)
    except Exception as e:
        return json.dumps({"error": str(e)}, indent=2)
```

### Extend security

```python
def create_custom_policy() -> SecurityPolicy:
    return SecurityPolicy(
        allowed_paths=["/your/paths"],
        forbidden_commands=["dangerous", "commands"],
        # ...
    )
```

### Standards

- PEP 8 · type hints on all public functions · docstrings on every tool
- Tests for new functionality — run `./run_tests.sh` before opening a PR
- Security-first: permission is *denied* until a policy explicitly grants it

See [CONTRIBUTING.md](CONTRIBUTING.md) and [CHANGELOG.md](CHANGELOG.md).

---

## 🔧 Troubleshooting

| Symptom | Cause & fix |
|---------|-------------|
| *Server appears to hang* | It's not — MCP servers run continuously over stdio and wait for messages |
| `ModuleNotFoundError: No module named 'mcp'` | Not using the venv; point Claude at `.venv/bin/python3` or `run_mcp_server.sh` |
| `SecurityViolation` errors | Path/command outside policy — review the audit log and widen the policy deliberately |
| `Permission denied` | The file isn't readable/writable by the server account — check `ls -la` |

**Debug mode:**

```bash
python main.py --log-level DEBUG --policy secure
tail -f /tmp/ubuntu_mcp_audit.log
```

---

## 📄 License & Disclosure

License: [MIT](LICENSE) · Contributing: [CONTRIBUTING.md](CONTRIBUTING.md)

Found a vulnerability? **Email [radjackbartok@proton.me](mailto:radjackbartok@proton.me)** — don't open a public issue.

---

<div align="center">

**Made for the security-conscious AI community** 🚀

> 💡 **Pro tip:** start with the `secure` policy and widen it deliberately — it's easier to grant than to recover.

</div>