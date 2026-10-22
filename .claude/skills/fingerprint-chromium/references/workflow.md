# Build, patch and rebase workflow

## Repo layout

This repo is an ungoogled-chromium tree, not a Chromium checkout. It holds config and patches;
the source is downloaded and patched under `build/` (gitignored).

| Path | Purpose |
|---|---|
| `chromium_version.txt` / `revision.txt` | Chromium version, and the fork's revision on top of it (tag = `<version>-<revision>`, see `devutils/print_tag_version.sh`) |
| `downloads.ini` | the `chromium-<ver>-lite.tar.xz` tarball + hashes from `commondatastorage.googleapis.com` |
| `flags.gn` | base GN args. It is only a subset; there are no Linux-specific args such as `is_official_build` or `use_sysroot` |
| `pruning.list`, `domain_substitution.list`, `domain_regex.list` | binaries to prune, files and regexes for domain substitution |
| `patches/series` | apply order. The order is `core/*`, then `extra/*`, then `extra/fingerprint/000…021` last |
| `utils/` | pipeline tools (`downloads.py`, `clone.py`, `prune_binaries.py`, `patches.py`, `domain_substitution.py`) |
| `devutils/` | validators, list updaters, quilt env, lint/test wrappers |
| `docs/` | unmodified upstream ungoogled-chromium docs |
| `.github/workflows/ci.yml` | CI: yapf + pylint + pytest on utils/devutils, then `validate_config.py`, then `validate_patches.py` against downloaded source |

There is **no platform repo**: a Linux binary needs ungoogled-chromium-portablelinux (with this repo
in place of its `ungoogled-chromium` submodule), or a custom script.

## Where things run

Builds, source downloads, `validate_patches.py -l` and quilt refreshes need a Chromium source tree,
so they run on **build-vm** (via the `ssh` MCP or an offloaded worktree), never on desktop-linux.
Checks that only read the repo are cheap enough to run locally:

```sh
devutils/validate_config.py      # patch readability, series duplicates/unused, flags.gn sorted, downloads.ini
devutils/check_all_code.sh       # yapf + pylint + pytest for utils/ and devutils/
```

There is no Taskfile or Makefile. The two commands above are the nearest thing to a gate.

## Build pipeline (on build-vm)

Sequence from `docs/building.md`:

```sh
mkdir -p build/download_cache
./utils/downloads.py retrieve -c build/download_cache -i downloads.ini
./utils/downloads.py unpack  -c build/download_cache -i downloads.ini -- build/src
./utils/prune_binaries.py build/src pruning.list
./utils/patches.py apply build/src patches
./utils/domain_substitution.py apply -r domain_regex.list -f domain_substitution.list \
    -c build/domsubcache.tar.gz build/src
mkdir -p build/src/out/Default && cp flags.gn build/src/out/Default/args.gn
cd build/src && ./tools/gn/bootstrap/bootstrap.py --skip-generate-buildfiles -j4 -o out/Default/
./out/Default/gn gen out/Default --fail-on-unused-args
ninja -C out/Default chrome chromedriver chrome_sandbox
```

## Editing patches with quilt

Run this on the source tree **before domain substitution**. If substitution was already applied,
revert it first with `utils/domain_substitution.py revert -c build/domsubcache.tar.gz build/src`.

```sh
source devutils/set_quilt_vars.sh      # QUILT_PATCHES=<repo>/patches, refresh args -p ab --no-index ...
cd build/src
quilt push -a --refresh                # on failure: quilt push -f, fix, quilt refresh, repeat
# new patch: push to the end first, then
quilt new extra/fingerprint/022-<name>.patch && quilt add <file> && <edit> && quilt refresh
quilt pop -a
cd ../.. && devutils/validate_config.py && devutils/validate_patches.py -l build/src
```

`quilt refresh` rewrites the fingerprint patches from the `diff --git` + `index` form to the bare
`---`/`+++` form. That churn is cosmetic, so expect it.

## Conventions

- **Fingerprint patch names**: `NNN-kebab-name.patch`. The numbers 004, 008, 017 and 019 were never
  in this repo, so the next free number is **022**. Add the new patch to the end of `series`.
- **No patch headers**: the files start straight at `diff --git`.
- **New switch**: add it in 000 (`ungoogled_switches.{cc,h}` + the renderer propagation list), then
  document it in the README switch table.
- **Patch dependencies**: a patch that changes lines another fingerprint patch added has to come
  after it (021 edits 005's hunk).
- **Python**: 3.10 (`.python-version`) with yapf `.style.yapf` (pep8, column limit 100). The pinned
  tools are in `.cirrus_requirements.txt`.

## Upstream and rebasing

- **Remotes**: `origin` (`github.com/jarrin/fingerprint-chromium`) is the only remote. The
  per-version branches `origin/144.0.7559.132` and `origin/148.0.7778.215` are upstream
  ungoogled-chromium history plus one adryfish commit that drops in all the fingerprint patches.
- **History**: `main` = the merge `e258b365` of that source branch with adryfish's README-only
  `main`, followed by translation and README cleanups.
- **To move to a new Chromium version** (inferred from history and `docs/developing.md`):
  1. `git remote add upstream https://github.com/ungoogled-software/ungoogled-chromium`, then merge
     its "Update to Chromium X" commit.
  2. Keep the fingerprint block at the end of `series`.
  3. On build-vm: `devutils/update_lists.py -t build/src`, then quilt-refresh every patch, then
     `validate_patches.py -l build/src`.
  4. Bump the hardcoded version tables in 002 (`kChromiumVersions`, `kEdgeVersions`,
     `kEdgeUAVersion`), because they go stale on every rebase.
  5. Optionally compare with adryfish's per-version branch on github.com/adryfish/fingerprint-chromium.
