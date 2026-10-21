# AINCRAD Ecosystem: Context for Planning

This document is a briefing for anyone (human or model) helping to plan or design work on the AINCRAD ecosystem. It describes what exists, how the pieces connect, the contracts between them, how they're built, what's known to be rough, and what's still open. It reflects the state of the code as of late October 2026. Check the repos before relying on a specific detail.

---

## 1. The idea in one paragraph

AINCRAD is a macOS desktop layer inspired by Sword Art Online's menus. A gesture or hotkey summons a radial system menu (the "launcher"), a themed top bar and dock ("the SAO shell") sit on screen, and a set of companion apps (a browser and a file manager) wear the same theme. The goal is that the whole desktop feels like one world. Everything is themable: colours, sizes, corner style, animation timings and sounds are all data.

Design values that come up repeatedly:
- **One theme, many windows.** AINCRAD owns the theme; companions read it and re-skin live.
- **No servers, no accounts.** Local files, macOS distributed notifications, and UserDefaults. No network dependency for the core features.
- **No extra permissions for the link.** The apps talk through files and notifications, not Accessibility or Full Disk Access (except where a feature genuinely needs it, such as the launcher's app menus and gestures).
- **Private by default.** Private browser windows are never shared with AINCRAD. Downloads announce only file names.
- **Native feel.** AppKit and WebKit, Dock-like behaviour, Finder-like conventions, system shortcuts respected.

---

## 2. The repositories

All repos live under the GitHub account `djag1337`. All are currently private.

| Repo | Local path | Stack | Role |
|---|---|---|---|
| **AINCRAD-Project** | `~/Documents/AINCRAD-Project` | Markdown | Hub: start-here README, screenshots, this document |
| **AINCRAD** | `~/Documents/AINCRAD` | Swift, AppKit, SwiftUI (settings), XcodeGen | The core launcher, shell, settings, widgets, scripting |
| **AINCRAD-Browser** | `~/Documents/AINCRAD-Browser` | Swift, AppKit, WebKit | Native browser that wears the theme |
| **AINCRAD-Discovery** | `~/Documents/AINCRAD-Discovery` | Swift, AppKit, XcodeGen | Finder replacement in the same style |
| **AINCRAD-catalog** | (GitHub) | JSON + assets | Community themes and extensions, read by the Browse pane |

Installed apps live in `/Applications` as `AINCRAD.app`, `AINCRAD Browser.app` and `AINCRAD Discovery.app`.

---

## 3. AINCRAD (the core)

### What it does
- **Launcher:** a radial, column-based menu (`Launcher/MenuView.swift`, `LauncherController.swift`). Opened by a hotkey (default ⌥` in settings) or a mouse-button gesture. Items come from `menu.json`: favourites, apps by category, places, web links, system, plus a "Web" circle and a "System" circle.
- **Shell:** a themed top bar and an SAO dock (`Shell/ShellController.swift`, `ShellDockView.swift`). The dock shows pinned apps, a divider, then running apps; it supports folder stacks, running dots, labels on hover, drag to reorder and drop to pin. It can replace Apple's Dock and menu bar, or show alongside them.
- **App menus:** right-click or click a dock app for its menus, built from Accessibility (`Shell/AppMenuPanel.swift`, `AXMenus`).
- **Settings:** a SwiftUI window with tabs grouped as Launcher (General, Menu, Themes, Widgets), Extend (Extensions, Browse, Quick Commands), Ecosystem (Browser, Discovery) and About (`Settings/SettingsWindow.swift`).
- **Themes:** `Config/Theme.swift`. A theme has colours, font, geometry (sizes), effects, animation timings and sounds. Themes are JSON files in `~/Library/Application Support/AINCRAD/themes/`. A "Copy sizes to…" action carries the numbers between themes and leaves the colours alone.
- **Widgets:** desktop widgets (clock, Claude chat, quests, HUD bits) managed by `Widgets/WidgetManager.swift`. A widget studio lets you lay them out.
- **Productivity ("Command Center"):** clipboard history, captures, notes pad, reminders, quick commands, workspaces (`Productivity/`).
- **Quick commands and items:** user-defined items with optional global shortcuts (`Input/ItemShortcuts.swift`). Shortcuts that clash with Discovery or Browser keys show a warning in the menu editor.
- **Input:** Carbon global hotkeys (`Input/HotKey.swift`), mouse chord gestures (`GestureMonitor`), Magic Mouse and trackpad touch gestures via a private framework (`TouchMonitor`, which can break on a future macOS and then turns itself off), and the ⌘-Tab switcher (`AppSwitcher`, opt-in).
- **Terminal:** a drop-down terminal built on SwiftTerm (`Terminal/DropTerminal.swift`), with a welcome card.
- **Notifications:** macOS notifications can be mirrored as SAO bubbles (`Notifications/`), and the browser's finished downloads appear the same way.
- **Extensions and scripting:** block-based scripts and extension packages (`Extensions/`, `Scripts/`, the Extension Studio). Extensions are data plus scripts that run with approved permissions.
- **System tools:** a system menu (power, network, battery, keep-awake, Mac settings via `MacSettings.swift`), Files menu (zip, trash, show in Finder or Discovery).

### Key files
- `AINCRAD/App/AppDelegate.swift`: wiring, `applyActivation()` and `applyShell()`, which push settings to subsystems.
- `AINCRAD/Config/Settings.swift`: `AppSettings`, the single settings model (JSON in `settings.json`).
- `AINCRAD/Config/ConfigStore.swift`: loads and saves settings, menu, themes; applies size and corner overrides on top of the saved theme.
- `AINCRAD/Config/DefaultMenu.swift`: the default menu, including the Discovery entry in Places.
- `AINCRAD/System/BrowserLink.swift` and `DiscoveryLink.swift`: the two companion links (see §6).
- `AINCRAD/Launcher/AppOpener.swift`: opens and brings apps forward (LaunchServices-based since the macOS 14 focus rules).

### Current state (rough)
- The launcher, shell, settings, themes and widgets are mature and in daily use.
- The dock's click and alt-tab activation has been unreliable on macOS 15. A fix moves to LaunchServices (`open -a` semantics); the user has not yet confirmed it in the running app.
- Full-screen panels (launcher panel, app-menu panel) could occasionally stay up and block the desktop. Safety timers now hide them; the exact trigger is not yet found.
- Settings › Discovery mirrors Discovery's own preferences (same keys, same notification).

---

## 4. AINCRAD-Browser (the companion browser)

- Native AppKit with WebKit (`WKWebView`). Tabs in a strip, a pill-shaped address bar, bookmarks and history in a popover, downloads in a dropdown with progress rings, a reader mode, shortcuts matching common browsers.
- Wears the AINCRAD theme and switches the moment the theme does.
- **Private windows** are isolated (non-persistent data store) and never shared with AINCRAD.
- **Downloads:** attachment and non-displayable responses download automatically. Right-click "Download Image" was broken (WebKit's own item never started a download); it's now replaced by a working item that queries the page for the image under the pointer and downloads it through `startDownload(using:)`.
- **Link to AINCRAD:** writes `~/Library/Application Support/AINCRAD Browser/link.json` (open tabs, bookmarks, recent history; never private windows). Announces changes and finished downloads (file name only) via distributed notifications. AINCRAD's Web circle lists tabs, bookmarks and history from that file.
- Opened by `aincrad-browser://` links: `open?url=…`, `tab?window=…&index=…`, `new`, `private`, `bookmark`, `welcome`, `done`.
- Known: some hosts failed to load in testing (e.g. speed.hetzner.de showed "hostname not found" while curl from the shell succeeded). Cause not investigated.

Key files: `Browser/BrowserViewController.swift` (toolbar, navigation policy, downloads routing), `BrowserTab.swift`, `DownloadManager.swift`, `DownloadingWebView.swift`, `AINCRADLink.swift`, `TabStripView.swift`, `SidebarViewController.swift`, `StartPage.swift`.

---

## 5. AINCRAD-Discovery (the file manager)

- Native AppKit, Finder-like: a sidebar (Favorites and Locations, editable by drag and drop), a toolbar with back/forward/up, list and column views, a path bar, search (Spotlight via `NSMetadataQuery`, with a Where column for results), Quick Look, an info popover, new folder, rename, trash, copy and move with drag and drop.
- **Undo (⌘Z):** reverses moves, renames, copies, new folders and moves to the Trash. Trash restores come from the Trash location the system reports. Undo never deletes.
- **Type to select** in the list, **status bar** with the volume's free space, empty-folder and no-results messages.
- **Window chrome:** a toolbar that tucks away until the pointer reaches the top edge, or stays pinned. Optional AINCRAD stoplights (round buttons in the theme's window colours) replace the system buttons.
- **Theme:** `AINCRADTheme.swift` is a copy of the launcher's theme loader. It reads the same `settings.json` and themes, registers bundled Orbitron and Rajdhani fonts, and listens for theme changes.
- **Icon:** generated by `scripts/make_icon.swift` in the same style as the browser's icon (tile, sky, glowing SAO button), with a folder instead of a globe.

Key files: `Discovery/UI/DiscoveryWindowController.swift` (layout, pins, keyboard, actions, sidebar wiring), `FileTableView.swift` (list, columns, type-ahead), `ColumnBrowserView.swift`, `FinderSidebar.swift`, `DiscoverySettings.swift`, `Discovery/Model/FileOps.swift` (operations), `DiscoveryUndo.swift`, `DiscoveryPrefs.swift` (settings keys), `Discovery/AppDelegate.swift` (menus, opening folders from other apps, restoring windows).

Current state:
- Working and tested in a real window: list, search, columns, undo (new-folder undo verified), sidebar layout, type-to-select, empty states, pinned and auto-hide toolbar, dark mode and drag and drop (confirmed by the user).
- Not yet verified: reopening folders at launch, dragging into the sidebar in the running app (user has since confirmed drag and drop works).
- Known rough edges: column view's empty right side (matches Finder, but could be used better), some menu items duplicate window key handling (harmless), no VoiceOver audit beyond sidebar labels.

---

## 6. The contracts between the apps

This is the most important section for design work. Anything that crosses an app boundary is listed here.

### Bundle identifiers
- AINCRAD: `com.djag.aincrad`
- Browser: `com.djag.aincrad.browser`
- Discovery: `com.djag.aincrad.discovery`

### Files
| What | Where | Written by | Read by |
|---|---|---|---|
| Settings | `~/Library/Application Support/AINCRAD/settings.json` | AINCRAD | AINCRAD, Discovery (theme only) |
| Menu | `~/Library/Application Support/AINCRAD/menu.json` | AINCRAD | AINCRAD |
| Themes | `~/Library/Application Support/AINCRAD/themes/*.json` | AINCRAD (and the catalog) | AINCRAD, Discovery, Browser |
| Browser link | `~/Library/Application Support/AINCRAD Browser/link.json` | Browser | AINCRAD |
| Browser prefs | `~/Library/Application Support/AINCRAD Browser/preferences.json` | Browser | Browser (and mirrored in AINCRAD's Browser pane) |
| Discovery prefs | `com.djag.aincrad.discovery` preferences domain (UserDefaults) | Discovery, and AINCRAD via CFPreferences | both |

### Distributed notifications (Mac-wide, no permissions)
- `com.djag.aincrad.themeChanged`: AINCRAD announces a theme or player name change. Browser and Discovery re-skin.
- `com.djag.aincrad.browser.changed`: browser state changed (tabs, bookmarks, history). AINCRAD refreshes the Web circle.
- `com.djag.aincrad.browser.downloadFinished`: payload `name` only. AINCRAD shows an SAO bubble.
- `com.djag.aincrad.discovery.prefsChanged`: AINCRAD changed Discovery's preferences; Discovery re-reads them and posts an internal change.

### URL schemes
- `aincrad://`: `open` (launcher), `settings/<tab>` (for example `settings/browser`), `key`.
- `aincrad-browser://`: `open?url=…`, `tab?window=W&index=I`, `new`, `private`, `bookmark`, `welcome`, `done`.

### Folder routing
- AINCRAD's `DiscoveryLink` opens folders and reveals files in Discovery when Discovery is installed and "Open folders in Discovery" is on. Otherwise it falls back to Finder. Used by the launcher, the dock's folder stacks, the Files menu, theme and extension folders, the Studio and notes.
- Discovery accepts folders and files from other apps (Open With, or the AINCRAD link) and opens them in a window, selecting the file when one is given.

### Settings keys (Discovery, mirrored in AINCRAD's Settings › Discovery)
`prefs.startFolder`, `prefs.startView`, `prefs.undoLimit`, `prefs.showHidden`, `prefs.reopenWindows`, `prefs.confirmTrash`, `prefs.showFreeSpace`, `prefs.showSearchLocations`, `prefs.showTrashInSidebar`, `prefs.typeToSelect`, `prefs.browserChrome`, `prefs.pinEdges`, plus internal state (`prefs.savedFolders`, `prefs.favoritePaths`, `prefs.lastFolder`). Keep the defaults identical on both sides.

### Themes and sizes
- The launcher, Browser and Discovery all read the same theme JSON. Colours, fonts and sizes come from it. The launcher applies "size and corner" overrides on top of the saved theme without changing the file.

---

## 7. How it's built and installed

- Each project uses XcodeGen (`project.yml` is the source of truth; the `.xcodeproj` is regenerated).
- Each repo has `scripts/build.sh`. It builds Release outside iCloud-synced folders (under `~/Library/Caches/...-build`), copies the app to `/Applications`, re-registers with Launch Services, and verifies the signature. Building replaces the running app on disk: quit it and reopen it to load the change.
- AINCRAD needs `-skipPackagePluginValidation` (the SwiftTerm build plugin).
- Signing: the team identifier is set in `project.yml`; hardened runtime is off for local builds.
- Logs: AINCRAD writes `~/Library/Application Support/AINCRAD/logs/aincrad.log`.
- Tests: there's an `AINCRADTests` target with settings and MacSettings tests. The companions are verified by running them.

---

## 8. Environment and constraints

- macOS 15.7 (Sequoia). Testing machine: a 2018 Mac mini class machine with 8 GB RAM and 6 cores. Keep parallel builds and agents small; never run two heavy builds at once.
- Two displays are common; window coordinates span both.
- Accessibility is granted to AINCRAD for app menus, gestures and some dock actions. Screen Recording is requested by other tools (the Claude app) and is not required by AINCRAD.

---

## 9. Conventions and preferences (for anyone contributing)

- **Commits:** no Claude co-author line. Author as `djag <daggerdylan@gmail.com>`. Don't push or publish without asking. Repos are private by default.
- **Don't kill the user's running apps or games** without asking. Quit gracefully or wait for them to restart; never `pkill` by name a process you didn't start.
- **Finish the whole task.** Build, install, verify what you can, and report what you couldn't verify (for example clicking through UI). Don't stop after each step to ask.
- **Settings must match behaviour.** If a help text says something, the code must do it. The settings audit found several places where it didn't; keep them aligned.
- **Wording:** Settings use "Favorites" (US) and "Minimize" (US) in new text; the codebase still mixes British and American spellings and that is not yet cleaned up.
- **Reporting:** say which reading of an ambiguous request you took, and list what remains unverified.

---

## 10. Known issues and debts

- Dock and ⌘-Tab activation on macOS 15: the fix is in place but unconfirmed in the running app.
- Stuck full-screen panels (launcher or app menus): safety timers mitigate; root cause open.
- The touch gesture uses a private framework and may break on future macOS.
- Two stale Xcode projects in AINCRAD (`AINCRAD 2.xcodeproj`, `AINCRAD 3.xcodeproj`) are untracked leftovers. Awaiting a decision to delete.
- Naming overlap: "Command Center" (productivity) and "Quick Commands" (settings tab) describe overlapping things. Decide one name.
- Retired settings (`shellHideMenuBar`, `shellHideDock`) are ignored, not migrated. This was a deliberate choice; revisit if users complain.
- Discovery's own menu shows some shortcuts that the window handles first (they still work).
- Some hosts fail in the browser; not investigated.
- The launcher's default menu includes Discovery in Places, but a user's own `menu.json` won't pick that up. They need to add it in the menu editor.
- Git history of the hub repo contains earlier screenshots with the owner's real folder names (current screenshots are demo-only). Rewrite history before making the repo public.

---

## 11. Open ideas (not built)

**An assistant, "like Siri, but yours."** Discussed, not started. The leading option is user-owned, not a service run by the developer:
- The user brings their own model access: an API key (stored in the Keychain) or a local model. Nothing goes through a server the developer runs.
- Start with text, typed into the existing SAO input box (`Productivity/SAOPrompt.swift`) from a hotkey.
- Actions are proposals only: the assistant suggests existing quick commands (open an app, set a reminder), and the user confirms.
- Voice: built-in speech synthesis (`AVSpeechSynthesizer`) for replies, which works offline and free. Prerecorded voice clips for fixed SAO-style lines (notifications, system messages). Clips need licensing; cloned voices of real people need consent.
- Open questions: own key, local model, or both; propose-only or autonomous actions; which surface (hotkey box, launcher, widget).

**Discovery:** remaining polish ideas include typing and column keyboard navigation, a dark-mode audit on each theme, a VoiceOver pass over the toolbar and list, and an "undo" for renames and copies across launches.

**Browser:** investigate the hosts that fail to load; a per-site settings view; download manager improvements.

**Ecosystem:** a unified "Reset to defaults" for the companion apps; a single place to see what each app is sharing; an install and update path for the companions (today they're built locally with `scripts/build.sh`).

---

## 12. Questions a planner should ask first

1. Which app does the next piece of work touch, and does it cross a contract in §6? If so, update both sides and this document.
2. Is it something the user can feel in the running app? If so, plan a screenshot or a quick run to check it.
3. Does it need a permission, a network call, or a secret? If yes, stop and ask first.
4. Does it change a default that a user already has saved? If yes, plan a migration or a note.
