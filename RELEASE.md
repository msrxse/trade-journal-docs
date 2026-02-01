# Release Process

This repository uses [Conventional Commits](https://www.conventionalcommits.org/) enforced by [commitlint](https://commitlint.js.org/) and automated releases via [release-please](https://github.com/googleapis/release-please).

## Commit Message Format

```
type(scope): description

[optional body]

[optional footer]
```

### Types

| Type       | Description                    | Triggers Release? |
|------------|--------------------------------|-------------------|
| `feat`     | New feature                    | Yes (minor)       |
| `fix`      | Bug fix                        | Yes (patch)       |
| `docs`     | Documentation only             | No                |
| `style`    | Formatting, semicolons, etc.   | No                |
| `refactor` | Code change (no fix/feature)   | No                |
| `perf`     | Performance improvement        | No                |
| `test`     | Adding/updating tests          | No                |
| `build`    | Build system changes           | No                |
| `ci`       | CI/CD changes                  | No                |
| `chore`    | Maintenance tasks              | No                |
| `revert`   | Revert previous commit         | No                |
| `spike`    | Experimental/research work     | No                |

### Scopes

| Scope      | Description                    |
|------------|--------------------------------|
| `api`      | Changes to the api service     |
| `frontend` | Changes to the frontend        |
| `worker`   | Changes to the worker          |
| `ci`       | Changes to CI config           |
| `deps`     | Dependency updates             |
| `docs`     | Documentation files            |

### Examples

```bash
feat(api): add user authentication endpoint
fix(frontend): resolve chart rendering bug
chore(deps): update dependencies
docs(api): add API documentation
refactor(worker): simplify queue processing
```

### Breaking Changes

For breaking changes, add `!` after the type or include `BREAKING CHANGE:` in the footer:

```bash
feat(api)!: change authentication to OAuth2

# or

feat(api): change authentication to OAuth2

BREAKING CHANGE: API now requires OAuth2 tokens instead of API keys
```

Breaking changes trigger a **major** version bump (0.x.x → 1.0.0).

## Release Workflow

### How It Works

1. **Commit** with conventional format → commitlint validates
2. **Push to main** → release-please workflow runs
3. **Release PR created** → accumulates all releasable changes
4. **Merge Release PR** → tags created, GitHub Releases published
5. **Tag triggers build** → `build-push.yaml` builds production images

### Two Phases of Release-Please

The same workflow runs twice with different behavior:

| Phase | Trigger | What Happens |
|-------|---------|--------------|
| **PR Creation** | Push with `feat`/`fix` | Creates/updates Release PR |
| **Tag Creation** | Merge Release PR | Creates tags + GitHub Releases |

When you merge the Release PR:
1. Release-please workflow runs again (triggered by the merge)
2. Detects the Release PR was merged
3. Creates tags (`api@1.0.0`, etc.)
4. Creates GitHub Releases with changelogs
5. Updates `.release-please-manifest.json` with new versions
6. `build-push.yaml` triggers on the new tag → builds production images

### Flow Diagram

```
Developer commits
       │
       ▼
┌─────────────────┐
│   commitlint    │──── Invalid? ──→ Commit rejected
│   (pre-commit)  │
└────────┬────────┘
         │ Valid
         ▼
    Push to main
         │
         ▼
┌─────────────────┐
│ release-please  │──── No feat/fix? ──→ No PR created
│   (workflow)    │
└────────┬────────┘
         │ Has releasable commits
         ▼
┌─────────────────┐
│   Release PR    │ ← Accumulates changes, updates CHANGELOG
│   (auto-created)│
└────────┬────────┘
         │ Merge PR
         ▼
┌─────────────────┐
│  Tag created    │ → api@1.0.0, frontend@1.2.0
│  GitHub Release │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  build-push.yaml│ → Production Docker images
│   (triggered)   │
└─────────────────┘
```

### Tag Format

Tags follow the pattern `{service}@{version}`:

```
api@1.0.0
frontend@1.2.0
worker@0.3.0
```

## Configuration Files

| File | Purpose |
|------|---------|
| `commitlint.config.ts` | Commit message validation rules |
| `.husky/commit-msg` | Git hook that runs commitlint |
| `.github/utils/release-please-config.json` | Release-please configuration |
| `.github/utils/.release-please-manifest.json` | Current versions of each package |
| `.github/workflows/release-please.yaml` | Workflow that runs release-please |

## Local Setup

After cloning, run:

```bash
npm install
```

This installs husky and commitlint, enabling commit validation locally.

## Manual Release (Emergency)

If you need to release without release-please:

```bash
git tag "api@1.0.0"
git push origin "api@1.0.0"
```

This triggers the build-push workflow directly.
