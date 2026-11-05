# Capture the Flag

Write-ups of the wargames and CTFs I play. Every write-up covers what I observed, how I got in, and **why the bug exists and how I'd fix it**. The fix is the part that carries over to real systems.

> Flags, passwords and copy-paste payloads are left out on purpose. See the [spoiler policy](#spoiler-policy).

## Scoreboard

| Category | Wargame | Focus | Flags | Write-up |
|---|---|---|---|---|
| [Linux](Linux/) | Leviathan · OverTheWire | Privilege escalation via setuid binaries | **7 / 7** ✅ | [Leviathan](Linux/Leviathan/README.md) |
| [Binary exploitation](Binary-Exploitation/) | Narnia · OverTheWire | Memory corruption | **2 / 9** | [Narnia](Binary-Exploitation/Narnia/README.md) |
| [Web](Web/) | Natas · OverTheWire | Information disclosure, injection, weak crypto | **12 / 35** | [Natas](Web/Natas/README.md) |
| [LLMs](LLMs/) | Gandalf · Lakera | Prompt injection | **8 / 8** ✅ | [Gandalf](LLMs/Gandalf/README.md) |
| [LLMs](LLMs/) | HackMerlin | Prompt injection | **6 / 7** | [HackMerlin](LLMs/Merlin/README.md) |

## Highlights

**Forging an encrypted cookie** ([Natas 11](Web/Natas/README.md#level-11-xor-encrypted-cookie)). The session cookie was JSON XOR-ed with a short repeating key. I knew the plaintext of my own default cookie, so XOR-ing it with the ciphertext gave me the key, and with the key I could build a cookie the server accepted as its own.

**One string, two interpretations** ([Leviathan 2](Linux/Leviathan/README.md#level-2-one-string-two-interpretations)). A setuid "file printer" approved my filename with `access()`, then pasted it into a `system()` call. The permission check saw a file I owned; the shell saw a filename plus a second command, which ran with the binary's privileges. I reconstructed the program's logic from `ltrace` output alone.

**A chatbot as a yes/no oracle** ([HackMerlin 6](LLMs/Merlin/README.md#level-6-structural-questions)). Merlin's filters blocked the password in any case and even reversed. Questions about what the word *meant* got confident nonsense, so I asked about its *structure* instead (length, letters) and narrowed a dictionary down to fourteen candidates.

**Bytes, not text** ([Narnia 0](Binary-Exploitation/Narnia/README.md#level-0-overflowing-into-a-neighboring-variable)). My first overflow typed the target value as text, and the program reported `val: 0x65647830`, which is the ASCII for `0xde` read backwards. Fixing it meant writing raw bytes in little-endian order.

## Skills

| Area | Concepts | Tools |
|---|---|---|
| Linux & binaries | setuid privilege model, dynamic tracing, command injection, symlink attacks, brute force | `ltrace`, `strings`, `ln`, bash |
| Memory corruption | stack layout, buffer overflows, byte order, shellcode, executable vs. non-executable memory | C source review, `xxd` |
| Web | information disclosure, client-side trust, header and cookie tampering, path traversal, command and argument injection, weak cryptography | Firefox DevTools (Network → *Edit and Resend*, Storage), CyberChef |
| LLM security | prompt injection, system-prompt leakage, filter evasion, inference through yes/no answers | structured probing, word-finder tools |

## What 35 flags taught me

1. **The client is the attacker.** Everything the browser receives can be read, and everything it sends can be rewritten: HTML comments, JavaScript restrictions, the `Referer` header, cookies. *(Natas 0, 1, 4, 5, 11)*
2. **A secret can be reached from wherever it's stored.** I recovered passwords from a bookmarks backup, from string comparisons inside binaries, from an include file served as text, and from LLM system prompts. If a component can read a secret, whoever controls that component's input has a path to it. *(Leviathan 0, 1, 3 · Natas 0, 6 · every LLM level)*
3. **Encoding is not encryption, and encryption is not integrity.** Binary, Base64, hex and string reversal are one step from the original. Repeating-key XOR falls to a single known plaintext, and nothing stopped me from editing the data. *(Leviathan 4 · Natas 8, 11)*
4. **Blocklists lose.** A filter on `; | &` didn't stop argument injection, a word blocklist didn't stop paraphrase, and an LLM classifier didn't stop a request that never mentioned the password. Structural fixes work: no shell, no secret in the model's context. *(Natas 10 · Gandalf 5–6 · HackMerlin 5)*
5. **The check and the use must see the same thing.** `access()` judged a filename that the shell then split in two; a setuid program trusted a fixed path in world-writable `/tmp`. *(Leviathan 2, 5)*
6. **Every answer is a channel.** A model that won't say the password will still describe it, classify its characters or answer yes/no about it, and enough answers add up to the secret. *(Gandalf 7 · HackMerlin 3–6)*
7. **Error messages are data.** Gandalf's refusals were worded differently depending on which defense fired, and one level's URL (`/word-blacklist`) named its defense outright. Reading failures carefully told me what to try next.

## Still open

- **Narnia 2 onward.** I found where the overflow starts crashing the program but haven't yet turned the crash into control over execution. Binary exploitation is where I have the most to learn; next on my list are GDB, stack-frame layout, and how stack canaries, NX and ASLR change the approach.
- **HackMerlin 7.** It answers "Cannot say" to nearly everything. My notes point to an input classifier that reacts to sentence style, which is the hypothesis I'll test next.
- **Natas 12 to 34.** Continuing in my own time.

## Spoiler policy

OverTheWire asks players not to publish solutions, so these write-ups explain vulnerabilities, reasoning and fixes rather than answers. Every flag and password is removed, and exploits are described in prose rather than pasted. All testing was against platforms built to be attacked.