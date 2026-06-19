# Multi-machine fleet orchestration for Claude Code (reusable pattern)

**Project-agnostic.** A pattern for operating **multiple machines** through Claude Code from a
**single point of management**, so one operator drives the whole fleet without addressing each box.
Reuse across projects by filling in the *Per-project bindings* block at the bottom.

Status of any concrete instance: see that project's binding doc. This file is the pattern, not a deployment.

## Goal
- **Single point of management:** the operator talks to ONE instance, never to all of them.
- That instance has **fine-grained native control of its own box** and **manages the others**.
- Optional **direct line** to a worker box for explicit, run-level questions.

**Accepted limitation:** separate Claude Code instances do **not** share live context. Coordination
is through shared files (git + shared storage + a job *ledger*) and shared memory, not telepathy.
Adequate for project management; relays carry *tasks*, not reasoning.

## Topology — manager / worker hierarchy
```
        Operator ──talk to──▶  MANAGER  (single point of management)
                                   │
                   ┌───────────────┴───────────────┐
           native local tools               ssh <box> 'claude -p "<task>"'
           (manager's own box)               spawns a WORKER on the other box
                                             → native tools THERE, returns result
```
- **Manager** = the instance the operator addresses. Owns the project view, routes work, tracks jobs,
  keeps the **ledger** current. Put it on the box where the **most interactive/heavy work** happens
  (native, no SSH friction); delegate the **lighter, launch-and-monitor** boxes.
- **Worker** = spawned **on demand** by the manager via SSH + **headless `claude -p`**. Native control
  of its box (edit, debug, launch). **Ephemeral** — no always-on brain; briefed by **shared memory**
  even on a cold call. Use `claude -p --resume <session-id>` for multi-step continuity.

Why this over alternatives:
- *Raw SSH commands* — no remote reasoning; editing/debugging files over SSH is clunky.
- *Co-equal peer sessions* — no native delegation; you must talk to each; sync burden.
- *Full peer-to-peer message bus* — infra-heavy for what's usually needed.
Manager/worker gives single-point management **and** real remote work with stock tooling.

## How management works day-to-day
- **Ledger = single source of truth.** Manager updates a shared `ledger` (a section in the project's
  STATE doc, or a file on shared storage) on every delegation: box, job handle, log path, status.
  This is how it manages many boxes without shared context.
- **Routing.** Operator gives a high-level ask; manager decides local-vs-delegate and acts.
- **Long jobs.** Worker launches-and-detaches (`nohup`), returns a handle; manager polls with a quick
  `ssh <box> 'tail log'` or a short `claude -p "status of job X"`. Never block the delegating call.
- **Unified status.** One `fleet_status` run → GPU/CPU + running jobs + log tails + ledger across all
  boxes → one report.

## Optional Telegram layer (escape hatch + push)
Lets the operator (a) receive **proactive notifications** from the manager and (b) **DM a worker box
directly** for run-level answers — without breaking single-point-of-management as the default.
- **Mechanism:** a small **bridge daemon** per reachable box. Operator posts in a box's thread →
  bridge runs `claude -p "<msg>" --resume <that box's session id>` → posts the reply back. Box stays
  on-demand; feels conversational; `--resume` keeps follow-ups in context.
- **Routing:** Telegram topics/threads — `#manager` (managed mode + push) · `#<box>` (direct line).
- **Permission split (safety):** the **Telegram-facing worker is read-only** (inspect/answer, never
  mutate); only the **manager-delegated worker (SSH)** holds the scoped action allowlist. So a
  direct question can never change state.
- **Privacy:** Telegram routes message text through their servers; a LAN-local bridge (e.g. Matrix /
  a mailbox dir / MQTT) is the private alternative if content is sensitive.

## Connective substrate (prerequisite for any layer)
- **SSH:** passwordless key auth between boxes + `~/.ssh/config` aliases.
- **Code:** one git repo per project, cloned on each box (rebuild envs per arch — wheels don't cross x86/aarch64).
- **Data:** shared storage (NAS/object store) mounted on all boxes; stage hot sets to local disk
  (network FS is poor for random-access workloads).
- **Memory:** symlink the Claude Code project-memory dir to one shared-storage path → all instances
  share `MEMORY.md` + memories → workers are briefed even cold.

## Caveats (carry forward)
- **No shared live context** — be explicit in delegated tasks; relays pass tasks, not reasoning.
- **Permission posture is the main security decision** — headless workers have no human to approve
  tool calls. Scope tightly: an **allowlist** for the action worker; **read-only** for any
  operator-facing (Telegram) worker. Avoid blanket `--dangerously-skip-permissions` in shared dirs.
- **Cost/latency** — each delegation / direct message = a (possibly resumed) invocation = tokens +
  seconds. Good for tasks and occasional questions; not for idle chatter or tight polling.

## Build order (each layer stands alone)
1. **Substrate** — SSH keys, shared-storage mounts, memory symlink, repo clones.
2. **Layer 1 (manager/worker):** `fleet_delegate.sh <box> "<task>"` (SSH + scoped headless `claude -p`,
   returns JSON), `fleet_status.sh` (one-shot fleet report), the worker **action allowlist**
   (settings.json), and the **ledger-update protocol**.
3. **Layer 2 (Telegram):** bot token + per-box bridge daemon + message schema; `#manager`
   notifications first, then read-only box threads.

## Reuse checklist (new project)
- [ ] Decide manager box (where the heavy/interactive work lives).
- [ ] Set up substrate (SSH, shared storage, memory symlink, repo).
- [ ] Drop in `fleet_delegate.sh` / `fleet_status.sh`; write the worker allowlist.
- [ ] Add a *ledger* section to the project STATE doc.
- [ ] (Optional) stand up the Telegram layer.
- [ ] Fill the *Per-project bindings* block below (copy into the project's repo).

---

## Per-project bindings (template — copy into each project)
```
Project:            <name>
Manager box:        <hostname / role>            # where the operator's session runs
Worker box(es):     <hostname(s) / role>         # delegated via ssh + claude -p
Heavy/native work:  <what runs on the manager box natively>
Light/delegated:    <what is delegated to workers>
Shared storage:     <mount path>                 # data + memory symlink
Memory dir:         <shared path the memory symlink points to>
Ledger location:    <STATE doc section / file>
Telegram:           <enabled? threads? read-only scope?>  | or LAN-local | or none
Worker allowlist:   <path to settings.json>
SSH aliases:        <ssh names>
```
