# How MacExplorer works (English)

MacExplorer is a file manager for macOS that offers Windows Explorer workflows
(address bar, folder tree, `Ctrl+C/X/V`, `F2`, Delete-to-Trash) while behaving
as a genuine native app (Quick Look, Spotlight, system icons, Light/Dark Mode).

## Five layers

```
UI (SwiftUI + AppKit)
  → ViewModels / State (per-tab state)
    → Domain (pure logic, no UI)
      → Services (touch the real filesystem)
        → macOS APIs (FileManager, Spotlight, Quick Look)
```

## 1. Domain — the brain that never touches the screen

- **NavigationState** holds three things: current location + back history +
  forward history. Back/Forward just swap stacks; navigating somewhere new
  clears the forward stack (browser semantics).
- **PathResolver** converts address-bar text (`~`, `/path`, `file://`) into one
  canonical location before navigating — the address bar and breadcrumbs share
  the same function, so navigation logic exists exactly once.
- **SortConfiguration** sorts (folders-first, 5 columns) off the main thread,
  keeping the UI responsive with 10,000+ files.
- **FileSystemError** maps technical errors to human-readable messages.

## 2. Services — the only layer that touches real files

- Views are **forbidden** from calling `FileManager` directly; everything goes
  through `LocalFileSystemService`. The iron rule: **the filesystem is the
  single source of truth** — the UI updates only *after* the filesystem
  confirms the result. No optimistic "show success and hope".
- **Data safety**
  - Copies go through a staging file followed by one atomic rename — an
    interrupted copy can never leave a half-written destination.
  - Replace backs up the existing item first, then moves the old one to Trash —
    always recoverable, never a silent overwrite.
  - Delete means Move to Trash, never permanent (confirmation is a setting).
  - Moving a folder into itself, moving the volume root, or transferring an
    item onto itself is rejected *before* a single byte is touched.
  - Symlink-aware listings: no traversal loops, no confused hierarchy.
- **DirectoryMonitor** uses OS-level filesystem events (no polling) with a
  250 ms debounce — files added by browsers, Terminal, or Finder appear
  automatically.
- **SpotlightSearch** uses the macOS index for recursive search, cancellable.

## 3. Presentation — every state has one owner

- **BrowserViewModel (one per tab)**: navigation, selection, sort, search, plus
  a **generation token** — navigating to a new folder cancels/discards stale
  loads, so rapid Back/Forward never shows the old folder over the new one.
- **TabStore**: independent state per tab + session restore.
- **ClipboardService**: cut state synced with the system pasteboard — copy in
  Finder, paste in the app (and vice versa).
- **FileOperationService**: owns name-collision dialogs
  (Replace / Keep Both / Skip / Cancel + apply-to-all).
- **WorkspaceService**: opens files in default apps + native Quick Look (Space).

## 4. UI — the right tool per job

- SwiftUI owns the window shell, toolbar, inspector, and settings.
- AppKit (`NSTableView`/`NSOutlineView`) owns the file table and tree, which
  need inline rename, drag & drop, and persistent columns — beyond what
  SwiftUI tables provide.

## 5. Keyboard

- Windows mode: `Ctrl+C/X/V/A/L`, `F2`, `Delete`, `Alt+arrows`.
- macOS mode: `Cmd+C/X/V/A/L` perform the same actions.
- Keystrokes are never intercepted while typing in a text field.

## Example flow: pressing Paste

```
Keypress → collision check + validation → work on background actor
        → filesystem confirms → UI refreshes
```

A name collision always asks first — never overwrites silently. A failure
always reports where the original was preserved.

## New in v0.2.0

- **Progress + Cancel:** large copies stream in 1 MB chunks with a progress bar
  (`Copying A.zip — 12 MB of 48 MB`) and a Cancel button — cancelling removes
  partial destinations, keeps the source intact, and reports no error.
- **Breadcrumb dropdowns:** every `>` opens its segment's sibling folders with
  a checkmark on the current location.
- **Favorites + Recent:** pin folders in the sidebar (drag to add) and revisit
  the last 30 locations from Go → Recent Locations.
