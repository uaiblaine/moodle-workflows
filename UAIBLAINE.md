# uaiblaine fork

This is a fork of [moodle-an-hochschulen/moodle-workflows](https://github.com/moodle-an-hochschulen/moodle-workflows).
The workflows, their copyright and their documentation (README.md) are Moodle an Hochschulen e.V.'s;
this fork carries four changes to `.github/workflows/moodle-plugin-ci.yml`, kept as small as possible so
they can be re-applied on top of upstream.

## The changes

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
3. **gherkinlint skips the plugin dependencies' features.** `moodle-plugin-ci grunt` runs core's
   `gherkinlint` task from the Moodle root whenever the plugin has Behat features, and from there core
   lints every `tests/behat/*.feature` in the tree, dependencies included, so `no-dupe-feature-names`
   fails the plugin on two dependencies that share a feature name (`mod_customcert` and
   `tool_certificate` share seven). The Grunt step first writes a `.gherkin-lintignore` at the Moodle
   root, which core's task honours, naming the directory each dependency was installed in (its
   `$plugin->component` placed through core's `lib/components.json`). The plugin's own features are
   still linted against core's, with core's `.gherkin-lintrc`; without dependencies the file is not
   written and the step is upstream's. The shell function is `bin/lib/mah-parity.sh`'s in
   `uaiblaine/moodle-dev`, copied verbatim, where `mdl selfcheck mahparity` tests it and `mdl ci`
   applies it to every leg whose job calls this fork.

4. **Self-hosted runner input.** A new optional input, `runner` (default empty). Every job runs on
   `${{ inputs.runner || 'ubuntu-24.04' }}`, so a caller passing `runner: ${{ vars.CI_RUNNER }}` stays on
   GitHub's runners until the repository variable names a self-hosted label (`mdl-mini`, the Mac mini
   in `uaiblaine/moodle-dev`'s `mdl runner`). On an ARM64 runner the Behat step's Selenium image
   defaults to `selenium/standalone-chromium:4`: `standalone-chrome` has no arm64 manifest.
   `MOODLE_BEHAT_SELENIUM_IMAGE` still overrides it.

Upstream's inputs keep their names and defaults, job names and the file name are upstream's, so callers and tools that read callers
(`uses: .../moodle-plugin-ci.yml`) work unchanged.

## Using it

```yaml
jobs:
  ci-503:
    uses: uaiblaine/moodle-workflows/.github/workflows/moodle-plugin-ci.yml@main
    with:
      moodle-core-branch: MOODLE_503_STABLE
      runner: ${{ (github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository) && vars.CI_RUNNER || '' }}
      plugin-dependencies: |
        uaiblaine/moodle-mod_interactivevideo,main
    secrets:
      PLUGIN_DEPENDENCIES_TOKEN: ${{ secrets.IV_READ_TOKEN }}
```

The token is a fine-grained personal access token with read-only Contents access to the private
dependency repositories only.

The `runner` expression asks for the self-hosted label in `CI_RUNNER` only for the repository's
own code. A pull request from a fork carries the fork's copy of the workflow, so the Mac mini's
dispatcher refuses it whatever it asks for (`uaiblaine/moodle-dev` #111); the expression sends
such a pull request to `ubuntu-24.04` instead of leaving its jobs waiting. Public repositories
also require approval for every external contributor's workflow run.

## Syncing with upstream

Merge upstream's `main` into this fork's `main`; on a conflict in `moodle-plugin-ci.yml`, take
upstream's file and re-apply `uaiblaine/mah-delta.patch` (regenerate it afterwards with
`git diff upstream/main -- .github/workflows/moodle-plugin-ci.yml`).
