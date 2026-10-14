# uaiblaine fork

This is a fork of [moodle-an-hochschulen/moodle-workflows](https://github.com/moodle-an-hochschulen/moodle-workflows).
The workflows, their copyright and their documentation (README.md) are Moodle an Hochschulen e.V.'s;
this fork carries one change to `.github/workflows/moodle-plugin-ci.yml`, kept as small as possible so
it can be re-applied on top of upstream.

## The change

1. **Private plugin dependencies.** A new optional secret, `PLUGIN_DEPENDENCIES_TOKEN`. Each entry of
   `plugin-dependencies` is first probed anonymously; a public repository is cloned exactly as upstream
   does. A repository that does not answer anonymously is cloned with the token, through a per-command
   `url.<authenticated>.insteadOf` rewrite of that one repository's URL (`GIT_CONFIG_COUNT`), so nothing
   is written to the git configuration, the clone keeps the public URL as `origin`, and later steps never
   see the token. The token is passed through step `env` and masked. Without a token the step fails with
   an explicit message (forks and Dependabot pull requests receive no secrets).
2. **Development Leftover Checker skips declared third-party code.** Locations listed in the plugin's
   `thirdpartylibs.xml`, and for an `amd/src/*.js` location its `amd/build` `.min.js` and `.min.js.map`,
   are left out of the `TODO` / `@testme` / conflict-marker search: vendored libraries are not the
   plugin's to clean, and AMD modules cannot live in an excluded `vendor/` directory.

Inputs, job names and the file name are upstream's, so callers and tools that read callers
(`uses: .../moodle-plugin-ci.yml`) work unchanged.

## Using it

```yaml
jobs:
  ci-503:
    uses: uaiblaine/moodle-workflows/.github/workflows/moodle-plugin-ci.yml@main
    with:
      moodle-core-branch: MOODLE_503_STABLE
      plugin-dependencies: |
        uaiblaine/moodle-mod_interactivevideo,main
    secrets:
      PLUGIN_DEPENDENCIES_TOKEN: ${{ secrets.IV_READ_TOKEN }}
```

The token is a fine-grained personal access token with read-only Contents access to the private
dependency repositories only.

## Syncing with upstream

Merge upstream's `main` into this fork's `main`; on a conflict in `moodle-plugin-ci.yml`, take
upstream's file and re-apply `uaiblaine/mah-delta.patch` (regenerate it afterwards with
`git diff upstream/main -- .github/workflows/moodle-plugin-ci.yml`).
