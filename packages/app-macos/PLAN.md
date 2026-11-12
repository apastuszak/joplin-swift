# Joplin for macOS (native) — Plan and Handoff

This file carries over the planning done in an earlier Claude Code session on a Linux machine. The agent picking this up on the Mac should treat it as the agreed plan and starting context. Decisions marked **Decided** were made by the user; don't re-open them without a good reason.

## Goal

Build a true native macOS front-end for Joplin, based on the current release in this repo (3.7.22, see `packages/app-desktop/package.json`).

- Reuse as much of the existing TypeScript backend (`packages/lib`) as possible.
- **Keep the Joplin database format and cross-platform sync fully compatible** with the official desktop, mobile and CLI apps. This is the top constraint.
- The editor should work like Obsidian's Live Preview: Markdown renders inline, with no separate edit and preview panes.
- macOS only.

## Decisions

- **Decided:** minimum deployment target is **macOS 27**, built with the latest Xcode. Any macOS 27 API may be used. Before relying on a specific new API, check the installed SDK rather than assuming it exists.
- **Decided:** Swift for the UI.
- **Decided:** development happens on the user's Mac with Claude Code running locally, so the agent can build, run and screenshot the app itself.
- **Proposed and accepted:** the architecture below.

## What the repo already provides

- `packages/lib` contains the whole backend: the SQLite database and migrations (`JoplinDatabase.ts`; `existingDatabaseVersions` goes up to 41 plus the migrations in `packages/lib/migrations`), models (`packages/lib/models`), sync (`Synchronizer.ts`, `services/synchronizer/*`, all the `SyncTarget*.ts` and `file-api-driver-*` files), E2EE (`services/e2ee`, including the old and new formats and PPK for sharing), search (`services/search`), revisions, resources and the Data API (`services/rest`).
- `packages/app-cli` proves the backend runs headless under plain Node, with no Electron. It boots through `BaseApplication.start()` (`packages/lib/BaseApplication.ts`, around line 766) and uses `initProfile`, `shim-init-node.ts`, and `sqlite3` 5.1.6 as the native DB driver.
- **The Obsidian-style editor already exists.** The CodeMirror 6 editor in `packages/editor/CodeMirror` has inline rendering extensions in `extensions/rendering/` (images, tables, checkboxes, links, bullet lists, dividers, format characters, inline HTML). The setting `editor.inlineRendering` in `packages/lib/models/settings/builtInMetadata.ts` is described as "Renders markup on all lines that don't include the cursor". There are also `editor.tableEditing` and `editor.imageRendering`.
- The mobile app already hosts this editor in a WebView: see `packages/app-mobile/contentScripts/markdownEditorBundle/contentScript.ts` (`createEditorWithParent`, which calls `createEditor` from `@joplin/editor/CodeMirror`). Use it as the model for the macOS web view bundle.
- Sync compatibility checks: `services/synchronizer/syncInfoUtils.ts` (`appMinVersion_ = '3.7.0'`) and `MigrationHandler.ts` (checks the sync target version against the `syncVersion` setting).

## Architecture

```
┌──────────── Joplin.app (SwiftUI + AppKit) ─────────────┐
│ Sidebar (notebooks/tags) │ Note list │ Editor          │
│   native                 │  native   │ SwiftUI WebView │
│                          │           │ hosting CM6 +   │
│                          │           │ inline rendering│
└───────────────┬─────────────────────────────┬──────────┘
                │ JSON-RPC (stdio or Unix domain socket)
┌───────────────▼────────── joplin-core (bundled Node) ──┐
│ packages/lib: BaseApplication, models, Synchronizer,   │
│ EncryptionService, SearchEngine, RevisionService ...   │
│ Same SQLite schema, same profile layout, same sync     │
└────────────────────────────────────────────────────────┘
```

### Backend process: `joplin-core`

- A small TypeScript entry point in `packages/app-macos/core/` that starts `lib` the same way `app-cli` does, with no GUI.
- It exposes a typed JSON-RPC API for the Swift app, for example: `folders.list`, `notes.list` (folder, tag, search, sort, paging), `note.get`, `note.save`, `note.delete`, `note.move`, `tags.*`, `resources.*`, `sync.start`, `sync.status`, `e2ee.status`, `e2ee.unlock`, `settings.get` / `settings.set`, and `syncTarget.configure` / `syncTarget.check`.
- It pushes events to Swift: item changes (from `ItemChange` / `eventManager` / the reducer), sync progress and errors, decryption progress.
- It ships inside the app bundle as a Node 22 binary plus the compiled JS and the `sqlite3` native module (arm64). It should be signed along with the app.
- **Rule:** Swift never writes to the SQLite database. All writes go through the `lib` models so `updated_time`, `item_changes`, `sync_items`, revisions and encryption stay correct. A read-only Swift connection (for example via GRDB) for very large note lists is a possible later speed-up, and only if measurements show it's needed.
- **Lifecycle:** booting like app-cli proves `lib` runs headless, but app-cli is a one-shot process with `hasGui()` returning `false`, and several things in `BaseApplication.ts` depend on that flag: recurrent sync (`reg.setupRecurrentSync`), `DecryptionWorker.scheduleStart()` after a sync or master key change, `ResourceFetcher.autoAddResources()` after a sync, and the folder and note refresh middleware. The core is a long-running process, so it subclasses `BaseApplication` with `hasGui()` returning `true` (as app-desktop does), runs the `lib` reducer, and starts each background service it needs explicitly: `ResourceService`, `RevisionService`, `DecryptionWorker`, `ResourceFetcher`, recurrent sync and log rotation. Each service has a documented start and stop, and shutdown is ordered: stop timers, wait for an in-progress sync to cancel or finish, `Setting.saveAll()`, close the DB.
- Don't use the existing REST Data API (port 41184) as the bridge. It doesn't cover sync control, settings or E2EE. Call `lib` directly from the RPC layer instead. The web clipper server can be turned on later as an extra feature.

### Bridge contract

- **Versioning:** the first message is a `hello` handshake carrying a protocol version (an integer). Swift refuses to talk to a core with a different major version. The app bundle always ships a matching core, so a mismatch means a broken build, not something to negotiate.
- **Errors:** every error response has a stable `code` (for example `notFound`, `validation`, `e2eeLocked`, `syncInProgress`, `syncTargetError`, `internal`), a human-readable `message`, and optional `data`. Swift branches on `code` only.
- **Idempotent writes:** a save whose response is lost must be safe to retry. New notes and folders get their id from Swift (32-character hex, saved with `isNew: true`), so a retried create can't make a duplicate. Updates send the full changed fields, so repeating one gives the same result. Each mutating request also carries a `requestId`; the core keeps the outcomes of recent `requestId`s and returns the stored result for a duplicate instead of running it again.
- **Ordering:** the core handles writes to the same item one at a time, in the order they arrive.
- **Events and recovery:** each event has an increasing sequence number. Events are hints ("item X changed, source sync"), and Swift re-fetches the data, so a lost event can't leave stale content shown as current. After a reconnect or a core restart, Swift re-sends `hello` with its last sequence number. If the core can't replay from that point (always the case after a restart), it says so and Swift reloads all visible state.
- **Crash restart:** if the core exits unexpectedly, Swift restarts it with backoff and a limit (for example 3 tries in 60 s, then an error). Unsaved editor text stays in Swift memory and is re-sent after the restart.

### Swift app

- SwiftUI throughout, using the current split-view, inspector, toolbar and search APIs.
- Drop to AppKit only where SwiftUI is weak: a very large note list (speed with tens of thousands of notes), parts of the menus and first-responder handling, and drag-and-drop details.
- State: Observation (`@Observable`), fed by events from the backend. No Core Data or SwiftData; the Joplin DB is the only source of truth.
- Swift 6 strict concurrency, with the RPC client as an actor.
- Keychain for sync credentials and the master password, if `lib`'s keychain service can be pointed at it; otherwise bridge it.
- Generate the Xcode project with `xcodegen` from a `project.yml`, so nobody edits `.pbxproj` by hand.

### Editor

- Build `@joplin/editor` (CodeMirror 6, with `editor.inlineRendering` on) into a bundle for the web view, following the mobile `markdownEditorBundle`.
- Host it in WebKit's SwiftUI `WebView` / `WebPage` API.
- Serve images and attachments from the profile's resource directory through a custom URL scheme handler (for example `joplin-resource://<id>`).
- Bridge in the native Find bar, the Edit menu commands, the spell-check and Services menus, and dictation.
- A pure-native editor (TextKit 2 with live preview) is a possible later replacement behind the same bridge. It's not part of the initial plan.

### Editing correctness (must be specified and tested before M4 ships)

These are the main risks of losing data, so the rules are fixed up front:

- **Autosave:** save after a short pause in typing (debounced, about 1 s), and also on blur, note switch, window close and app quit. The editor tracks a dirty flag and the `updated_time` of the version it loaded (`baseUpdatedTime`).
- **Switching notes:** switching starts a flush of the old note's pending save before the new note loads. The save is queued in order and doesn't block the UI. If it fails, the old note's text is kept in memory, the user sees an error, and the save is retried. It is never discarded silently.
- **Quit:** `applicationShouldTerminate` returns `.terminateLater`, reads the current text from every open editor, waits for all save acknowledgements, then tells the core to shut down (see Lifecycle), and only then terminates. If saving fails or the core is down, unsaved text goes to a recovery file in the app's Application Support folder (not the Joplin DB), and the app offers to restore it at next launch.
- **Changes arriving from sync while editing:** the core sends an item-change event with `ItemChange.SOURCE_SYNC`. If the editor has no unsaved changes, it reloads the note in place and keeps the cursor and scroll position where it can. If it has unsaved changes, it doesn't overwrite them. The next save sends `baseUpdatedTime`. If the stored note has changed since then, the core saves the user's text as the note and keeps the version from sync as a conflict copy via `Note.createConflictNote` (the same mechanism `handleConflictAction.ts` uses), so neither version is lost. Exactly which version becomes the conflict copy should match what the official desktop app does; check that before implementing.
- These rules get automated tests in the core (Jest) and a manual checklist for the Mac app.

### Where the code lives

- `packages/app-macos/core/` — TypeScript backend process (a monorepo workspace package).
- `packages/app-macos/JoplinMac/` — Swift sources and `project.yml`.
- Staying in the monorepo means each upstream release brings backend fixes for free. Prefer not to modify `packages/lib`; if a change is unavoidable, keep it small and make it the kind upstream would accept.

### Options considered and rejected

- Full Swift rewrite of the backend: months of work and high risk of breaking sync.
- Running `lib` inside JavaScriptCore: `lib` needs Node's `fs`, `crypto` and the native `sqlite3` module, so large shim layers would have to be rewritten.
- Electron re-skin, Tauri, Catalyst: none is truly native.

## Milestones

| # | Milestone | Contents |
|---|---|---|
| **M0** | Headless core | `packages/app-macos/core` starts `lib` against a profile directory, runs migrations, configures a sync target, syncs and lists notes. Includes the `snapshot` command used by the sync checkpoints. Testable from the command line. |
| **S1** | Spike: signed bundle | A minimal `.app` that bundles Node 22, the compiled core and the `sqlite3` native module, signed with the hardened runtime (and the JIT entitlements Node needs), launches the core, opens a profile and gets an answer over stdio. Proves the packaging works before the UI is built on it. |
| **S2** | Spike: editor | The CodeMirror bundle in a SwiftUI `WebView` with inline rendering on, an image loaded through the `joplin-resource://` scheme handler, and the native Edit menu (undo, redo, cut, copy, paste, select all), Find bar, spell-check and dictation working in it. Proves the editor approach before M3 and M4 rely on it. |
| **M1** | Bridge | JSON-RPC protocol following the bridge contract above, Swift client actor, process start/stop/crash restart, event stream. |
| **M2** | Sync setup UI (one provider) | One sync target, the one chosen for testing (see open questions), end to end: configure it, check it, sync status and errors, E2EE master-password prompt, decryption progress. Other targets and OAuth providers move to M5. |
| **M3** | Read-only three-pane UI | Notebook tree, tags, note list (sort and search), note display in the editor in read-only mode. |
| **M4** | Editing | Editable CodeMirror with inline rendering, following the editing correctness rules above; create, delete and move notes and notebooks; to-dos; save via `Note.save`; undo; the conflicts notebook. |
| **M5** | Feature parity | Remaining sync targets (including Dropbox and OneDrive via OAuth in the browser), attachments (drag-drop, paste images), tags, trash, note history (revisions), full-text search, import/export, printing and PDF, web clipper server. |
| **M6** | Mac polish and shipping | Menus and shortcuts, multiple windows, Share extension, Spotlight, App Intents, notarization, auto-update. |
| Later | | Plugins (they assume Electron and React; large effort), the rich-text (TinyMCE) editor, a TextKit 2 native editor. |

S1 and S2 need the Mac. They can run in parallel with M1, but both must pass before M2 starts.

### Sync test checkpoints

Every checkpoint is judged by comparing snapshots, not by looking around.

- **Profile snapshot:** a `snapshot` command in the core (read-only, through `lib` models) writes a sorted JSON manifest of every item: id, type, parent id, `updated_time`, `user_updated_time`, `created_time`, title, a SHA-256 of the body, to-do fields, tag links, encryption state, and for resources the size and SHA-256 of the blob file. Deleted items and conflict notes are listed separately. The same command runs against a profile from the official client (the CLI app can open a copy of one), so the two can be compared with a plain diff.
- **Target snapshot:** a listing of the sync target with each file's path, size and SHA-256, plus the contents of `info.json`. This works for filesystem and WebDAV targets; for Joplin Server/Cloud, use the item list from its API.
- **Sync report:** the counts from the `Synchronizer` report (`createRemote`, `updateRemote`, `deleteRemote`, `createLocal`, `updateLocal`, `deleteLocal`) are logged for every sync.

1. **After M0:** sync the core into an empty profile from the test target.
   - Profile snapshot equals the official client's snapshot of the same target. Only fields set locally (for example sync times) may differ, and those are listed in the diff tool.
   - All items are decrypted, and every resource blob is downloaded and matches its hash.
   - Then sync three more times with no edits: every sync reports zero remote creates, updates and deletes, and the target snapshot is byte-for-byte unchanged.
2. **After M0–M3 plus the E2EE prompt from M2:** repeat checkpoint 1 from the Mac app, and also check that every notebook, note and attachment in the snapshot is shown in the UI.
3. **After M4:** a scripted two-way test on the test target, with an official client on another device:
   - On the Mac: create, edit, move, tag and delete notes and notebooks, add an attachment, and toggle a to-do. After syncing both sides, the two profile snapshots match.
   - The same set of changes made on the official client, checked the same way on the Mac.
   - A deliberate conflict (same note edited on both sides between syncs) gives the same result as the official clients: one note plus one conflict copy, with no text lost.
   - With E2EE on, everything new that the Mac pushes is encrypted on the target.
   - Again, three no-op syncs with zero remote changes.

## Safety rules for testing

- The native app always uses **its own profile directory**, never shared with the official desktop app (`~/.config/joplin-desktop`) or any other client. This rule is permanent, not just for testing. Two clients on the same profile is not a supported Joplin setup.
- Until the move to real data below, all syncing is against a **test target**: a throwaway account or folder, or a copy of the real target.
- **Moving to real data** happens only when all of these are true:
  1. Checkpoints 1–3 have passed on the test target, and the latest build has passed them again.
  2. There's a verified backup of the real data: a JEX export from an official client **and** a copy of the sync target, with the JEX restored into a scratch profile and its snapshot checked against the original.
  3. The first sync of the real target is pull-only (the `Synchronizer` already supports `syncSteps: ['delta']`), followed by a snapshot comparison against an official client, before any normal two-way sync.
  4. For the first weeks of real use, check another device after each editing session, and keep the backup until the native app has been used for a while with no problems.

## Mac setup (user to install)

- Latest Xcode (open it once), then: `sudo xcode-select -s /Applications/Xcode.app`, `sudo xcodebuild -license accept`, `xcodebuild -runFirstLaunch`.
- Homebrew, then `brew install node@22 git python@3 xcodegen`.
  - The repo needs Node 22.19 or newer (root `package.json` `engines`); use 22 LTS for the `sqlite3` native module.
  - Python is needed to compile the native module.
- Yarn 4.17.0 via `corepack enable`.
- `yarn install` at the repo root.
- Claude Code installed and signed in.
- In System Settings → Privacy & Security, grant the terminal app running Claude Code **Screen Recording** (to screenshot the app) and **Accessibility** (to drive the UI in tests).
- Signing: an Apple ID in Xcode → Settings → Accounts (the free Personal Team is enough for local builds). A paid Apple Developer account is needed only for notarization in M6.
- For sync tests: a throwaway sync target (Joplin Cloud/Server account, WebDAV/Nextcloud folder, or a copy of the real target), plus another device running an official Joplin client.

## Repo conventions (from `CLAUDE.md`; they apply to all TypeScript here)

- Tabs, single quotes, proper types (no `any`), no unneeded return or const type annotations.
- `//` comments only, no JSDoc, and only for a non-obvious why.
- After adding a new TypeScript file, run `yarn updateIgnored` from the root.
- Type-check with `yarn tsc --noEmit`.
- One `describe()` per Jest test file; use `test.each` instead of repeated test blocks.
- Unknown words flagged by cSpell are handled as described in `readme/dev/spellcheck.md`.
- SQL only inside `packages/lib/models`.
- Markdown files: don't hard-wrap paragraphs.
- Commit only when the user asks; branch off `dev` first.

## Next step: M0

1. Check the toolchain: `node -v` (22.x), `yarn -v` (4.17.0), `xcodebuild -version`, and that `yarn install` succeeded, including the `sqlite3` native build. (The core can also be developed on the user's Linux machine. There, run Yarn with `TMPDIR` on disk if `/tmp` is a small tmpfs.)
2. Read `packages/app-cli/app/app.ts`, `packages/app-cli/app/main.js` and `packages/lib/BaseApplication.ts` to see how a headless app boots `lib`, then apply the Lifecycle rules above (`hasGui()` returns `true`, services started explicitly).
3. Create the `packages/app-macos/core` workspace package with an entry point that:
   - starts `lib` with a given profile directory;
   - sets up a sync target from arguments or environment (the one chosen for testing);
   - runs one sync (optionally pull-only), unlocks E2EE if a master password is given, and prints a folder and note summary;
   - has the `snapshot` command described under Sync test checkpoints.
4. Run checkpoint 1 against the user's test sync target.
5. Then on the Mac: spikes S1 and S2, the JSON-RPC protocol (M1), and the Xcode project generated with `xcodegen`.

## Open questions for the user

- Which sync target should be supported first for testing (Joplin Cloud/Server, WebDAV/Nextcloud, Dropbox, OneDrive, S3, filesystem)?
- Is E2EE enabled on the real sync target? It should be supported from the first sync test if so.
- App name and bundle identifier (for example `net.cozic.joplin-mac` is upstream's namespace; probably use something of the user's own).
