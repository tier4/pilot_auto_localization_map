## TIER IV universe / split mirror (localization and map)

This repository split mirrors the localization and map subtrees of several upstream repositories, used for TIER IV release workflows.

This branch only holds the mirror configuration and its tooling. See the mirror branches for source code.

### Published branches

| Branch | Contents |
| --- | --- |
| `awf-latest` | `autowarefoundation/autoware_universe:main`, `localization/`, `map/` |
| `awf-core-latest` | `autowarefoundation/autoware_core:main`, `localization/`, `map/` |
| `awf-launch-latest` | `autowarefoundation/autoware_launch:main`, `autoware_localization_*` and `autoware_map_*` |
| `feat/v0.64/e2e` | `tier4/autoware_universe:feat/v0.64/e2e`, the same paths as the universe mirror |
| `awf-combined-latest` | the three mirrors above replayed into one linear history |

In the combined branch each member is filed under the name of its upstream,
which keeps the histories out of one another's directories and makes provenance
visible in the path. Everything else at the root comes from `autoware_universe`.

`awf-launch-latest` is configured ahead of the upstream restructure that creates
its paths: [autoware_launch#1971](https://github.com/autowarefoundation/autoware_launch/pull/1971)
migrates `tier4_localization_launch` to `autoware_localization_launch` plus a
separate config package, and map is to follow. Until those directories exist the
filter matches nothing, and the source is marked `optional` so that is a skip
rather than a failure and the combined target leaves the member out. The mirror
starts on its own once the upstream change lands.

```text
awf-combined-latest/
├── universe/localization/
├── universe/map/
├── core/localization/
├── core/map/
├── launch/
└── .github/  docs/  LICENSE  NOTICE  README.md  ...
```

The mirror branches keep the flat names the previous workflows used, so nothing
that already points at `awf-latest` or `awf-core-latest` has to move. Grouping
them under an `awf-latest/` namespace would read better but would require
deleting the existing `awf-latest` branch first, which the organisation rulesets
do not permit here. Because every branch is a deterministic function of its
inputs, renaming later costs one configuration change and one rerun.

### How it works

[`.sync/sources.yaml`](.sync/sources.yaml) is the single source of truth. It
describes every upstream, the paths to retain, the commit-message rewriting and
the branch each mirror is published to. `.github/workflows/mirror.yaml` derives
its job matrices from that file, so adding or changing a mirror is a
configuration change and never a workflow change.

The pipeline has two stages:

1. **Filter.** `tools/mirror.py mirror SOURCE` clones the upstream and runs
   `git-filter-repo` with arguments generated from the configuration, then
   pushes the result to that source's `mirror_branch`.
2. **Combine.** `tools/mirror.py combine TARGET` reads the mirror branches that
   stage 1 published and appends any not-yet-reflected member commits onto the
   already published combined tip (or builds from scratch on first publish).
   It never clones an upstream, so the combined branch cannot disagree with the
   per-source mirrors.

### Determinism and publishing

Per-source mirrors are pure functions of `(upstream commit, .sync/sources.yaml)`:

- `git-filter-repo` rewrites a given history the same way every time. The
  version is pinned in the workflow, because a different version may rewrite
  differently.
- The same upstream tip therefore republishes as a fast-forward. `awf-latest`
  is configured with `force: false` on purpose, so losing reproducibility fails
  the job instead of silently rewriting.

The combined branch trades a different guarantee. It appends onto the already
published tip: only member commits not yet reflected there are replayed, and
the resume point is recovered from the tip tree (each member's renamed subtree
oids). Published combined commit ids never change, so a ruleset that forbids
force pushes does not block normal updates. What is given up is rebuild
identity — recreating the branch from scratch is not expected to reproduce
those ids.

Content stays auditable:

- `tools/mirror.py combine --verify` checks that the tip tree matches what the
  current member tips compose to, and that appending from the same published
  tip is reproducible. The scheduled workflow always passes `--verify`.
- Every push reports whether the previously published tip is still an ancestor
  of the new one. Under append-only operation that update is always a
  fast-forward (or unchanged).

### Working on the configuration

```bash
python3 -m pip install pyyaml git-filter-repo==2.47.0

tools/sync_config.py validate                # check the configuration
tools/sync_config.py show autoware_universe  # the git-filter-repo call it implies
```

Without `--push`, `tools/mirror.py mirror` and `tools/mirror.py combine` are dry runs.
