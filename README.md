# EVE — Developer Bundle

EVE is a rule-driven blockchain platform. An institution declares the rules its
submissions are judged by, and the chain evaluates them deterministically.

This repository holds the downloadable developer bundle. Everything in it runs
**offline, on your own machine** — no account, no API key, no connection to any
hosted service.

## Download

Get the latest bundle from [Releases](../../releases/latest). The current
version is **0.5.0**: download `eve-0.5.0-linux-x64.tar.gz` and its `.sha256`
into the same folder.

## Requirements

- **Linux x64** — including Ubuntu under Windows Subsystem for Linux (WSL)
- **Node.js 18 or newer**

Native Windows, macOS, and Linux ARM are not release targets yet. On Windows,
run the bundle inside WSL: the archive ships a Linux binary that PowerShell
cannot execute.

Go and the EVE chain source are **not** required. The bundle includes a compiled
`eved`.

## Install

```bash
# verify the download before extracting
sha256sum -c eve-0.5.0-linux-x64.tar.gz.sha256

tar -xzf eve-0.5.0-linux-x64.tar.gz
export PATH="$PWD/eve-linux-x64:$PATH"

eve doctor
```

`eve doctor` reports your Node version, the bundled chain binary and its
capabilities, and whether the devnet storage path is writable. Run it first —
it tells you what is wrong before anything else can fail confusingly.

## Quick start

```bash
eve init my-protocol
cd my-protocol

eve validate                 # check rules offline, without running author code
eve dev --detach             # start a disposable local chain + gateway in the background
eve run rules/example.js     # install your rules and endpoints on it
eve run tests/example.test.js tests/endpoint.test.js   # submit, and check the result
eve call submit_amount amount=100                      # call one endpoint and see what changed
```

`eve run` passes the local chain's address and keys to your scripts, so run
tests through it rather than with plain `node` or `npm test`. Stop the chain
with `eve dev --teardown`.

## What the bundle contains

| Part | Purpose |
|---|---|
| `eve` CLI | scaffold, validate, pack, run, and the local Studio UI |
| `eved` | the EVE chain binary, with its SHA-256 recorded in `manifest.json` |
| Local gateway | the HTTP API `eve dev` runs alongside the chain |
| Chain provisioner | builds the throwaway devnet |
| Templates | a scaffolded project with example rules and a passing test |

It never touches an existing `eved` installation or an existing chain.

## Commands

```bash
eve init my-protocol     # scaffold a project
eve doctor               # check the installation
eve validate             # structural + rule-syntax checks, offline
eve dev                  # disposable local chain and gateway (--detach, --teardown)
eve run rules/example.js # run a project script with the bundle's dependencies
eve call <endpoint> ...  # call an endpoint and show each rule step and the state change
eve studio               # loopback-only local UI for the workspace
eve pack                 # deterministic dist/*.eve.json with a SHA-256 digest
```

### Rule shape

A rule set is a pipeline of three channels — `entry → process → exit`. Each
channel requires a **Creator** and may add a **Reviewer** and an **Enforcer**.
Rules are written one of two ways:

- **3x1** — the Creator alone holds the checks and formulas. A Creator-only
  pipeline is complete and runnable.
- **3x3** — one job per role. The Creator lists the fields a submission must
  contain (a missing one is rejected), the Reviewer works out the formulas and
  decides, and the Enforcer carries out the decision; it never judges.

```yaml
entry:
  creator:
    fields: amount, discount
  reviewer:
    formulaDefinitions: 'CALC:total = amount - discount'
    checkConditions: total > 0          # all true: approved
    rejectionConditions: amount <= 0    # any true: rejected; otherwise held for review
  enforcer:
    onApproved: advance_to_process
    onRejected: return_to_sender
```

List in the Creator every field the Reviewer uses: a missing field reads as 0 in
a formula, so only the Creator's list catches it.

### Going online

`eve pack` produces a package with an integrity digest. Uploading it to a hosted
EVE instance is a separate, optional step:

```bash
eve login    --server https://your-eve-host --institution my_institution
eve upload   dist/my-protocol-0.1.0.eve.json --server https://your-eve-host
eve packages --server https://your-eve-host
eve deploy   my-protocol@0.1.0 --server https://your-eve-host
eve deployments --server https://your-eve-host
```

The server recomputes the digest and re-runs its own chain binary's validator
before storing anything — a local pass is a convenience for you, never evidence
the server accepts. Upload stores a version; it does not deploy it or change
chain state. `eve deploy` shows exactly what will change and asks you to type
`deploy` before it goes live.

The CLI stores only a short-lived token, in your user config directory. It never
stores your password, and never writes credentials into the project.

## Verifying what you downloaded

Every release ships a `.sha256` beside the archive. The bundled chain binary
carries its own digest in `manifest.json`:

```bash
cat eve-linux-x64/manifest.json
sha256sum eve-linux-x64/app/vendor/bin/linux-x64/eved
```

If those two disagree, do not run it.

## Source

The EVE chain and gateway source are maintained separately. This repository
carries the built bundles only.
