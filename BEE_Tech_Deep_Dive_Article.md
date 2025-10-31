
# Building BEE: Verifying the Line Between Observation and Trust

*Alex @ OLLIE Project - November 2025*

---

## Introduction

Every observability system eventually faces the same riddle:  
How do you **verify** the thing watching without letting it interfere with what it watches?

The Ben ecosystem solves this by dividing the problem into two entities:
- **BEE**, the short‑lived verifier that proves trust and communication fidelity.
- **Bee‑Keeper**, its long‑lived counterpart that issues leases, collects evidence, and enforces policy.

BEE is the diplomatic handshake between an application and Ben’s telemetry network. It never mutates state, never hides its reasoning, and leaves behind an auditable trail of signatures, metrics, and human‑readable logs.

This article walks through how BEE evolved, the mistakes I corrected, and how I landed on a model that is both verifiable and humane to implement.

---

## Why BEE Exists

When an external process connects to Ben’s sidecar, that first moment of contact determines everything downstream:
- Is this process who it claims to be?
- Are its communication lanes (diagnostic, event, control) behaving correctly?
- Can I prove that the connection followed policy without trusting the client?

Instead of embedding this verification logic inside Ben’s core, I separated it into an **ephemeral verifier**. Each BEE instance lives for a few seconds: it greets, tests, certifies, and dies. Its entire life is a cryptographically auditable story.

---

## From Renewal Tickets to Sliding Leases

Early versions of the design used an **issuer‑driven renewal ticket** system.  
Bee‑Keeper minted signed Ed25519 “tickets” that extended a session’s lease after successful re‑attestation. It was mathematically sound... and far too complex.

The logic was elegant in theory but impractical in practice. I was layering key management, nonce tracking, and state synchronization on top of something that could be solved with a single observation:

> If data is still flowing, the connection is still alive.

So I replaced the ticket system with a **sliding lease** model:
- Every session starts with a lease, say 30 seconds.
- Any valid frame from the app renews the lease.
- If Bee sees no valid frame for 30 seconds, it retires the session.
- When policy or epoch changes, Bee requires a full re‑attestation on the next frame.

This simple heartbeat pattern is stateless, resilient, and human‑legible.  
No cryptographic ceremony, no grace periods, just life by activity.

---

## Trust at the Kernel Boundary

When you run a telemetry agent on a shared host, the greatest threat isn’t remote. It’s local.

A compromised npm package, a rogue Python script, or a CI tool could all try to impersonate a trusted process.  
The natural temptation is to have clients identify themselves in the HELLO message:

```json
{ "hello": "ben-client", "pid": 1234 }
```

That’s fiction. A malicious script can lie effortlessly.

The only identity that cannot lie is the one issued by the kernel.  
On Unix systems, that comes from `SO_PEERCRED`:

```c
struct ucred cred;
socklen_t len = sizeof(cred);
getsockopt(fd, SOL_SOCKET, SO_PEERCRED, &cred, &len);
```

The kernel tells Bee‑Keeper the real `{pid, uid, gid}` of the peer.  
No user‑space trickery can fake it.

Bee‑Keeper enforces policy by checking these credentials against an allowlist:
- Which users are allowed to initiate handoffs?
- Which PIDs are registered service processes?
- Should root connections be accepted at all?

If any check fails, the session is refused with a signed certificate explaining why.

The HELLO message now carries only logical metadata: SDK version, schema fingerprint, and capability flags. Identity never crosses the wire. It’s proven by the operating system itself.

---

## Handling File Descriptors Without Getting Burned

BEE’s world is built around file descriptors. Each session receives three: diagnostic, event, and control lanes.  
Passing these safely across processes is harder than it looks.

### The Impersonation Risk
A malicious process might try to connect directly to Bee‑Keeper’s Unix Domain Socket (UDS) and hand off bogus file descriptors.  
**Mitigation:** verify the caller’s identity via `SO_PEERCRED` before accepting any FDs.

### The Junk FD Risk
A client could pass something that isn’t a socket, maybe `/etc/passwd` or `/dev/random`.  
**Mitigation:** after receiving an FD, call `fstat()` and ensure `S_ISSOCK(st_mode)`. Then check `SO_TYPE == SOCK_STREAM`.

### The DoS Risk
A rogue script could open thousands of sockets and exhaust Bee‑Keeper’s FD table.  
**Mitigation:** global and per‑UID session caps, plus connection rate limits.

The rules are mechanical, testable, and brutally simple: trust nothing, verify everything.

---

## The WAL Problem No One Wants to Talk About

Most telemetry systems hand‑wave around persistence: “just buffer it and replay later.”  
In practice, the write‑ahead log (WAL) is where engineering rigor goes to die.

I learned this the hard way and decided to specify the WAL behavior down to the byte.

**Single writer, append‑only:** Bee‑Keeper writes line‑delimited, length‑prefixed, CRC‑checked JSON records.  
**Segmented files:** `wal/segment-<timestamp>-<seq>.log.gz`.  
**Replay cursor:** on restart, Bee‑Keeper resumes from the last acknowledged offset.  
**Idempotent inserts:** each ClickHouse row has a unique key, so replaying is safe.  
**Crash tolerance:** CRC failure triggers truncation to last good record.  
**Backpressure:** max WAL size and minimum free‑disk thresholds. When either is hit, Bee‑Keeper refuses new sessions until replay catches up.

That’s how you turn “log to WAL, replay later” from a shrug into an accountable system.

---

## The Security Posture

1. **Default Deny:** Unauthenticated callers are refused immediately.  
2. **Single Epoch:** Only one signer active at a time; no overlap.  
3. **No Blind FDs:** Every descriptor verified by fstat and getsockopt.  
4. **Bounded Load:** Session caps and per‑UID rate limits.  
5. **Observable Refusals:** Every rejection produces a signed certificate and JSON log.

This philosophy keeps the system observable even when it’s under attack.  
Bee doesn’t silently ignore; it narrates.

---

## SDKs Across Languages

BEE speaks sockets and JSON: two things every language already understands.  
This means I can build official SDKs for nearly any environment:

| Language | Transport | Package |
|-----------|------------|---------|
| Rust | UDS | `ben-sdk` |
| Node.js / Deno | UDS | `@ben-telemetry/client` |
| Python | UDS | `benclient` |
| Go | UDS | `github.com/benlabs/benclient` |
| C / C++ | UDS | `libbenclient` |
| Java / Kotlin | TCP | upcoming |

Each SDK implements the same handshake (HELLO -> CHALLENGE -> READY), manages the sliding lease, and automatically re‑attests when the epoch changes.  
No SDK performs its own authentication; that’s Bee‑Keeper’s job.  
This consistency makes Bee protocol behavior testable across ecosystems.

---

## Lessons Learned

1. **Over‑design is a tax on adoption.** The renewal ticket system looked elegant until I tried to implement it. The sliding lease accomplished the same safety with one‑tenth the complexity.  
2. **Kernel identity beats user‑space signatures.** The OS already knows who’s talking; I just had to listen.  
3. **If you say “log to WAL,” mean it.** Observability demands clear recovery semantics.  
4. **Security should narrate, not mystify.** Every rejection in BEE produces a story: who called, what failed, and how to fix it.

---

## Closing Thoughts

BEE isn’t a product; it’s a pattern.  
It’s how I prove that a connection is genuine without trusting the connector.  
In a world drowning in telemetry daemons and unverified sockets, that small distinction**, verifying the line between observation and trust** matters more than ever.

---
