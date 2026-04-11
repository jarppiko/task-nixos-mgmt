# task-nixos-mgmt

A lightweight Taskfile-based toolkit for managing small fleets of NixOS hosts.

This project provides a structured and safe way to build, test, deploy, and operate NixOS configurations across multiple machines using [`go-task`](https://taskfile.dev).

> ⚠️ **Not a replacement for large-scale tools**
> This is designed for **small to medium-sized NixOS fleets** (e.g. homelabs, small infra, edge nodes).
> It is **not intended to replace tools like Colmena** or other large-scale fleet orchestration systems.

## ✨ Features

* Multi-host NixOS management via simple `task` commands
* Local + remote host handling with unified interface
* Built-in safety mechanisms for remote deployments
* Git-based workflow support
* SSH-based execution (no agent required, but recommended)
* Minimal dependencies and easy to understand logic


## ⚙️ Requirements

### 🔑 SSH Access (Required)

All managed hosts must be accessible via SSH.

* SSH key-based authentication is **required**
* Password-based SSH is **not practical** (you would be prompted repeatedly)
* An SSH agent (e.g. `ssh-agent`) is **strongly recommended**

Typical setup:

```bash
ssh-keygen -t ed25519
ssh-copy-id user@host
eval "$(ssh-agent)"
ssh-add ~/.ssh/id_ed25519
```

Why this matters:

* The Taskfile executes multiple SSH commands per operation
* Remote deployments include several validation steps
* Without an SSH agent, you would need to enter your password many times

The Taskfile also uses **SSH agent forwarding (`ssh -A`)**, so ensure it is enabled and trusted in your environment.


### 🧾 hosts.yml Configuration

You must define the hosts you want to manage in `hosts.yml`.

Example:

```yaml
version: '3'

vars:
  HOSTS:
    - host1
    - host2
    - another_hosts
```

Optional: define SSH aliases to use short host names that are not in DNS:

```yaml
vars:
  HOST_ALIASES:
    host1: host1.domain.example
    host2: host2.domain2.example
```

Notes:

* If no alias is defined, the hostname is used directly
* Aliases allow you to decouple logical host names from actual network addresses
* Internally, the Taskfile resolves this mapping automatically 

---

## 📦 Repository Structure

```text
.
├── Taskfile.yml
├── hosts.yml
├── hosts/
│   └── <hostname>/
│       └── configuration.nix
```

Each host:

* Must exist in `hosts.yml`
* Must have a corresponding directory under `hosts/`
* Must contain a valid `configuration.nix`

Example:

```text
hosts/
├── host1/
│   └── configuration.nix
├── host2/
│   └── configuration.nix
```

The Taskfile dynamically maps hosts to:

```text
hosts/<host>/configuration.nix
```

and validates their existence before execution 

---

## 🚀 Getting Started

### Validate configuration

```bash
task eval
```

Checks syntax using `nix-instantiate` 

---

### Dry build (safe pre-check)

```bash
task check
```

Runs `nixos-rebuild dry-build` for all hosts 

---

### Deploy configuration

```bash
task switch
```

Aliases: `task deploy`, `task build`

Applies configuration locally or remotely depending on host 

---

### Test configuration (temporary)

```bash
task test
```

Runs `nixos-rebuild test` (non-persistent) 

---

### Update channels

```bash
task update
```

---

### Run arbitrary command

```bash
task cmd HOST CMD="df -h"
```

---

### Check system status

```bash
task status
```

---

### Git workflow

```bash
task push        # push repo
task pull        # pull on hosts
task push-pull   # both
```

---

## 🎯 Targeting Specific Hosts

```bash
task switch mylly portti
```

If no hosts are specified, all hosts are targeted.


## 🔐 Safe Remote Deployment Logic

One of the key design goals of this project is **safe remote management**.

The Taskfile implements a **multi-step deployment safety mechanism**:

### 1. SSH connectivity check

* Verifies host is reachable before deployment

### 2. Automatic rollback timer

A rollback reboot is scheduled:

```bash
shutdown -r +10
```

If something goes wrong, the machine will reboot into the previous generation.

### 3. Test deployment (`nixos-rebuild test`)

* Applies configuration temporarily
* Does **not** make it default

### 4. SSH re-check

* Ensures system is still reachable after applying config

### 5. Cancel rollback

If everything is OK:

```bash
shutdown -c
```

### 6. System health validation

```bash
systemctl is-system-running
```

* If **degraded → rollback is triggered**
* If **healthy → continue**

### 7. Final switch

```bash
nixos-rebuild switch
```

Only executed after all checks pass.


## 🧠 Why This Matters

This approach prevents:

* Locking yourself out via SSH
* Deploying broken configurations
* Leaving systems in degraded states
* Risky blind `switch` operations

It gives you a **safe, reversible deployment workflow without needing a full orchestration system**.


## 🆚 When to Use Something Else

Use tools like Colmena when:

* Managing **dozens or hundreds of hosts**
* Needing **parallel orchestration**
* Requiring **stateful deployments or secrets distribution at scale**


## 🧩 Design Philosophy

* Keep it simple and transparent
* Prefer shell + Taskfile over complex frameworks
* Optimize for **operator confidence and safety**
* Make failures visible and actionable


## 📝 Notes

* Uses SSH agent forwarding (`ssh -A`)
* Assumes Git-based configuration management
* Works without flakes (compatible with traditional Nix setups)
* Includes a small spinner utility for better CLI UX 

## 📄 License

[MIT](https://mit-license.org/)

