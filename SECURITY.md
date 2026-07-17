# Security policy

Wyrm is local-first and holds your project memory on your own machine, so we take reports seriously.

## Reporting a vulnerability

Please do not open a public issue for a security problem. Instead:

- Email **ryan@ghosts.lk** with the details, or
- Open a private [security advisory](https://github.com/Ghosts-Protocol-Pvt-Ltd/wyrm-mcp/security/advisories/new) on this repository.

Include the version (`wyrm --version`), your platform, and steps to reproduce. We aim to acknowledge within a few days and will keep you updated through to a fix.

## Scope

This repository is the public home for Wyrm's docs and community. The source is maintained privately. Reports about the published `wyrm-mcp` package, its install and update flow, its handling of local data, or any hosted Wyrm service are all in scope.

## Security by design, not by obscurity

Wyrm's security does not depend on our code being secret. The published package is
minified for commercial reasons, and the source is maintained privately, but neither is a
security control. Every guarantee below is structural and holds even if the implementation
is fully known (this is Kerckhoffs's principle, the standard serious cryptosystems follow):

- **Local-first by default.** Your memory lives in a SQLite database on your own machine.
  Nothing leaves it unless you explicitly enable a cloud or sync feature, and the free
  local tier runs with no account and no network at all.
- **Licensing** is verified with an offline **Ed25519 signature**. The package ships only
  the public verifying key; the private signing key is never distributed. Reading or
  reverse-engineering the client cannot forge a valid license.
- **Untrusted-content quarantine.** Memory tagged as untrusted in origin is categorically
  withheld from the context briefs an AI model reads. This holds whatever the content
  contains and does not rely on any detector pattern being secret, so a hostile
  instruction planted in imported or scraped text cannot be replayed to the model as a
  command.

So the fact that older published versions are readable, or that any release can be
un-minified, does not weaken any of this. We state it plainly so you do not have to take
it on trust.

## Transparency note: type-declaration exposure in 7.2.2 to 7.5.0

Between wyrm-mcp 7.2.2 and 7.5.0, a packaging error caused the published npm tarballs to
include TypeScript declaration files (`.d.ts`) and source maps (`.js.map`, `.d.ts.map`) that
were meant to be stripped from the release. We are disclosing it because transparency is the
standard we hold ourselves to, even where there is no user impact.

**What was exposed.** For those specific versions, the declaration files and maps reveal
Wyrm's internal type surface and code comments (its design and structure). This is our own
source-adjacent material.

**What was not.** No user data of any kind was involved. No secrets were present: no API
keys, tokens, credentials, or infrastructure identifiers shipped in any release. The source
maps carry no embedded source (`sourcesContent` is null), so the original source cannot be
reconstructed from them, and the compiled JavaScript was minified in these versions as
intended.

**Why it does not affect your security.** No control depends on this material being private
(see "Security by design, not by obscurity" above). License verification, authentication,
tenant isolation, and the untrusted-content quarantine are all structural and hold with the
implementation fully known.

**What we did.** The affected files are stripped from every release since 7.6.0, verified in
the build. We added a build guard that refuses to publish if any declaration file, map file,
or un-minified source is present in the package, so this class of error cannot recur. The
affected versions remain on npm because published versions cannot be unpublished; they are
superseded, and the current `latest` line is clean.

## Supported versions

The current `latest` on npm is the supported line. Please reproduce on the latest release before reporting where you can.
