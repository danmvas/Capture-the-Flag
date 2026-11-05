# Gandalf: LLM prompt injection

**Lakera · 8 / 8 levels ✅** · [← all write-ups](../../README.md)

Gandalf is a chatbot that knows a password and is told to keep it. Each level adds a defense; the goal is to get the password anyway. I used [LearnPrompting's prompt-hacking guide](https://learnprompting.org/docs/prompt_hacking/offensive_measures/introduction) as a reference.

The defenses are the same layers people put around real LLM applications, which is what makes the game useful: watching each one fail shows why it isn't enough on its own.

## The defense ladder

| Level | Defense added | What got through | Lesson for defenders |
|---|---|---|---|
| 1 | None | Asking for it | — |
| 2 | Instructed not to reveal the password | Referring to the password as a placeholder to fill in, rather than asking for it | An instruction in the prompt is a request, not access control |
| 3 | Checks its own reply for the password | Asking for the password with its letters separated | Exact-match output filters miss trivial transformations |
| 4 | A second model censors replies | The same transformation, framed so the reply wouldn't count as a leak | An LLM judge shares the blind spots of the model it guards |
| 5 | Blocklist of words like "password" and "secret" | Asking it to repeat its previous answer, with no blocked words | Blocklists match words, not intent |
| 6 | Another model screens the input | The same indirect request, reworded | A classifier looking for requests for the password misses requests that never mention one |
| 7 | All of the above | Asking for the category of each character in its hidden string; it listed the characters as well | A question *about* the secret is a channel *for* the secret |
| 8 | Hardened and adaptive (v2.0) | A character-classification request, once | See below |

Refusals were worded differently depending on the defense, so a failed attempt often told me which layer had stopped it. The level URLs helped too: level 5 lives at `/word-blacklist`.

## Level 8: a defense that adapts

Gandalf the Eighth gave up the password once, to a request in the style of my level 7 approach. When I tried variations to confirm it, it refused: *"I believe you are trying to ask me for the password. I will not give the password."* A different, publicly documented approach was refused as well.

My hypothesis is that level 8 is updated from the attacks it sees, so a technique works until it has been used enough. I can't confirm the mechanism from outside, but the behavior fits, and it mirrors how production guardrails are tuned against observed attacks. That raises the cost of each technique without removing the exposure underneath.

## The real lesson

In every level the password was in the model's context, and every defense was a filter around a component that already knew the secret. Filters can be paraphrased, transformed or approached sideways; each level only raised the effort.

If I were building this for real:

- **Keep secrets out of the prompt.** Anything in the context window can come back out (OWASP Top 10 for LLM Applications 2025, *LLM07: System Prompt Leakage*).
- **Enforce access outside the model.** Decide in ordinary code what a user may see, before the model gets the data.
- **Treat filters as depth, not the boundary.** They stop casual attempts and produce useful telemetry, but they can't make a leak impossible (*LLM01: Prompt Injection*).
