# Leviathan: Linux privilege escalation

**OverTheWire · 7 / 7 levels ✅** · [← all write-ups](../../README.md)

Each level is a Linux account reached over SSH, and the goal is the next account's password in `/etc/leviathan_pass/`, which only that account can read. The site gives no hints on purpose; everything needed is on the machine. Most levels hide a **setuid binary**, a program that runs with its *owner's* privileges no matter who starts it, so each level comes down to finding what that program trusts.

**My loop for every level**

1. Enumerate: `ls -la` for hidden files, owners and the setuid bit.
2. Run the binary normally to learn its interface.
3. Trace it with `ltrace` to see library calls and their arguments (`strcmp`, `access`, `system`, `fopen`). The kernel ignores the setuid bit on a traced program, so tracing shows the logic without running anything privileged.
4. Form a hypothesis about what the binary trusts, exploit it, confirm with `whoami`, read the next password.

| Level | Finding | Vulnerability | CWE |
|---|---|---|---|
| 0 | Password in a hidden bookmarks backup | Credential stored in a backup file | [CWE-530](https://cwe.mitre.org/data/definitions/530.html) |
| 1 | setuid binary comparing input to a built-in string | Hard-coded credential | [CWE-798](https://cwe.mitre.org/data/definitions/798.html) |
| 2 | setuid file printer that hands filenames to the shell | OS command injection | [CWE-78](https://cwe.mitre.org/data/definitions/78.html) |
| 3 | Level 1's pattern plus a decoy comparison | Hard-coded credential | [CWE-798](https://cwe.mitre.org/data/definitions/798.html) |
| 4 | Binary printing the password as binary octets | Encoding mistaken for protection | — |
| 5 | setuid binary reading a fixed path in `/tmp` | Symlink following | [CWE-59](https://cwe.mitre.org/data/definitions/59.html) |
| 6 | setuid shell behind a 4-digit code | Unlimited guesses on a tiny keyspace | [CWE-307](https://cwe.mitre.org/data/definitions/307.html) |

## Level 0: secrets in a backup

`ls` showed nothing, but `ls -a` revealed a `.backup` directory holding a browser `bookmarks.html`. Among hundreds of links, one bookmark's title contained the next password.

- **Root cause:** a credential stored in plain text, in a file nobody thinks of as sensitive.
- **Fix:** credentials belong in a secrets store; backups need the same access control as the data they copy.
- **Next time:** `grep -i password` instead of scrolling through the file.

## Level 1: a hard-coded password, visible to `ltrace`

The `check` binary asks for a password. Under `ltrace`, my input and the real password show up side by side as the arguments to `strcmp` (trimmed, password redacted):

```text
$ ltrace ./check
printf("password: ")                     = 10
getchar(...)                             = ...   # my input, one character at a time
strcmp("<my input>", "<redacted>")       = -1    # the real password, in clear text
puts("Wrong password, Good Bye ...")     = 29
```

Typing the real password opened a shell as `leviathan2`.

- **Root cause:** a secret compiled into the program. Anything a program compares against, a tracer or debugger can read.
- **Fix:** don't gate privileges on a secret inside the binary. Use OS permissions, or at least compare against a hash stored where the user can't read it.

## Level 2: one string, two interpretations

`printfile` prints a file you name. Being setuid `leviathan3`, it should refuse files you can't read yourself. `ltrace` showed how it decides, which I rewrote as C:

```c
// Reconstructed from ltrace output
if (access(path, R_OK) == 0) {               // permission check as the real user
    snprintf(cmd, 511, "/bin/cat %s", path); // path pasted into a shell command
    setreuid(geteuid(), geteuid());          // become leviathan3
    system(cmd);                             // the shell parses the string again
}
```

The two halves disagree about what `path` is. `access()` treats it as one filename; the shell splits it at metacharacters. So I created a file in `/tmp` whose *name* contained a command separator followed by a command. `access()` approved it, since I owned the file. The shell ran `cat` on the part of the name before the separator, which failed, and then ran my command as `leviathan3`.

- **Root cause:** input concatenated into a shell command, validated under a different interpretation from the one that runs.
- **Fix:** a program that only prints files the caller can already read doesn't need to be setuid at all. If it must be, it should open the file with the caller's privileges and never involve a shell: `execve()` with an argument array, not `system()`. Checking with `access()` and opening later is also a race in its own right ([TOCTOU, CWE-367](https://cwe.mitre.org/data/definitions/367.html)).

## Level 3: the same flaw, plus a decoy

`level3` asks for a password. `ltrace` showed two `strcmp` calls: one before the prompt comparing two constants, which affects nothing, and one comparing my input to the real password. Entering it printed `[You've got shell]!` and dropped me into a shell as `leviathan4`.

- **Lesson:** a decoy comparison costs an attacker seconds. The fix is the same as level 1.

## Level 4: encoding is not protection

A hidden `.trash/bin` printed eleven 8-bit binary groups. Decoding them as ASCII in CyberChef (*From Binary*) gave ten characters and a newline: the next password.

- **Lesson:** binary, hex and Base64 are representations, not encryption. If decoding needs no key, the data is effectively plain text.

## Level 5: following a symlink

`leviathan5`, setuid `leviathan6`, printed `Cannot find /tmp/file.log`, and `ltrace` confirmed it opens that fixed path and prints the contents. `/tmp` is writable by everyone (listing it was blocked, but creating files wasn't), so I created `/tmp/file.log` as a **symbolic link** to the next password file. The binary followed the link with its elevated privileges and printed the password.

- **Root cause:** a privileged program opening a predictable path in a shared, writable directory, and following links.
- **Fix:** no fixed filenames in shared directories (`mkstemp()`), refuse links (`O_NOFOLLOW`), and drop privileges before touching any path a user can influence.

## Level 6: a 4-digit code with no lockout

`leviathan6` wants a 4-digit code. `strings` showed `/bin/sh`, `setreuid` and `atoi`, but no code, and decoding a few odd-looking strings in CyberChef led nowhere. With only 10,000 possibilities and no delay between attempts, a short bash loop over `0000`–`9999` found the code and opened a shell as `leviathan7`.

- **Root cause:** a tiny keyspace with unlimited, instant retries.
- **What I'd do differently:** `atoi` in the imports suggests the input is compared as an integer, which would sit inside the machine code where `strings` can't see it. Disassembling the comparison with `objdump -d` or GDB should reveal it directly. Brute force was just quicker to write.

## Level 7

A `CONGRATULATIONS` file, and a request not to publish solutions, which is why these write-ups describe vulnerabilities rather than answers.

## Takeaways

- `ltrace` turns a black-box binary into a readable list of decisions. It gave away the logic of four of the six binaries.
- A setuid program is only as safe as everything it trusts: input strings, filenames, paths in shared directories, the shell.
- The fix is almost never to hide something better. It's to remove the trust: no shell, no secrets in the binary, no fixed paths in `/tmp`, no unlimited guesses.
