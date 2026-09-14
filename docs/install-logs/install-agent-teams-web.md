# AgentTeams install log — `web` profile

- Date: 2026-09-14 (host boot 12:59, install 13:05)
- Target profile: `web` (`C:\Users\Vinci\.dsh\profiles\web`), `DSH_HOME=C:\Users\Vinci\.dsh`
- Host: `@deepseek-ai/dsh@0.1.5-rc.2`, launched as `pnpm dsh web` from `E:\Works\deepseek-harness`
  (live process tree: `pnpm dsh web` → `node --import tsx/esm apps/cli/src/bin.ts web`, listening on
  `127.0.0.1:3080`)
- Plugin: `@nanmicoder/dsh-agent-teams@0.1.18` (npm `latest`), installed with `--save-exact`

## Pre-install baseline

Profile `dependencies`: four `link:` entries (`@wenqi_bian/dsh-dashboard`,
`@wenqi_bian/dsh-task-clock`, `@wenqi_bian/dsh-web-search-anysearch`, `liangwengu`).
Profile `dsh.profile.bundles`: `@deepseek-ai/dsh-base`, `@deepseek-ai/dsh-web-app`,
`@wenqi_bian/dsh-dashboard`, `liangwengu`, `@wenqi_bian/dsh-web-search-anysearch`,
`@wenqi_bian/dsh-task-clock`. `--dump-config` produced 581 lines
(`dump-config-pre.txt`), with no `agent-teams` row.

## Command

```sh
node "C:\Users\Vinci\.dsh\profiles\node_modules\@deepseek-ai\dsh\lib\bin.js" \
  plugin --profile web add --save-exact @nanmicoder/dsh-agent-teams@0.1.18
```

Result: exit 0 (`plugin-add.txt`). pnpm added one dependency and recorded
`minimumReleaseAgeExclude: ['@nanmicoder/dsh-agent-teams@0.1.18']` in the profile's
`pnpm-workspace.yaml`.

## Post-install evidence

- Profile `dependencies` gained `"@nanmicoder/dsh-agent-teams": "0.1.18"` (exact, not a range).
- Profile `dsh.profile.bundles` gained `@nanmicoder/dsh-agent-teams` as the last entry.
- `--dump-config` grew to 587 lines (`dump-config-post.txt`); the only added lines are the
  plugin layer:
  `# == @nanmicoder/dsh-agent-teams`, `- id: agent-teams`,
  `name: '@nanmicoder/dsh-agent-teams'`, `config:`, `stateDir: .agent-teams`,
  `memberProvider: spawn`. No other profile layer changed.
- Package landed at `C:\Users\Vinci\.dsh\profiles\web\node_modules\@nanmicoder\dsh-agent-teams`;
  `package.json` reports `0.1.18`. Every entry point in `exports` exists:
  `lib/index.js`, `lib/client.js`, `cordis.patch.yml`, `package.json`, plus
  `compatibility.json`, `scripts/doctor.mjs`, `README.md`, `README_ZH.md`
  (58 files under `lib/`).

## Known limitations recorded at install time

- `scripts/doctor.mjs --host-root <profile>` resolves the host to `0.1.5-rc.2` and fails with
  `Unsupported host 0.1.5-rc.2; recommended target is 0.1.5-rc.1`. `compatibility.json`
  lists `0.1.5-rc.1`, `0.1.2-rc.1`, `0.1.2-alpha.5`, `0.1.2-alpha.2`; rc.2 is a patch
  ahead of the recommended rc.1 and was not part of the published acceptance matrix. The
  check reports no mixed DSH package identities and no missing packages.
- The running host composed its config tree at boot and only watches
  `cordis.patch.yml` (`patchReload: live`). The install changed the bundle list in
  `package.json`, so the booted process does not hold the plugin: **restart required**.
- Runtime behavior (tool registration, `/agent-teams`, Web panel) was not exercised in this
  install; that needs the restarted host.
