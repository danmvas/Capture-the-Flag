# Natas: web application security

**OverTheWire · 12 / 35 levels** · [← all write-ups](../../README.md)

Each level is a small website behind HTTP basic auth. The password for the next level sits in `/etc/natas_webpass/` on the server, readable by the current level's web user, and each site hides a way to reach it. I used Firefox DevTools and CyberChef throughout.

| Level | Vulnerability | Root cause | CWE |
|---|---|---|---|
| 0 | Password in an HTML comment | Secret sent to the browser | [CWE-615](https://cwe.mitre.org/data/definitions/615.html) |
| 1 | Same, with right-click disabled | Client-side "protection" | [CWE-602](https://cwe.mitre.org/data/definitions/602.html) |
| 2 | Directory listing exposes a users file | Auto-indexing on a public folder | [CWE-548](https://cwe.mitre.org/data/definitions/548.html) |
| 3 | `robots.txt` points to a "hidden" folder | Security through obscurity | [CWE-656](https://cwe.mitre.org/data/definitions/656.html) |
| 4 | Access decided by the `Referer` header | Trusting a client-controlled header | [CWE-293](https://cwe.mitre.org/data/definitions/293.html) |
| 5 | Login state in a `loggedin` cookie | Authentication state on the client | [CWE-565](https://cwe.mitre.org/data/definitions/565.html) |
| 6 | Secret in an include file served as text | Include file inside the web root | [CWE-552](https://cwe.mitre.org/data/definitions/552.html) |
| 7 | `?page=` displays any file | File path taken from user input | [CWE-22](https://cwe.mitre.org/data/definitions/22.html) |
| 8 | Reversible "encoded" secret | Encoding instead of hashing | [CWE-261](https://cwe.mitre.org/data/definitions/261.html) |
| 9 | Search input reaches the shell | OS command injection | [CWE-78](https://cwe.mitre.org/data/definitions/78.html) |
| 10 | Same search, behind a character blocklist | Argument injection | [CWE-88](https://cwe.mitre.org/data/definitions/88.html) |
| 11 | XOR-"encrypted" preferences cookie | Weak cipher, no integrity check | [CWE-327](https://cwe.mitre.org/data/definitions/327.html) |

## Levels 0–3: information disclosure

- **0, HTML comment.** The page says the password is "on this page". It's in an HTML comment, visible in the DevTools inspector.
- **1, right-click blocked.** JavaScript disables the context menu; F12 opens DevTools anyway. Client-side restrictions are UX, not access control.
- **2, directory listing.** The page embeds `files/pixel.png`. Opening `/files/` returns Apache's auto-generated index, which includes a `users.txt` with the next credentials. *Fix:* turn off `Options Indexes` and keep data files out of the web root.
- **3, robots.txt.** A comment says "Not even Google will find it this time". `robots.txt` tells crawlers which paths to skip, and in doing so lists them for everyone. It pointed to a folder with another `users.txt`. *Fix:* protect sensitive paths with authentication, not by asking crawlers to look away.

## Levels 4–5: trusting the client

- **4, Referer.** The page says authorized users come only from the natas5 URL, and echoes back my real `Referer`. In the Network tab I used **Edit and Resend** to change that header; the response contained the password.
- **5, cookie.** "You are not logged in", alongside a cookie `loggedin=0`. Resending the request with the value flipped to `1` was enough.

*Root cause for both:* an authorization decision based on data the client fully controls. *Fix:* keep session state on the server behind an opaque session ID, or sign anything stored client-side so tampering is detected.

## Level 6: an include file served as text

The page's source shows `include "includes/secret.inc"`. The server runs `.php` files through PHP but returns `.inc` files as plain text, so requesting the include directly showed the secret the form was waiting for.

*Fix:* keep includes outside the web root, or deny direct requests to them.

## Level 7: arbitrary file read through `?page=`

Navigation uses `index.php?page=home` and `?page=about`, and an HTML comment gives the path of the next password file. Putting that path in `page` displays the file.

*Fix:* map `page` against an allowlist (`home`, `about`) instead of treating it as a path.

## Level 8: encoding is not hashing

The source compares my input against a stored value built like this:

```php
bin2hex(strrev(base64_encode($secret)))
```

All three steps are reversible, so I applied the inverses in reverse order in CyberChef (*From Hex* → *Reverse* → *From Base64*), recovered the secret and submitted it.

*Fix:* store a real password hash (`password_hash()` / `password_verify()`), which can't be run backwards.

## Level 9: command injection

```php
passthru("grep -i $key dictionary.txt");   // $key comes straight from the search box
```

The search term is pasted into a shell command. Ending the `grep` with a command separator and appending my own command to read the next password file worked.

## Level 10: argument injection past a blocklist

The same search, now rejecting any input that contains `;`, `|` or `&`:

```php
if (preg_match('/[;|&]/', $key)) {
    print "Input contains an illegal character!";
} else {
    passthru("grep -i $key dictionary.txt");
}
```

I didn't need a second command, because `grep` already reads files. Words separated by spaces become **separate arguments**, so a pattern that matches every line, followed by the path of the password file, made `grep` print that file along with the dictionary.

*Fix for levels 9 and 10:* don't build shell commands from input. Escaping turns the input into exactly one argument, and `--` stops it from being read as an option:

```php
passthru("grep -i -- " . escapeshellarg($key) . " dictionary.txt");
```

Better still, skip the shell: read the dictionary in PHP and filter it with `stripos()`. A blocklist has to anticipate every character an attacker might use; escaping takes away the shell's chance to interpret any of them.

## Level 11: XOR-encrypted cookie

The page keeps a background-color preference in a cookie:

```php
setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
```

`$d` defaults to `{"showpassword":"no","bgcolor":"#ffffff"}`, the password is only printed when `showpassword` is `yes`, and the key is censored in the published source.

XOR undoes itself: if `ciphertext = plaintext ⊕ key`, then `ciphertext ⊕ plaintext = key`. I knew the plaintext of my own cookie (the default JSON), so:

1. Base64-decode the cookie and XOR it with the default JSON. Out comes a short key, repeated.
2. XOR the edited JSON, with `"showpassword":"yes"`, against that key and Base64-encode the result.
3. Resend the request with the new cookie. The page printed the password.

*Root cause:* repeating-key XOR gives up its key to one known plaintext, and nothing checks the cookie's integrity, so an edited cookie is accepted ([CWE-565](https://cwe.mitre.org/data/definitions/565.html) as well). *Fix:* keep preferences on the server behind a session ID, or sign the cookie with an HMAC (or use authenticated encryption) so the server rejects anything it didn't issue.

## Takeaways

- When the source is offered, read it. Levels 6 and 8–11 fell to understanding the code.
- Most of these bugs are trust bugs. Anything the server reads from the browser is not a fact.
- For the next levels I'll script requests with `curl` or an intercepting proxy instead of editing them in the browser, so every attempt is repeatable and easy to document.
