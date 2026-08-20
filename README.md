# renovate-presets

Shared Renovate configuration for this org. Every repo extends `default`; the rest are opt-in by what the
repo actually contains.

| Preset | Extend it when |
| --- | --- |
| `default` | always — it sets the semantic-commit baseline every other preset builds on |
| `dev-tools` | the repo has a toolchain Renovate can manage (npm/bun, poetry/pep621, mise, pre-commit) |
| `github-actions` | the repo *consumes* GitHub Actions and its workflows are internal CI |
| `github-actions-provider` | the repo *publishes* workflows or actions, so what it pins reaches consumers |
| `kubernetes` | the repo contains Kubernetes manifests, Flux sources, or kind/kubectl pins |

## Commit scopes these presets emit

These are the shared scope names. A repo's `commitlint.config.js` must accept every scope its extended
presets can emit, or Renovate's own pull requests fail lint and dependency updates stop.

| Scope | Means |
| --- | --- |
| `internal-dependencies` | tooling used to develop, lint or test the repo, which does **not** ship with its built artifact |
| `shipped-dependencies` | dependencies resolved at **consumer**-run time by whatever the repo publishes |
| `github-actions` | a GitHub Actions `uses:` reference was bumped or re-pinned — that, and nothing else |
| `kubernetes-api` | a Kubernetes `apiVersion` was bumped |
| `release` | release machinery, and the release cuts themselves |
| `renovate` | the repo's own Renovate configuration |

## Major updates are marked breaking

`default.json` puts `!` in the header of every **major** update — `feat(scope)!:`, or `feat!:` in a repo
that sets no scope. Conventional Commits readers and release-please both key on that marker, so a major
dependency bump cuts a **breaking** release rather than a quiet one.

This is deliberately conservative: a shared preset cannot know whether a given dependency reaches your
consumers, and marking too many majors breaking is loud and reversible, whereas missing one ships a
breaking change as a patch.

**If a major genuinely cannot affect your consumers** — an internal linter, a test fixture, a pin used only
by your own CI — override the prefix locally for those paths and re-add `!` only where it belongs:

```jsonc
// strip the inherited marker, then put it back for what actually ships
{ "matchUpdateTypes": ["major"],
  "commitMessagePrefix": "{{semanticCommitType}}{{#if semanticCommitScope}}({{semanticCommitScope}}){{/if}}:" },
{ "matchUpdateTypes": ["major"], "matchFileNames": ["<paths that ship>"],
  "commitMessagePrefix": "{{semanticCommitType}}{{#if semanticCommitScope}}({{semanticCommitScope}}){{/if}}!:" }
```

Note that setting `commitMessagePrefix` yourself disables Renovate's own lower-casing of the subject line,
so pair it with `"commitMessageTopic": "{{lowercase depName}}"` if you want the previous casing.

### `internal-dependencies` vs `shipped-dependencies`

One question decides it, asked per file:

> **Is this resolved at *consumer*-run time by the artifact this repo publishes?**

If yes it is a shipped dependency; if only the repo's own CI or local dev loop ever uses it, it is an
internal one. The same tool can land on both sides of that line in the same repo — a linter pinned in a
pre-commit config is internal, while the same linter pinned inside a workflow that consumers call is
shipped. Ask where the pin lives, not what the tool is for.

**These presets cannot make that call for you.** Which paths are "published" is a fact about each repo's
layout, and it is not the same shape everywhere — for a repo publishing reusable workflows the pins under
`.github/workflows/` ship, while for one publishing composite actions those same paths are its own CI and
the shipped pins live with the actions. So the presets emit `internal-dependencies` as the conservative
default, and a repo whose artifact includes those paths overrides the scope locally with a `packageRules`
entry keyed on the file. Nothing here emits `shipped-dependencies`; the name is defined here so that every
repo making that override uses the same word.
