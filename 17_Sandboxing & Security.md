# 17_Sandboxing & Security

# M12 — Sandboxing & Security

> Filesystem/network isolation, OS-level enforcement, sandbox modes, security limitations, and best practices.

---

## Why Sandboxing Matters

Traditional permission-based security leads to approval fatigue, reduced productivity, and limited autonomy. Sandboxing creates **defined boundaries** where Claude works freely with OS-level enforcement.

> **Critical:** Effective sandboxing requires **both** filesystem AND network isolation. Without network: exfiltrate SSH keys. Without filesystem: backdoor system resources.

---

## Sandbox Architecture

**Filesystem Isolation**

| Behaviour | Default |
| --- | --- |
| Write access | Current working directory and subdirectories only |
| Read access | Entire computer (except denied directories) |
| Blocked | Cannot modify files outside CWD without explicit permission |

> OS-level enforcement: Seatbelt (macOS), bubblewrap (Linux/WSL2). Applies to ALL subprocesses including `kubectl`, `terraform`, `npm`.

**Network Isolation**

| Feature | How |
| --- | --- |
| Domain restrictions | Only approved domains accessible |
| New domain requests | Trigger permission prompts |
| Custom proxy | Advanced organisations can implement custom rules |
| Coverage | All scripts, programs, and child processes |

**OS-Level Enforcement**

| Platform | Technology |
| --- | --- |
| macOS | Seatbelt (built-in) |
| Linux/WSL2 | bubblewrap (`apt-get install bubblewrap socat`) |
| WSL1 | Not supported |
| Windows native | Planned |

---

## Getting Started & Two Modes

Enable with `/sandbox`.

| Mode | Behaviour |
| --- | --- |
| Auto-allow | Sandboxed Bash commands run automatically without permission. Non-sandboxable commands fall back to normal permission flow |
| Regular permissions | All Bash commands go through standard permission flow, even when sandboxed |

> **Important:** Auto-allow works independently of your permission mode. Even without "accept edits" mode, sandboxed bash commands execute without prompting.

---

## Configuration

**Grant subprocess write access:**

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "allowWrite": ["~/.kube", "/tmp/build"]
    }
  }
}
```

**Path prefix rules:**

| Prefix | Meaning | Example |
| --- | --- | --- |
| `/` | Absolute path | `/tmp/build` |
| `~/` | Home-relative | `~/.kube` → `$HOME/.kube` |
| `./` or no prefix | Project-relative (project settings) or `~/.claude`-relative (user settings) | `./output` |

**Deny paths:**

```json
{
  "sandbox": {
    "filesystem": {
      "denyRead": ["~/"],
      "allowRead": ["."],
      "denyWrite": ["~/.ssh"]
    }
  }
}
```

> `allowRead` takes precedence over `denyRead` for specific paths. When defined in multiple settings scopes, arrays are **merged** (not replaced).

---

## Security Protection

**Against prompt injection**

- **Filesystem:** Cannot modify `~/.bashrc`, `/bin/`, or denied paths.
- **Network:** Cannot exfiltrate data, download malicious scripts, or call unapproved domains.
- **Monitoring:** All access attempts outside the sandbox are blocked at OS level with immediate notifications.

**Against supply chain attacks** — limits damage from malicious npm packages, compromised build scripts, and social engineering.

**Escape hatch** — when a command fails due to sandbox restrictions, Claude may retry with `dangerouslyDisableSandbox`, which goes through the normal permissions flow.

```json
{ "sandbox": { "allowUnsandboxedCommands": false } }
```

---

## Sandboxing vs Permissions

| Layer | What It Controls | Scope |
| --- | --- | --- |
| Permissions | Which tools Claude can use | All tools (Bash, Read, Edit, WebFetch, MCP) |
| Sandboxing | What Bash commands can access (OS-level) | Bash commands and child processes only |

> They are complementary. Permissions evaluated **before** a tool runs. Sandbox enforced **during** execution.

---

## Security Limitations (Know These)

| Limitation | Risk |
| --- | --- |
| Network filtering | Domain-based only. Does not inspect traffic content |
| Broad domains | Allowing `github.com` could enable exfiltration |
| Domain fronting | Possible to bypass filtering in some cases |
| Unix sockets | `allowUnixSockets` with Docker socket grants host access |
| Filesystem writes | Overly broad `allowWrite` to `$PATH` dirs enables privilege escalation |
| Linux weak mode | `enableWeakerNestedSandbox` (for Docker) considerably weakens security |

---

## Tool Compatibility

| Tool | Sandbox Compatible? | Notes |
| --- | --- | --- |
| Standard CLI tools | Yes | `git`, `npm`, `python`, etc. |
| `kubectl`, `terraform` | Yes | May need `allowWrite` for config dirs and domain access |
| `watchman` | No | Use `jest --no-watchman` |
| `docker` | No | Add to `excludedCommands` |

---

## Network Configuration

**Custom proxy:**

```json
{
  "sandbox": {
    "network": {
      "httpProxyPort": 8080,
      "socksProxyPort": 8081
    }
  }
}
```

**Block non-allowed domains:**

```json
{
  "sandbox": {
    "allowManagedDomainsOnly": true
  }
}
```

> Non-allowed domains are blocked without prompting.

---

## Best Practices

1. **Start restrictive** — expand as needed.
2. **Monitor violations** — review sandbox violation attempts.
3. **Environment-specific configs** — different rules for dev vs production.
4. **Combine with permissions** — defence-in-depth.
5. **Test configurations** — verify legitimate workflows aren't blocked.
6. **Use managed settings** — enforce organisation-wide sandbox policies.

**Open source sandbox runtime:**

```bash
npx @anthropic-ai/sandbox-runtime <command-to-sandbox>
```

> Can sandbox any program — including MCP servers.