# Fork desktop build notes (`taoofshawn/deepseek-harness`)

How this fork builds, installs, and runs the Windows desktop app, and why it
diverges from upstream. Upstream owns `AGENTS.md` (their contribution guide);
this file is fork-local and survives pulls because upstream never creates it.

## What this fork changes vs upstream

Two commits on `master`, both required for the desktop build to work here:

1. **`fix(desktop): resolve pnpm entry through pnpmInvocation`** — upstream's
   `apps/desktop/scripts/package-target.ts` spawns `process.execPath` with
   `npm_execpath` directly. On machines where pnpm is the standalone native
   executable (`@pnpm/exe` → `pnpm.exe`), Node throws
   `ERR_UNKNOWN_FILE_EXTENSION`. The fix routes through
   `scripts/pnpm-invocation.ts` (already in the repo, handles both the JS
   entrypoint and the `.exe` case) for both the supervised packaging-run path
   and the plain spawn. Without it, `pnpm run package:desktop:win:x64:unsigned`
   fails immediately on this machine.

2. **`ci: add on-demand unsigned Windows installer workflow`** —
   `.github/workflows/build-windows.yaml`. Builds the unsigned installer on a
   GitHub `windows-2022` runner and uploads it as an artifact.

Re-applying after pulls: both are `git cherry-pick`-able onto a fresh
`origin/master`. If upstream lands their own fix for (1) or changes the same
lines, resolve toward upstream and drop the patch.

## Why builds run on GitHub Actions, not locally

The Windows packaging pipeline compiles a native installer-UI DLL
(`apps/installer/window-frame.cpp` via
`apps/desktop/scripts/prepare-windows-installer.ps1`), which requires Visual
Studio C++ Build Tools + Windows SDK (`vswhere` +
`Microsoft.VisualStudio.Component.VC.Tools.x86.x64`). The local machine has
none of these, and installing them into the host OS is a last resort by
decision. The `windows-2022` runner image ships VS Enterprise 2022 with the
C++ workload, Windows SDK, VSWhere, and NSIS 3.10 preinstalled — no local
setup needed. A full run takes ~15 minutes.

Run it from the Actions tab ("Build Windows" → Run workflow) or:

```sh
gh workflow run build-windows.yaml --repo taoofshawn/deepseek-harness --ref master
gh run watch <run-id> --repo taoofshawn/deepseek-harness --exit-status
gh run download <run-id> --repo taoofshawn/deepseek-harness --name deepseek-harness-windows-x64-unsigned
```

The workflow writes `apps/desktop/.env.windows` (git-ignored) from repository
variables, falling back to the example values. Override via repo Settings →
Secrets and variables → Actions → Variables (`DSH_DESKTOP_APP_ID`,
`DSH_DESKTOP_MANDATORY_UPDATE_TEST_ORIGIN`,
`DSH_DESKTOP_MANDATORY_UPDATE_PROD_ORIGIN`).

## Mandatory-update policy: removed for unsigned builds

**[Pending at time of writing — see "Pending work".]** Upstream's
`apps/desktop/scripts/electron-builder-config.mjs` unconditionally resolves
the mandatory-update policy (`resolveDesktopPolicyEnvironment`) and embeds it
into the packaged manifest as `dshMandatoryUpdatePolicy`. The example
`.env.windows` selects the **test** deployment, which sets
`authentication: "feishu-test"` against `https://harness-test.deepseek.com`.

Consequence on first launch of the installed app: a modal "Sign in to the
test environment" window (parented, `modal: true`) blocks the main window
until sign-in. In this build that window renders **blank** (the
`dsh-app://shell/update-dialog.html` document loads with no scripts and no
stylesheets — an empty `<html><head></head><body></body></html>`), so the app
appears hung: blurred (the overlay inserts `body { filter: blur(2px) }` into
the parent), unclickable, and even the native close button is unreachable
because the invisible modal window sits exactly on top of the main window and
swallows all mouse input.

Emergency recovery without a rebuild: attach to the app's CDP
(`--remote-debugging-port=9222` on launch), find the page target
`dsh-app://shell/update-dialog.html` in `http://127.0.0.1:9222/json/list`,
and send `Page.close` over its WebSocket. The overlay closes, the blur is
removed, and the app becomes fully usable for that session. Every fresh
launch re-creates the dialog.

Durable fix directions (pick one):

- **No policy at all (chosen):** make
  `electron-builder-config.mjs` skip `resolveDesktopPolicyEnvironment` and
  omit `dshMandatoryUpdatePolicy` from `extraMetadata` when
  `DSH_DESKTOP_UNSIGNED === '1'`. The app then boots with no update check and
  no policy window (`resolveDesktopPolicyConfig(undefined)` returns
  `undefined` in `apps/desktop/src/mandatory-update-policy.ts` — the "no
  policy" path is supported at runtime; only the build script forces it in).
- **Anonymous auth instead:** package with
  `DSH_DESKTOP_AUTO_UPDATE_ENV=production`, which sets
  `authentication: "anonymous"` against `https://harness.deepseek.com`. No
  sign-in, but the app still polls a DeepSeek endpoint you may not control.

## Pending work (as of 2026-09-18)

- Rebuild without the mandatory-update policy (option above), push, run the
  workflow, reinstall. The currently installed
  `deepseek-harness-0.1.6-alpha.2-win-x64.exe` still carries the test policy.
- Websearch plugin ("build without plugin, add later" decision): the packaged
  app's plugin manager installs only from `registry.npmjs.org`
  (`DESKTOP_REGISTRY` hardcoded in `apps/desktop/src/project-manager.ts`;
  `packageNameFromSpec` rejects `file:` and `://` specs). To use the local
  DuckDuckGo plugin there, publish `dsh-plugin-ddg-search` to npm, then
  install it via the app's plugin manager (menu → Plugins). The plugin source
  lives at `C:\Users\sdrew\code\scratch\dsh\DeepSeek-Harness-Web-Tools` with
  its own git repo; its desktop wiring is documented in its
  `DESKTOP-INSTALL.md`. The local search shim (Python venv, `server.py`) must
  be running on `127.0.0.1:8899` (autostarts via a Startup-folder shortcut).
  Alternative: point `DESKTOP_REGISTRY` at a private registry in this fork
  and rebuild — cheap now that builds run in the fork.

## Sessions: dev home vs installed app

The dev-mode desktop (`start-desktop.cmd` in the repo checkout, which still
works) keeps its harness home at
`apps\desktop\.desktop-build\development\home\`; the installed app uses
`%USERPROFILE%\.dsh\`. They are separate homes; sessions are plain files
under `<home>\sessions\<workspace-slug>\<session-id>\session.v3.jsonl.zstd`.
There is no import UI — to move dev sessions into the installed app, quit the
app and copy the session folders into
`%USERPROFILE%\.dsh\sessions\<same-workspace-slug>\` (workspace slugs are
derived from project paths and must match).

## Build commands (runner and local reference)

```sh
pnpm install --frozen-lockfile
pnpm run build                                # full workspace build
pnpm run package:desktop:win:x64:unsigned     # unsigned NSIS installer
```

The packaging env file `apps/desktop/.env.windows` is required (copy
`.env.windows.example`). `DSH_DESKTOP_APP_ID` must be a reverse-DNS id; this
fork uses `com.deepseek.harness`. Output lands in
`apps/desktop/.desktop-build/targets/win-x64/unsigned-artifacts/`.

## Gotchas learned the hard way

- Git pushes to the fork are rejected when commits carry the real email and
  GitHub email privacy is on. Local repo config is set to
  `29876809+taoofshawn@users.noreply.github.com`; keep it.
- `git pull` on this fork: upstream is `origin` (deepseek-ai/deepseek-harness),
  the fork is `fork`. Pull from `origin`, push to `fork`.
- `fs-ext` is gone from the runtime dependency graph (replaced by the prebuilt
  `@deepseek-ai/node-addon-system/flock`); if an old smoke fixture still
  requires it unconditionally, that's an upstream bug — gate on resolvability.
  (Fixed upstream since 0.1.6-alpha.2.)
- The packaged app opens no web port; the desktop backend prints
  `dsh web: http://127.0.0.1:<port>/?token=...` to stdout when launched with
  `--enable-logging`. That URL is the same UI the window renders — useful for
  separating "backend broken" from "shell broken" when diagnosing.
- Windows runner image reference:
  https://github.com/actions/runner-images/blob/main/images/windows/Windows2022-Readme.md
