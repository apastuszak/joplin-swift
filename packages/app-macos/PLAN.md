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
- Don't use the existing REST Data API (port 41184) as the bridge. It doesn't cover sync control, settings or E2EE. Call `lib` directly from the RPC layer instead. The web clipper server can be turned on later as an extra feature.

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
| **M0** | Headless core | `packages/app-macos/core` starts `lib` against a profile directory, runs migrations, configures a sync target, syncs and lists notes. Testable from the command line. |
| **M1** | Bridge | JSON-RPC protocol, Swift client actor, process start/stop/crash restart, event stream. |
| **M2** | Sync setup UI | Choose a sync target (Joplin Cloud/Server, WebDAV/Nextcloud, S3, filesystem; Dropbox/OneDrive via OAuth in the browser), sync status and errors, E2EE master-password prompt, decryption progress. |
| **M3** | Read-only three-pane UI | Notebook tree, tags, note list (sort and search), note display in the editor in read-only mode. |
| **M4** | Editing | Editable CodeMirror with inline rendering; create, delete and move notes and notebooks; to-dos; save via `Note.save`; undo; the conflicts notebook. |
| **M5** | Feature parity | Attachments (drag-drop, paste images), tags, trash, note history (revisions), full-text search, import/export, printing and PDF, web clipper server. |
| **M6** | Mac polish and shipping | Menus and shortcuts, multiple windows, Share extension, Spotlight, App Intents, signing and notarization, auto-update. |
| Later | | Plugins (they assume Electron and React; large effort), the rich-text (TinyMCE) editor, a TextKit 2 native editor. |

### Sync test checkpoints

1. **After M0:** the headless core syncs with a test target: pulls everything down, decrypts, pushes back without changing anything. This is the first proof that the approach works.
2. **After M0–M3 plus the E2EE prompt from M2:** first test in the Mac app. Set up sync, pull, decrypt, and browse everything natively.
3. **After M4:** full two-way test. Edit on the Mac and check the changes on the phone and other desktop apps; check that conflicts are handled.

## Safety rules for testing

- Use a **separate Joplin profile** and a **test sync target**, or a copy of the real one, until M4 has been used against real data for a while.
- Don't point the app at the official desktop app's profile directory (`~/.config/joplin-desktop`) while the Electron app is running.
- Check the result on another device with an official Joplin client pointed at the same test target.

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

1. Check the toolchain: `node -v` (22.x), `yarn -v` (4.17.0), `xcodebuild -version`, and that `yarn install` succeeded, including the `sqlite3` native build.
2. Read `packages/app-cli/app/app.ts`, `packages/app-cli/app/main.js` and `packages/lib/BaseApplication.ts` (`start()`, and the `hasGui()` branches around `reg.setupRecurrentSync` and `reg.scheduleSync`) to see exactly how a headless app boots `lib`.
3. Create the `packages/app-macos/core` workspace package with a minimal entry point that:
   - starts `lib` with a given profile directory;
   - sets up a sync target from arguments or environment (start with Joplin Server/Cloud or WebDAV, whichever test target the user has);
   - runs one sync, unlocks E2EE if a master password is given, and prints a folder and note summary.
4. Run it against the user's test sync target. Confirm with an official client that nothing on the target was changed unexpectedly.
5. Then define the JSON-RPC protocol (M1) and generate the Xcode project with `xcodegen`.

## Open questions for the user

- Which sync target should be supported first for testing (Joplin Cloud/Server, WebDAV/Nextcloud, Dropbox, OneDrive, S3, filesystem)?
- Is E2EE enabled on the real sync target? It should be supported from the first sync test if so.
- App name and bundle identifier (for example `net.cozic.joplin-mac` is upstream's namespace; probably use something of the user's own).
