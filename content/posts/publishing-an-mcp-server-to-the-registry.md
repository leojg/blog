---
title: "Publishing an MCP Server to the Registry: What the Schema Doesn't Tell You"
date: 2026-08-05T10:00:00-03:00
draft: false
tags: [mcp, python, packaging, pypi, open-source]
description: "The manifest validates, the publisher accepts it, the entry goes live — and the install is still broken. The convention that isn't in the schema, and why it cost me a second PyPI package."
---

I shipped [rihla](https://github.com/leojg/rihla) — a flight-search MCP server — to GitHub,
PyPI, and the official MCP registry in a single day. The mechanical steps went exactly as
planned. The one thing that didn't was the thing no schema could have told me: a `server.json`
that validates perfectly can still describe an install that doesn't work.

Here's the trap, the fix, and three smaller ordering gotchas that are cheap to avoid and
expensive to discover after upload.

## The convention that isn't in the schema

The registry manifest is small and well-specified. Mine looked like this:

```json
{
  "name": "io.github.leojg/rihla",
  "version": "0.1.0",
  "packages": [
    {
      "registryType": "pypi",
      "identifier": "rihla-mcp",
      "runtimeHint": "uvx",
      "transport": { "type": "stdio" }
    }
  ]
}
```

`mcp-publisher validate` passes. The schema is satisfied. But look at `identifier` and
`runtimeHint` together: the client is going to run `uvx <identifier>` and expect an MCP
server to start talking stdio at it.

My original plan pointed that identifier at `rihla` — the package that already existed,
which already shipped an MCP entry point. It would have been broken on arrival. `rihla` is a
CLI-first package with a deliberately lean default install; the MCP dependencies sit behind
a `[mcp]` extra. So `uvx rihla` installs the package *without* the MCP layer and launches
the flight-search CLI, which then sits there not speaking stdio.

The obvious question is whether the convention is really that rigid — surely a manifest can
express "install with this extra" or "run with these arguments"? So I sampled the registry:
roughly 1,500 entries, in early July 2026. Every one of the 63 PyPI-backed servers followed
the same shape — `uvx <identifier>` launching the server directly, zero using runtime
arguments or extras to get there. The official docs assume it too. It is not enforced by the
schema; it is just what every client and every human expects, which makes it more binding
than a schema rule, not less.

## The fix: a package with no code in it

Three options were on the table: make the MCP dependencies part of the default install
(giving every CLI user a dependency tree they didn't ask for), fight the convention and hope
clients cope, or publish a second package.

I published a second package. `rihla-mcp` is a pure-metadata launcher — no modules, no code
of its own:

```toml
[project]
name = "rihla-mcp"
dependencies = ["rihla[mcp]"]

[project.scripts]
rihla-mcp = "rihla.mcp_server:main"

[tool.setuptools]
packages = []
```

That's the whole thing. It exists so `uvx rihla-mcp` resolves and starts the server, while
the main package keeps its lean CLI-only default install.

Two details make it near-zero maintenance:

- **The dependency is deliberately unpinned.** `rihla[mcp]`, not `rihla[mcp]==0.1.0`.
  Publishing a new version of the real package doesn't require touching the stub. If I'd
  pinned it, I'd have doubled my release process forever to save a hypothetical
  compatibility problem I've never had.
- **`packages = []`** so setuptools doesn't go looking for source to include, and the build
  produces a metadata-only wheel.

The honest cost is one extra PyPI project name to hold, and one more artifact in every
verification pass. Cheap, given the alternative was an entry that appears to work and
doesn't.

## The README ships inside the artifact

My release plan sequenced a README edit — swapping *"not on PyPI yet, install from source"*
for `pip install rihla` — for *after* the first upload. That ordering is wrong, and it's
wrong in a way you can't fix afterwards.

The README is packaged into the sdist and the wheel, and PyPI renders the project page from
the uploaded artifact's metadata. PyPI does not let you re-upload a version. So the 0.1.0
page would have shown "not on PyPI yet" for as long as 0.1.0 exists, on the very page people
land on to find out how to install it.

The general rule, which cost me nothing this time only because I caught it while working:
**anything embedded in a published artifact must be final before upload, not patched after.**
Order your doc edits by where they ship, not by when they become true.

There's a second thing that has to be in the README before upload: the registry ownership
marker. A line reading `mcp-name: io.github.leojg/rihla` in the README ends up in the
published package metadata, which is how the registry verifies that whoever claims the
namespace also controls the PyPI package. Miss it and publishing fails — after you've
already burned the version number.

## twine needs a real terminal

Small, but it stopped me cold. `twine upload` prompts for the API token through `getpass`,
which requires a TTY. The shell my coding agent runs commands in doesn't have one, so the
upload just hangs or errors depending on how you invoke it.

There's no clever fix — you run the upload from your own terminal, or you set up
non-interactive credentials. What's worth generalizing is the planning habit: when you write
a release plan, mark which steps need a human-attended terminal. Credential prompts,
device-flow logins, anything with a `getpass` in it. It's an obvious constraint that's
invisible right up until it blocks the one irreversible step in the process.

(The registry's own `mcp-publisher login github` uses a device flow — you open a URL and
paste a code. That one works fine anywhere, because it prints a URL instead of reading a
password.)

## What I actually verified before and after upload

Publishing is irreversible per version, so the checklist is worth more than usual. All of
this ran green:

- Full test suite and linter clean on the tagged tree.
- `twine check` on all four artifacts (two packages × sdist + wheel).
- **Fresh-venv install of the local wheel** — catches missing files in the sdist.
- **Fresh-venv install of `rihla[mcp]` from live PyPI** — catches "works from my checkout".
- **Fresh-venv install of the stub wheel**, which pulls the real package from PyPI. This is
  the one that actually proves the two-package arrangement resolves.
- PyPI JSON metadata check after upload: license expression, project URLs, and the
  `mcp-name` marker present.
- `mcp-publisher validate` before publishing, and a registry query after — confirming the
  entry live with status `active`.

The fresh-venv installs are the load-bearing ones. Everything else is a spelling check.

## The lesson

The registry has a schema, a validator, and a publisher CLI, and passing all three tells you
your manifest is *well-formed* — not that your server installs. The binding constraints live
in convention: what `uvx <identifier>` is expected to do, what a client assumes when it sees
`runtimeHint`, what shape every other entry in the registry has.

Ten minutes of sampling existing entries would have surfaced all of it before I wrote the
plan, instead of mid-release with a tag already pushed. That generalizes past MCP: **when
you're publishing into an ecosystem you haven't published into before, read the neighbours,
not just the spec.** The spec tells you what's permitted. The neighbours tell you what works.

## Sources

- [rihla on GitHub](https://github.com/leojg/rihla) — the server, the stub package under
  `packaging/rihla-mcp/`, and `server.json`
- [`rihla`](https://pypi.org/project/rihla/) and [`rihla-mcp`](https://pypi.org/project/rihla-mcp/) on PyPI
- [modelcontextprotocol/registry](https://github.com/modelcontextprotocol/registry) — schema,
  publisher CLI, and publishing docs
