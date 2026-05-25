# Contributing

This is a community knowledge base for The Isle EVRIMA dedicated-server modding. Corrections, additions, and new findings are welcome.

## What's in scope

- Server-side UE4SS Lua patterns and gotchas
- UE4SS C++ side-mod patterns
- Asset extraction (CUE4Parse plus Oodle pipeline)
- EVRIMA-specific UFunction surface, class layouts, hooks that fire vs don't
- Tool recipes (RCON, asset packs, debug tooling)

## What's not in scope

- Cheats, exploits, or anything that violates EAC
- Specific server-owner credentials, API keys, or steam IDs
- Closed-source mod redistribution
- Marketing material or commercial pitches

## How to contribute

### Reporting an outdated fact

Open an issue describing what's wrong and what you've observed. Include:

- The document and section.
- What it says now.
- What you observed instead.
- Game version and UE4SS version where you observed it.

If you have a screenshot, log excerpt, or repro steps, those help a lot.

### Adding new knowledge

Open a pull request. The expected shape:

- New gotcha or rule: add to `EVRIMA_Lua_Safety_Rules.md` if it's a crash pattern, or to the relevant cookbook if it's a feature-specific finding.
- New mod architecture: create `EVRIMA_<ModName>_Architecture.md` following the structure of the existing architecture docs.
- New reference catalog: create `EVRIMA_<Topic>.md` and link it from the README.
- Updates to an existing doc: edit the doc and mention the change in the PR description.

### Style guide

The existing docs use a few consistent conventions:

- Lab-notebook voice. First person "I" when sharing observed behavior, impersonal voice for general statements. No "we" or "our."
- No em dashes. Periods, commas, parens, or hyphens.
- Code blocks use the appropriate language tag (`lua`, `cpp`, `powershell`, `bash`, `json`).
- File paths use `<game>/` as the root placeholder for the EVRIMA install dir.
- Steam IDs in examples use `76561198XXXXXXX` placeholders, never real IDs.
- Credentials (AES keys, EOS client IDs, RCON passwords) are never included; redact to `<YOUR_X>` placeholders.

If you're adding a new doc, the simplest path is to copy the structure of an existing one as a template.

### Verifying changes

If your change touches a code snippet or a specific behavior claim, verify it on a live dedicated server before submitting. The docs' value depends on the claims being accurate. "I think this works" is worse than no claim at all.

If you can't verify but believe a claim is wrong, opening an issue (not a PR) is the right path. Someone with a test server can then verify before merging.

## Maintainer

Original author and primary maintainer: [@diplomatic-tendencies](https://github.com/diplomatic-tendencies).

PRs reviewed on a rolling basis. For substantial changes, opening an issue first to discuss the approach saves rework.
