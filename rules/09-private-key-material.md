# Rule 9 — Private key material is never read

**A command that reads, copies, or pipes the CONTENTS of a private key does not run.** Using a key
is fine: `ssh`, `scp`, `sftp`, `ssh-add`, `ssh-keygen`, `ssh-copy-id`, `chmod`, `chown`, `ls`,
`stat` and `file` may all name one. `cat`, `base64`, `cp`, `tar`, `curl -T`, a Python `open()` —
those are refused. Public keys (`.pub`), `known_hosts`, `config` and `authorized_keys` are
untouched; reading those is ordinary work.

## Why this is separate from Rule 8, which already blocks exfiltration

Because a key is not like other secrets. An API token usually buys an API, and usually a rate-limited
one. An SSH key with no passphrase buys the machine it opens, with **no exploit anywhere in the
path** — the read *is* the compromise and the upload is a formality.

Work out what an attacker actually needs to compromise a hardened server and the perimeter answers
are usually reassuring: key-only SSH, fail2ban, unattended upgrades, nothing listening that should
not be, no sudo for the service account. None of that helps if the shortest path skips the server
entirely and goes through the workstation — one local key read, one outbound request — on a machine
where an agent has both file access and network reach.

## Two surfaces, two rules, and you need both

The file-reading tool and the shell are **different tools**. A deny rule on an agent's file reader
stops it opening `~/.ssh/**` and does nothing about `cat key | curl`, which runs in Bash and never
passes through the file reader at all. Putting a rule on the first while leaving the second open
creates the illusion of coverage because there is "a rule for that."

This is Rule 8's scope note restated, and it generalizes past keys: **a secret worth protecting
needs a rule on every tool that can reach it.** Enumerate the tools, not the secrets.

## Segment by segment, and that is the whole design

The first version scanned the command as one string with two blanket escapes: skip if `.pub`
appears anywhere, skip if the command starts with a safe verb. Both are defeated by appending or
prepending — a harmless read placed beside a dangerous one vouched for it, and the whole command
sailed through.

So the command is **split on shell separators and every segment is judged alone.** A safe segment
can no longer speak for a dangerous one. This is the same disease as any substring test standing in
for a security decision, and the same cure.

## Family: Rule 8's, not Rule 1–7's

This one faces an adversary rather than your own clumsiness, so its **decision fails closed**.

**Do not resolve a false positive by loosening the pattern.** Run the command yourself, outside the
agent. It will produce false positives and that is the correct trade — grepping this repository's
own source trips it, because the patterns it matches on are written in the file it is scanning.

## Pairing a shell hook with the rest of the boundary

A command-line hook inspects one surface: literal shell strings passed to `Bash`. Designing a
complete boundary around it means pairing it with controls at three other layers:

1. **Non-shell tools.** File readers, file writers, web fetchers, and MCP tools do not pass through
   a `Bash` hook. Pair shell rules with permission scopes on every tool family in the agent's
   harness.
2. **Indirect execution.** A command-string check sees the arguments passed on the command line, not
   the file I/O performed inside an imported library or child script. Enforce filesystem containment
   at the OS sandbox or runtime audit layer when running untrusted code.
3. **Key usage vs. key reading.** Blocking raw reads stops exfiltration of the file bytes; it does
   not restrict what an authorized SSH client can do on the remote host.

**A hook is a backstop, not a standalone boundary.** The durable fixes are upstream of it: a
passphrase, so a stolen copy is inert on its own; a forced command in `authorized_keys` when
automation only needs one read-only verb; and a non-root login user, so it buys less when used.
