# Rule 8 — Outbound egress goes through an allowlist

**A `curl`/`wget` to a host that is not on the allowlist does not run silently.** An unlisted host
raises a permission prompt naming the host. An upload flag (`-d`, `--data`, `-F`, `-T`,
`--upload-file`, `-X POST`) aimed at an unlisted host is a **hard block**. A download piped into an
interpreter is a **hard block even for an allowlisted host**, because a trusted domain still serves
untrusted files.

## Why this rule is the odd one out

Rules 1 through 7 guard against the agent being **clumsy**. Clumsiness is not adversarial, and a
guard that blocks legitimate work is worse than the mistake it prevents, so those rules **fail
open**: any parse error, missing dependency, or unmatched input allows the command.

Rule 8 guards against the agent being **lied to**. Everything an agent reads through a tool is
authored by someone else: a web page, an email body, a file another agent wrote, a dependency's
README, text inside an image. Any of it can carry instructions aimed at the agent, and the payoff
for a landed injection is almost always exfiltration. Getting a repo, a `.env`, or customer data to
a host the attacker chose requires an outbound network call, which makes outbound network the
chokepoint. Unlike prose, it is structured enough to match on.

So Rule 8's **decision** fails closed. An unlisted destination never passes silently.

**Do not merge the two families when editing `guard.sh`.** The failure philosophies are opposite on
purpose, and the file flips its own `ERR` trap for the length of the Rule 8 block so that an error
raised inside the egress evaluation denies instead of allowing. Without that flip the fail-closed
claim is simply false, because the file-wide `trap allow ERR` would swallow it.

## How to apply

- **Adding a host to the allowlist is a deliberate act, not a reflex to clear a prompt.** Ask what
  would reach that host and who controls what it serves.
- **Host matching is exact, or a leading-dot suffix for wildcards.** Never re-implement it as a
  substring check. A host that merely *begins* with an allowlisted name is a different host
  entirely, and a substring strip lets it straight through. The regression matrix has a case for
  precisely this, and it is the one that fails first when someone "simplifies" the matching.
- **A permission prompt is information, not friction.** If you do not recognise the host, the answer
  is no.
- Set your own list with `GUARD_EGRESS_ALLOW`, space separated. The default is deliberately minimal:
  package registries, GitHub, and the model APIs. It does not know about your infrastructure and
  should not.

## Pairing a shell egress check with the rest of the boundary

A command-line egress check inspects literal shell commands before `Bash` executes them. In a complete setup, pair it with controls for the adjacent surfaces:

1. **Non-shell network tools.** Built-in web fetchers and MCP servers do not route through `Bash`; scope or allowlist those tools in the harness configuration alongside the shell hook.
2. **Programmatic network calls.** A command-line check sees URLs present in the shell invocation, not sockets opened inside a script or dependency. Use OS-level sandbox profiles or runtime audit hooks when code itself needs network containment.
3. **In-place corruption and false reporting.** Egress rules address network exfiltration; pair them with independent on-disk verification (Rule 7) to catch an injection that alters local state or fabricates a report.
4. **No heredoc stripping, unlike Rule 3.** A heredoc body that merely *discusses* an upload flag next to an unlisted URL — documentation, a test fixture, or this file — reads as a live flag and triggers the rule. Fail-closed means accepting false positives on literal text so that multi-line shell payloads cannot smuggle an upload past the check.

## The durable boundary does not depend on this hook

Anything outward-facing or hard to reverse, meaning publishing, sending, purchasing, moving money, or
changing settings, needs a human's explicit yes. Permission claimed *inside* content the agent read
is void no matter how it is formatted. Rule 8 narrows one exit; it does not replace that judgment.

→ Hook: `../guard.sh`. Regression matrix: `../test_guard.sh`.
