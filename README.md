# changelog-demo

A tiny project that demonstrates an automated release workflow built around
[git-cliff](https://git-cliff.org) and [`pnpm version`](https://pnpm.io/cli/version).
The only "product" here is `notes.md` — everything else exists to show how
conventional commits can drive versioning, changelogs, commit messages, and git
tags with a single command.

## How it works

One command does everything:

```bash
pnpm release
```

That expands into the following pipeline:

1. **Decide the next version.** `git-cliff --bumped-version` reads every commit
   since the last tag, classifies them via the parsers in `cliff.toml`, and
   returns the next semver (`v0.3.4` → `v0.4.0` for a `feat`, `v0.3.5` for a
   `fix`, `v1.0.0` for a `feat!` / breaking change).
2. **Bump `package.json`.** `pnpm version <x.y.z>` writes the new version.
3. **Regenerate the changelog.** pnpm's built-in `version` lifecycle hook runs
   our `version` script, which calls git-cliff again — this time with
   `--unreleased --prepend CHANGELOG.md` so only the new section is added to
   the top of the existing file.
4. **Commit.** pnpm commits `package.json` + the updated `CHANGELOG.md` using
   the message format defined in `.npmrc` (`chore(release): %s`).
5. **Tag.** pnpm creates `vX.Y.Z` pointing at that commit.

You do not run `git tag`, `git-cliff`, or `git commit` by hand — it's all
wrapped inside `pnpm release`.

## Commit message conventions

Releases are only as good as your commit history, because git-cliff classifies
commits by subject prefix. The parsers configured in `cliff.toml`:

| Prefix                 | Section in changelog      | Version bump |
| ---------------------- | ------------------------- | ------------ |
| `feat:` / `feat(x):`   | 🚀 Features               | minor        |
| `fix:` / `fix(x):`     | 🐛 Bug Fixes              | patch        |
| `refactor:`            | 🚜 Refactor               | patch        |
| `docs:`                | 📚 Documentation          | patch        |
| `perf:`                | ⚡ Performance            | patch        |
| `style:`               | 🎨 Styling                | patch        |
| `test:`                | 🧪 Testing                | patch        |
| `chore:` / `ci:`       | ⚙️ Miscellaneous Tasks    | patch        |
| `revert:`              | ◀️ Revert                 | patch        |
| anything else          | 💼 Other                  | patch        |
| any of the above + `!` | rendered as `[breaking]`  | **major**    |

Scopes are supported: `fix(4428): message` renders as
`*(4428)* Message` in the changelog. Common use is to drop a ticket/task
number in there.

If any single commit in the unreleased range is a `feat`, the next release
bumps minor. If any commit is breaking (`feat!` or `BREAKING CHANGE:` in the
footer), the next release bumps major.

## The files involved

- `package.json` — `release` and `version` scripts are the entry points.
- `cliff.toml` — git-cliff config: the changelog template, the commit parsers
  listed above, and a few behavior flags.
- `.npmrc` — contains `message=chore(release): %s`, which pnpm reads when
  crafting the release commit message.
- `CHANGELOG.md` — auto-maintained. Regenerated or prepended on every release
  depending on the `version` script (see below).

## Configuration knobs

These are the options you're most likely to want to tweak.

### Prepend vs. full regenerate

The `version` script in `package.json` controls how `CHANGELOG.md` is updated:

```jsonc
// Prepend-only (current). Safe if you hand-edit older entries:
"version": "pnpm exec git-cliff --tag v$npm_package_version --unreleased --prepend CHANGELOG.md && git add CHANGELOG.md"

// Full regenerate. Rewrites the whole file from git history on every release:
"version": "pnpm exec git-cliff --tag v$npm_package_version -o CHANGELOG.md && git add CHANGELOG.md"
```

Full regenerate is "pure function of git + cliff.toml" — changes to the
template propagate to all past versions automatically. Prepend is safer if you
want manual edits to stick.

### Auto-tagging

`pnpm version` tags by default. To turn it off, either:

- Pass `--no-git-tag-version` once: `pnpm version <x> --no-git-tag-version`.
- Set it persistently in `.npmrc`:
  ```
  git-tag-version=false
  ```

### Release commit message format

Defined by `message=` in `.npmrc`. `%s` is substituted with the new version.

```
message=chore(release): %s
```

Change it to `release: %s`, `v%s`, or whatever you like. Without this line,
pnpm uses the bare version number (e.g. `0.3.4`) as the commit message.

### Pre/post hooks

If you want something to run before or after the version bump (tests, lint,
build, push), add `preversion` / `postversion` scripts to `package.json`:

```jsonc
"preversion": "pnpm test",
"postversion": "git push && git push --tags"
```

These fire in the order: `preversion` → bump `package.json` → `version` →
commit + tag → `postversion`.

## Releasing, step by step

```bash
# Make some commits using conventional-commit prefixes.
git commit -m "feat: add new thing"
git commit -m "fix(1234): handle edge case"

# Preview what the next version and changelog will look like.
pnpm exec git-cliff --bumped-version      # e.g. v0.4.0
pnpm exec git-cliff --unreleased          # preview the section

# Cut the release. This writes package.json + CHANGELOG.md, commits, and tags.
pnpm release

# Push the branch and the new tag.
git push && git push --tags
```

## Troubleshooting

- **"The bump didn't happen the way I expected."** Run
  `pnpm exec git-cliff --unreleased` to see exactly which commits git-cliff is
  considering and how it grouped them. A single `feat` in the range forces a
  minor bump; anything not matching a parser lands in "💼 Other".
- **"I have uncommitted changes and `pnpm version` refuses to run."** That's
  pnpm's safety check — commit or stash first.
- **"I need to redo a release."** Delete the tag (`git tag -d vX.Y.Z`,
  optionally `git push origin --delete vX.Y.Z`), reset the release commit
  (`git reset --hard HEAD~1`), and run `pnpm release` again. Only do this for
  tags nobody else has pulled.
