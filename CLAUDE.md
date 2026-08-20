# CLAUDE.md

Shareable Renovate presets. Nothing here runs on its own — every file is consumed by *other* repos, so a
change lands as a behaviour change in someone else's dependency automation.

## Repo layout

Five presets, extended à la carte by each consuming repo:

| File | Purpose |
| --- | --- |
| `default.json` | the semantic-commit baseline: type by update level, and the global scope default. Every repo extends this |
| `dev-tools.json` | toolchain managers (npm/bun, poetry/pep621, mise, pre-commit) — and the **only** place `pre-commit` and `lockFileMaintenance` are *enabled*, since both default to `enabled: false`. `default.json` matches `lockFileMaintenance` in a rule but does not turn it on |
| `github-actions.json` | for a repo that *consumes* Actions: its workflows are internal CI |
| `github-actions-provider.json` | for a repo that *publishes* workflows or actions: what its workflows pin reaches consumers |
| `kubernetes.json` | Kubernetes API versions, Flux sources, kind/kubectl pins |

`github-actions.json` and `github-actions-provider.json` are near-copies. The difference is deliberate and
load-bearing: the provider variant drops `semanticCommitType` (so dependency bumps keep the semver-derived
`feat`/`fix` rather than flattening to `chore`) and scopes workflow-regex deps `shipped-dependencies`
instead of `internal-dependencies`. **Any edit to one almost certainly belongs in the other** — check both,
and keep the divergence to those two axes.

`README.md` carries the shared scope vocabulary that consumers read. Keep it in sync when scopes change;
it is the only place the internal-vs-shipped question is written down.

## Blast radius — read before editing any preset

**This repo consumes its own presets unpinned.** `.github/renovate.json` extends
`github>ppat/renovate-presets` with no `#tag`, so main is live against itself the moment it merges. That
makes it a useful canary, and it means this repo's own `commitlint.config.js` `scope-enum` must accept
whatever main emits — in the same commit, not afterwards.

**Consumers pin by tag**, so they are insulated until their pin moves. That is the only thing that makes
breaking changes survivable here.

**A change to an emitted scope is breaking for every consumer.** Their `commitlint.config.js` gates
Renovate's own pull requests; emit a scope their enum doesn't list and their dependency updates stop
silently. The migration order is always: *add the new scope to the consumer's enum → let its pin move →
drop the old scope.* Say so in the pull request body — the person merging is the person who has to
sequence it.

Thirteen repos extend these presets today. Enumerate them before a scope or label change rather than
trusting a list that will rot — `grep -rl "ppat/renovate-presets" */.github/renovate.json` from the parent
directory of your checkouts.

## Verifying a change — resolve it, don't read it

**Reading the JSON is not verification.** These rules interact through last-match-wins merging across a
50-rule chain, and the failures that matter are the ones no current dependency triggers. Two real bugs got
past careful reading and were only caught by resolving: a `groupName` that Renovate deletes before it can
apply, and a scope leak that existed only in combinations nothing exercises yet.

Resolve through Renovate's own code, at the version consumers pin:

```bash
mkdir -p /tmp/rp-check && cd /tmp/rp-check
echo '{"name":"c","private":true}' > package.json
npm i renovate@<version-consumers-pin>          # not "latest" — match the consumer
```

```javascript
// ESM, and applyPackageRules is async — awaiting it is not optional
import {applyPackageRules} from 'renovate/dist/util/package-rules/index.js';
const out = await applyPackageRules({
  ...globals,                    // semanticCommits/Scope/Type from default.json's top level
  packageRules,                  // every preset's rules concatenated in `extends` order, then the repo's own
  manager, packageFile, lockFiles, depName, packageName, datasource, updateType, depType,
});
```

Then sweep a **cross product** — every tracked file × every manager the config can enable × every update
type (all ten, not just the semver three) × the datasources any rule keys on — and diff the resolved
output before vs after your change. Assert on properties, not eyeballs: no scope outside the consumer's
enum, no scope contradicting the file's ship status, no breaking marker on something that cannot reach a
consumer.

Distinguish **reachable** cells (the manager can own that file, the path isn't in `ignorePaths`, the
manager can raise that update type) from the rest. Resolve both; the unreachable ones are where latent
defects hide, so do not filter them out of the checks.

## packageRules semantics that have already caused bugs

- **Last match wins, per field.** Ordering is load-bearing. If a rule only works because a later rule
  undoes an earlier one, tighten the matcher instead — and say in the rule's `description` when order
  matters, because nothing else records it.
- **`matchFileNames` tests `packageFile` *and* every `lockFiles` entry**, so a lock file can be matched by
  its own name rather than via its manifest.
- **List-level negation does not work.** Renovate evaluates the patterns with `.some()`, so
  `["a/**", "!a/b"]` still matches `a/b`. Use an extglob inside one pattern — `a/!(b)*` — which does work.
- **`groupName` with a `{{packageName}}` template is inert.** `depNames` is deduplicated when computing
  `groupEligible`, so a per-package group is always single-dep, `groupSingleUpdates` defaults to `false`,
  and Renovate deletes `groupName` outright. A group only engages when it spans several packages.
- **Consolidating one dependency across many files is `branchTopic`, not grouping.** It keys on the dep and
  target version, so N files already collapse to one branch and one pull request with no group configured.
- **Renovate has no "emit a breaking marker" option.** `!` requires hand-building the header with
  `commitMessagePrefix`; supplying it suppresses Renovate's own semantic-prefix assembly.
- **`commitBody` is a truthy check**, so `""` is identical to unset. It is not release notes (those are the
  PR body) and not the dependency table (that is `commitBodyTable`, separate, default `false`).
- **A bare `BREAKING CHANGE` body parses as nothing** — no colon, no note, no major bump. If you mean it,
  it needs a colon and a description; if you don't, leave the body empty and let the subject line speak.

## Commit conventions

commitlint gates every pull request, and release-please derives the version and changelog from the header.
Scopes are limited to `''`, `github-actions`, `internal-dependencies`, `release`, `renovate-presets`.

That enum is for **this repo's own commits**, which is not the same list as the vocabulary the presets
*emit* for consumers — only add a scope here when something in this repo actually commits under it.

Use `renovate-presets` for changes to the preset files themselves, `internal-dependencies` for this repo's
own tooling, `github-actions` for `uses:` bumps, `release` for release machinery.

## Releases

release-please, driven by commits on main; `release.yaml` opens the release pull request and merging it
cuts the tag. Do not hand-edit `CHANGELOG.md` or `.release-please-manifest.json`.

There is no `bump-minor-pre-major`, so **`!` cuts a major** — including from `0.x`, where release-please
would otherwise treat a breaking change as a minor. Given how much of this repo is a published interface,
assume a scope rename, a label rename or an emitted-type change is a `!`.

## Working in this repo

- Validate every preset you touch with `renovate-config-validator` at the version consumers pin, not
  whatever is newest. Older validators reject keys that are perfectly valid in the pinned version.
- Presets are flat: none of them uses `extends`, so what you read in a file is all it does. Keep it that
  way — the ordering rules above are hard enough to reason about without indirection.
- Preset **filenames are public API**: consumers name them in `extends`. Renaming one breaks every consumer
  at once, and unlike a scope change their config cannot absorb it — Renovate fails to resolve the preset,
  so the whole repo config errors and *no* updates run, not merely wrongly-scoped ones. Rename what a
  preset *emits* instead.
- **A filename names the situation the preset is for, not the scope it emits**, and the two are not
  expected to match. `github-actions.json` emits `github-actions` *and* `internal-dependencies`;
  `github-actions-provider.json` emits `github-actions` *and* `shipped-dependencies`. `dev-tools.json` is
  the only preset whose emitted scope is uniform, which makes renaming it to `internal-dependencies.json`
  look tempting — don't. It selects deps by *which manager owns them* (npm/bun dev deps, poetry groups,
  mise, pre-commit), which is a different question from whether those deps reach a consumer. It governs
  `package.json` in repos where `package.json` ships, so a file named `internal-dependencies.json` would
  be making a claim the preset cannot support. And since `github-actions.json` emits that scope too, the
  name is not even uniquely available.
- A preset cannot decide whether a dependency ships — that depends on which paths a repo publishes, and it
  is not the same shape even between two repos extending the same preset. Emit the conservative default and
  let the consumer override by file. `README.md` explains this to them; do not try to solve it here.
- Labels in `addLabels` must already exist in the consuming repo before Renovate can apply them, so a label
  rename is coordination work, not a config change. Keep it out of unrelated pull requests.
