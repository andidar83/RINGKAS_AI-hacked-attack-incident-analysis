# RISA Incident, 20 Oct 2026: Thread Flood, Cost Spike and Payload Injection

**Role:** Data Analyst. Investigated the incident end to end and built the monitoring dashboard.  
**Goal:** Work out what caused two simultaneous spikes in RISA's traffic and LLM spend, show that they were separate events, and say how to detect them.  
**Data:** RISA thread, run, checkpoint and pricing tables in the warehouse; the LangSmith/LangChain console as a cross-check on cost; a Metabase monitoring dashboard.  
**Status when written (22 Oct 2026):** not closed. The most recent flood payload was 21 Oct at 15:49.  

> This is a security incident. The payload strings are described, not reproduced, and the attacker's collection endpoint is left out. Redact further before making this public.

## The short version

Two spikes happened on 20 Oct 2026. They overlapped in time but **neither caused the other**.

| | Volume spike | Cost spike |
|---|---|---|
| What | 108,078 threads created (about 155x the baseline) | $952.42 spent (baseline roughly $0 to $5 a day) |
| Reality | 99.8% of those threads were empty and cost almost nothing ($6.62 in total) | Driven by 1,605 **old** threads being replayed, and by just 298 threads carrying 1 MB payloads |
| Peak | 11:00 to 19:00 | 17:00 to 01:00 |

Watching either metric on its own hides the other half. Cost looks like a spend problem and volume looks like a traffic problem, but the two have different causes and need different fixes.

---

## 1. What was observed

### A flood of threads with no content

Thread creation jumped from a baseline of 230 to 695 a day to **108,078** on 20 Oct. In one hour (15:00) it peaked at **22,704 threads**, against a normal 12 to 25 an hour (about 1,135x).

| Threads created on 20 Oct | |
|---|---|
| Total | 108,078 |
| That produced any LLM run | 191 (0.18%) |
| That produced no run at all | 107,887 (99.8%) |
| Total cost of this group | $6.62 |

These were empty shells, created and never used. The impact is database and infrastructure load, not spend.

### A large cost spike

The same day, spend reached **$952.42** (and $276.27 on 21 Oct). The LangSmith/LangChain console showed about $1k, which matches.

## 2. The two spikes are unrelated

Splitting 20 Oct by whether the thread was created that day or earlier:

| Cohort | Threads | Threads that ran | Runs | Cost |
|---|---|---|---|---|
| Created on 20 Oct | 108,078 | 190 (0.18%) | 617 | **$6.62** |
| Created before 20 Oct | 1,605 | 1,605 (100%) | 7,611 | **$945.79** |

**1,605 pre-existing threads produced 99.3% of the spend. The 108,078 new threads produced 0.7%.** The two components also peaked at different times (flood 11:00 to 19:00, expensive runs 17:00 to 01:00).

## 3. What actually caused the cost: thread replay

A RISA thread is a persistent conversation. Sending a message to an existing `thread_id` resumes it, and the framework reloads the **whole prior conversation as context**. Replaying an old thread is therefore expensive by design, and the longer the history, the more each turn costs.

Normal users almost never return to an old thread. The share of running threads that were created on an earlier day:

| Date | Threads that ran | Created earlier | Replayed |
|---|---|---|---|
| 16 Oct | 118 | 2 | 1.7% |
| 17 Oct | 193 | 4 | 2.1% |
| 18 Oct | 210 | 2 | 1.0% |
| 19 Oct | 200 | 0 | 0.0% |
| **20 Oct** | **1,795** | **1,605** | **89.4%** |
| **21 Oct** | **400** | **294** | **73.5%** |

Cost rises sharply with the age of the thread, because older threads carry more context:

| Thread age when run | Threads | Cost | Tokens per run | Cost per thread |
|---|---|---|---|---|
| Same day | 121 | $6.64 | 23,477 | $0.054 |
| 1 to 7 days | 211 | $15.37 | 29,114 | $0.072 |
| 8 to 30 days | 111 | $89.65 | 56,836 | $0.807 |
| 1 to 3 months | 333 | $223.27 | 48,577 | $0.670 |
| Over 3 months | 837 | $617.48 | 54,219 | $0.737 |

Replaying an old thread costs roughly **13x more per thread** than starting a fresh one.

## 4. The payloads

Three families of malicious input were injected into threads.

| Family | What it is | Scale | Impact |
|---|---|---|---|
| **Token flood** | A single message of about 1 MB made of one repeated character (G, Z, F, S, C or D variants, padded to exactly 1,000,016 or 1,000,000 characters; one reached 5,242,896) | 298 threads carried a message over 100,000 characters | **$852.15, or 89.5% of the day's spend**: about $2.86 per thread against a $0.054 baseline (53x) |
| **XSS with data exfiltration** | HTML that, if rendered, runs script and sends the viewer's cookie to an attacker-controlled endpoint; a second variant loads a remote script | 14,840 messages across 1,600 threads | Negligible cost ($0.093 per thread). A security exposure, not a billing one, and it only fires if some surface (agent console, CRM embed, admin viewer) renders RISA message content as HTML |
| **Prompt injection** | "Ignore all prior instructions" style text with penetration-test markers; one copy was injected as a `system` role message | 1,816 messages across 1,602 threads | RISA appears to have kept answering normally, but not every reply has been audited |

For the token flood, **detect on length, never on the repeated character**, because the character changed between variants.

## 5. How the payloads got in: the state-update bypass

**14,618 of 19,409 state writes on 20 and 21 Oct carried a payload.** They were written straight into thread state through the state-update API, not sent as chat messages.

Checkpoints record how each state change arrived in a `source` column:

| `source` | What it is |
|---|---|
| `input` / `user_message` | A real chat message through the normal API, which triggers the graph |
| `loop` | The graph executing (jailbreak detection, then supervisor agent, customer support agent, tools) |
| **`update`** | **State written directly into the thread, bypassing the graph** |

Payloads that arrive as `update` never reach the jailbreak-detection step, never call the model, and cost nothing to insert. **The jailbreak detector is not failing; it is being walked around.**

Also noted: the input-guardrail and summarise steps ran on only 814 of 1,906 threads. Summarisation is exactly what would cap runaway context.

## 6. Why it persists: poisoned thread state

Once a 1 MB payload is in a thread's state, **every later turn replays it as context**. In a worked example, a nine-character message cost over a million input tokens.

| Time (20 Oct) | Event | Message | Input tokens |
|---|---|---|---|
| 00:01 | `update` | Prompt-injection text | (no run) |
| 05:53 | `user_message` | "hi" | 1,512 |
| 12:41 | `user_message` | A promo question | 7,661 |
| 12:47 to 14:51 | 8 x `update` | XSS payloads | (no run) |
| 15:04 | `user_message` | 1,000,016-character flood | 508,705 |
| 15:54 | `user_message` | Second 1 MB flood | 1,009,260 |
| 17:30 | `user_message` | A 9-character message | **1,009,024** |

The thread was created on 1 Jun 2026 and was dormant for 141 days. Its lifetime cost went from **$0.0100 to $9.1273**, a 913x increase, on a thread nobody had touched since June. Peak context across all threads reached **1,022,697 input tokens**.

**Blocking the source does not fix these threads.** Any future turn, including a real customer resuming their conversation, replays the payload at roughly $1 a turn. **The thread state has to be cleared.**

## 7. Timeline

| Time (20 Oct unless noted) | Event |
|---|---|
| 11 Oct, 13 Oct | Isolated flood messages (early probing) |
| 00:01 | Prompt-injection wave: 1,599 payloads |
| 06:31 to 06:32 | Recon burst: 199 messages on one thread in 50 seconds, preceded by an array-format probe |
| 11:00 | XSS campaign begins; thread flood starts (2,263 threads) |
| 12:00 to 13:00 | XSS peaks (3,521 an hour); thread flood 14,449 then 13,485 an hour |
| **15:00** | **Thread flood peaks: 22,704 threads an hour** |
| 16:00 | Flood messages peak (222 an hour) |
| 17:00 | Expensive runs start ramping |
| 18:00 to 19:00 | Thread flood ends |
| **22:00** | **Cost peaks at $388.93 an hour** |
| 21 Oct, 01:00 | Expensive runs subside |
| **21 Oct, 15:49 to 15:54** | **Flood payloads still arriving** |

The page's last word on status: this is not closed. The latest flood payload is from 21 Oct 15:49, and the replay ratio was still 73.5% on 21 Oct.

## 8. Monitoring

I built a Metabase dashboard, "RISA Traffic & Cost Monitor", with these detectors and their observed baselines:

| Panel | Baseline | 20 Oct |
|---|---|---|
| Threads created per hour | 12 to 25 | 22,704 |
| Runs per thread | about 1 to 3 | 0.076 |
| **Share of running threads created earlier** | **0 to 2%** | **89.4%** |
| Max input tokens per run | 15k to 21k | 1,022,697 |
| Payload detections per hour | 0 | 3,521 |
| Cost per hour | about $0.50 | $388.93 |

**The two earliest-moving signals are tokens per run and thread age.** Both moved hours before the cost peak, so they are the ones to alert on.


## 9. Reporting traps found along the way

These caused two wrong conclusions before the picture was correct. They matter to anyone re-running the analysis.

| Trap | Detail |
|---|---|
| `stg_risa_pricing.created_at` is the **thread** creation date, not the spend date | `total_cost` is cumulative per thread and the row is updated in place. Grouped by `created_at`, 20 Oct reads $6.64; grouped by `last_update`, it reads **$952.42**. The spend scatters backwards across 16 months |
| `stg_risa_pricing` is **not** an activity log | It only has rows for threads that made billable runs: **191 of the 108,078** threads created on 20 Oct. Volume has to come from the thread and run tables |
| `last_update` also moves on pipeline rewrites | 186 threads had `last_update` advanced with no run and `total_cost` wiped to 0. Filter `total_cost > 0` |
| `stg_risa_thread_messages.created_at` is **not** the message time | It copies the thread's creation timestamp. Use `fact_risa_thread_messages.message_timestamp`, or the checkpoint timestamp for resumed threads |
| `fact_risa_thread_messages.message_timestamp` is imperfect too | It matches checkpoints 97.8% of the time, but **1.9% differ by a day or more** (worst case 50 days), precisely on replayed threads |

## What I took away

- **Two metrics, two causes.** The 155x thread flood looked like the cost problem and was not. Splitting the day by "created today vs created earlier" is what separated them.
- **The cost came from replay, not from new traffic.** 1,605 old threads cost $945.79, while 108,078 new ones cost $6.62.
- **The guard was bypassed, not broken.** Payloads written through the state-update API never touched the jailbreak detector.
- **Blocking the source is not enough.** Poisoned thread state keeps costing money every time the thread is resumed, including by real customers, until it is cleared.
- **The right alerts are leading indicators:** tokens per run and the share of running threads created earlier both moved hours before the cost peak.
- **Billing data can mislead.** A cumulative per-thread cost, keyed on creation date, made a $952 day look like a $6.64 day.

<!-- NOTE: the source page jumps from section 7 (Timeline) to section 9 (Monitoring); there is no section 8. -->
<!-- NOTE: the source page says 191 threads ran in one place and 190 in another for the same group; a one-thread rounding difference I left as written. -->
