# Security policy

Wyrm is local-first and holds your project memory on your own machine, so we take reports seriously.

## Reporting a vulnerability

Please do not open a public issue for a security problem. Instead:

- Email **ryan@ghosts.lk** with the details, or
- Open a private [security advisory](https://github.com/Ghosts-Protocol-Pvt-Ltd/wyrm-mcp/security/advisories/new) on this repository.

Include the version (`wyrm --version`), your platform, and steps to reproduce. We aim to acknowledge within a few days and will keep you updated through to a fix.

Our full disclosure policy, with what we commit to and the safe harbour for good-faith research, is at [ghosts.lk/security](https://ghosts.lk/security).

## Scope

This repository is the public home for Wyrm's docs and community. The source is maintained privately. Reports about the published `wyrm-mcp` package, its install and update flow, its handling of local data, or any hosted Wyrm service are all in scope.

## Supported versions

The current `latest` on npm is the supported line. Please reproduce on the latest release before reporting where you can.

## Thanks

People who reported a security problem privately and gave us time to fix it:

- **Loly**, independent security researcher. Two weaknesses in wyrm-mcp 9.5.0, one in how Wyrm syncs memory between machines and one in a leftover update tool, reported on 18 November 2026 and fixed in 9.5.1 on 19 November 2026. See the [9.5.1 changelog entry](CHANGELOG.md).
