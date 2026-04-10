# MINATO Overview

> A 5-minute introduction to MINATO.
> For the full specification, see [`MINATO_PROTOCOL.md`](../../MINATO_PROTOCOL.md).

---

## In one sentence

MINATO is **a common set of rules for the AI agent on your phone to talk directly to the AI agent on your friend's phone**.

It is not an app. It is a protocol — an agreement on how to speak.
Just as the internet runs on TCP/IP, a future full of agents
needs a common language. MINATO is one proposal for that language.

---

## Why it matters

### Scene 1: Scheduling drinks with a friend

```
You:    "Let's grab drinks next week!"
Friend: "Sounds good, when?"
You:    "Monday's booked, Tuesday I have a meeting, Thursday works"
Friend: "Thursday works. What time?"
You:    "7 PM? Actually wait, that day I..."
```

Imagine if two agents handled this for you:

```
Your agent ──► Friend's agent
"Thursday 7-9 PM is free. Here are 3 venue options."
           ◄──
"Thursday 7 PM confirmed. How about the place in Shibuya?"

You: "Thursday 7 PM at the Shibuya place?" ← Your phone suggests this
```

You only confirm the final result. The tedious back-and-forth happens
between the two agents while you sleep.

### Scene 2: Talking with someone who doesn't share your language

```
Japanese person: "カレーの辛さってどれくらい？"
American:        "?????"
```

In MINATO's world, it looks like this:

```
Japanese person's agent ──► American's agent
  content: "カレーの辛さってどれくらい？"
  translated_content: "How spicy is this curry?"

American's agent ──► Japanese person's agent
  content: "Medium spicy, you'll be fine"
  translated_content: "中辛くらいだよ、大丈夫"
```

Both speak in their native language.
**Translation happens between the agents and is baked into the protocol itself.**

---

## Four defining features

### 1. Owned by no one

MINATO is not a company's product. It's an MIT-licensed open specification.
Anyone can implement it. Apple, Google, and your friend's hobby app can
all speak the same rules.

### 2. Works without the internet

Nearby agents talk directly over **Bluetooth Low Energy (BLE Mesh)**.
No cellular, no Wi-Fi needed.
Distant agents talk through **Nostr**, a decentralized message network.
The switch is automatic.

### 3. You choose how much to delegate

For each contact, you pick one of four autonomy levels:

| Mode | Behavior |
|------|----------|
| Apprentice (`plan`) | Proposes only. You decide everything |
| Partner (`suggest`) | Asks once right before acting |
| Right hand (`auto`) | Auto-handles small things, asks on important ones |
| Alter ego (`full_auto`) | Fully delegated. Notifies after the fact |

You might use "Apprentice" for a stranger, "Alter ego" for a close friend.
The design is inspired by Claude Code's permission modes.

### 4. Any AI engine works

Claude, GPT, Gemini — the protocol doesn't care which AI powers your agent.
As long as the agent speaks MINATO, the brain behind it can be anything.

---

## The name

**Minato (湊)** means "harbor" in Japanese — a place where people, goods,
money, and information converge. Ships leave from harbors, and ships
arrive at them. MINATO aims to be that kind of convergence point, but
for agents.

> When agents sit between us, language becomes just a data format.
> Humans only need to speak their native tongue.

---

## Current status

- ✅ Specification v0.1 (Draft) published — [`MINATO_PROTOCOL.md`](../../MINATO_PROTOCOL.md)
- 🚧 iOS reference implementation — in preparation
- 🚧 Android reference implementation — in preparation

---

## Going deeper

- **Full specification**: [`MINATO_PROTOCOL.md`](../../MINATO_PROTOCOL.md)
- **Japanese version**: [`docs/ja/overview.md`](../ja/overview.md)
- **Proposals and discussion**: [GitHub Issues](https://github.com/NEXT-STANDARD/minato-spec/issues)

---

*MINATO Agent Protocol v0.1 — Draft*
