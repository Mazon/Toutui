# Toutui — Full Code Review

**What the project is:** a Rust TUI client for Audiobookshelf (ratatui + crossterm + tokio + reqwest + rusqlite) that browses books/podcasts and plays them via a locally-launched VLC controlled over VLC's RC TCP port. State (token, user prefs, last listening session) lives in a SQLite DB under `~/.config/toutui`, and an install/update flow runs a remote bash script. The repo is self-declared archived/beta, which explains (but doesn't excuse) the state below.

---

## 🔒 Security

### 1. Server auth token exposed on the process command line (high)
`src/player/vlc/start_vlc.rs` builds the VLC argument as `format!("{}{}?token={}", server_address, content_url, token.unwrap())`. VLC is spawned as a child process, so any local user (or tool) can read your full Audiobookshelf server token via `ps` / `/proc/*/cmdline` for the entire playback duration. This token grants full API access to your library.
**Fix:** pass the token to VLC out-of-band (env var read by a wrapper, or open the URL via the RC `input` command after connect instead of as a launch argument).

### 2. "Encryption" of the stored token is weak + documented default key (high)
- `src/utils/encrypt_token.rs` uses **magic-crypt** — a home-grown XOR keystream cipher with no authentication. It provides no real protection against anyone who can read both the DB and the `.env`.
- README and `config.example.toml` docs literally recommend `TOUTUI_SECRET_KEY=secret` — a public default key for a "security" feature.
- The `.env` is created 644 by the install script (see #6), and `db.sqlite3` is likewise world-readable by default.
**Fix:** be honest that this is obfuscation, or use a proper KDF+AEAD (e.g. scrypt/argon2 → AES-GCM), generate a random per-install key (never ship "secret" as the example), and `chmod 600` both files.

### 3. Install/update chain of trust is broken (high)
Verified in `hello_toutui.sh`:
- The script itself is fetched with **bare `curl`, no checksum** (README one-liner). The SHA-256 checks inside it only prove the *downloaded artifacts* match hashes *frozen inside the unverified script* — so a compromise of the repo (or MITM of that first fetch) is full RCE as root (`sudo curl`, `sudo tar`).
- Hashes are pinned to the *latest release at script-edit time* while the download URL is `releases/latest` — any new release breaks installs/updates until the script is hand-edited.
- `shasum -a 256` (L71) doesn't exist on most Linux distros → the checksum step is a **silent no-op / guaranteed mismatch** on Linux, i.e. verification effectively doesn't run.
- `curl -L` (L663) has no `-f` (404 pages get written to the archive), no `--proto '=https'`.
- Unverified remote script execution: Homebrew installer (L247) and `sh.rustup.rs | sh` (L284).
- `sudo` is used to write/remove files inside `$HOME` (L654, L687, L987–1025) → root-owned files in user space; uninstall even does `sudo rm -r "$HOME/.config/toutui"`.
**Fix:** pin the release tag, ship a GPG/sigstore-signed `SHA256SUMS` (or switch to distro packages only), `sha256sum` on Linux, `curl -fL --proto '=https'`, no `sudo` inside `$HOME`.

### 4. `--update` / `--uninstall` execute remote code from the app binary (medium)
`src/utils/clap.rs` shells out to `curl | bash` of the same unverified script, with the hash triple-duplicated (binary, README ×3). The stale-hash problem is compounded: the binary you update *with* carries the hash that validates the update.

### 5. ANSI escape injection from remote content (low/medium)
`hello_toutui.sh` L883–887 prints the raw GitHub release body via `echo -e` → escape sequences in a release note could drive xterm escape sequences in the user's terminal. The app's own changelog is static, but the release-body path is remote.

### 6. Secret key entered/written insecurely (low)
Install script reads the key with plain `read` (echoed on screen) and writes `.env` under default umask → 644.

---

## 🐛 Bugs (crashes, races, wrong behavior)

### App

1. **`render_player` panics on a short `player_info` vec** — `src/ui/player_tui.rs` indexes `player_info[0..9]`, but `player_info()` (`src/player/integrated/player_info.rs`) returns a **single-element** vec on `Ok(None)`/`Err`. In `main.rs` the player is rendered whenever `is_vlc_running == "1"`; if the `listening_session` row is missing (DB error, deleted row, crash-recovery gap), the whole app panics mid-render.
2. **`select_last` underflows on empty lists** — `src/app.rs` computes `list.len() - 1` on an empty `Vec` → panic in debug builds, `usize::MAX` selection in release.
3. **`Q` (quit) silently does nothing on the error path** — `src/logic/sync_session/sync_session_from_database.rs`: exit goes through `clean_exit()` (process::exit) from a spawned task; if `get_listening_session()` returns `Err`, nothing exits and the `App.should_exit` flag is **never read** by `main`. The app can end up in a state where Q appears dead.
4. **Playback can wedge forever** — `wait_prev_session_finished` (`src/logic/sync_session/wait_prev_session_finished.rs`) busy-loops (blocking `std::thread::sleep` on a tokio worker) until the previous task sets `is_loop_break = 1`. The previous task only sets it if it got far enough; if it panicked earlier (e.g. `start_vlc`'s `.expect("Failed to execute program")` when VLC is missing, `exec_nc`'s `.expect` when kitty is missing), the flag never flips and every subsequent "play" hangs forever on "Syncing your last session…" until restart.
5. **Refresh kills the app** — `main.rs` does `app = App::new().await?` on `R`; any transient network failure during refresh propagates out of `main` and exits the whole app with an eyre report instead of showing an error.
6. **Startup race on login** (matches known bug `4b3045`): `main.rs` sleeps only **1s** after the login screen before re-checking the DB; `auth_process` spawns as a fire-and-forget task (`src/logic/auth/auth_input.rs`) and does many sequential API calls. Slow server → DB not populated → user is sent back to the login screen and logs in again concurrently; the code comment even admits "maybe add more time (like 6 sec)… it will work at the second attempt".
7. **Token decryption failure is swallowed** — `app.rs` prints the error and continues with the *encrypted blob* as bearer token → `get_all_libraries` 401s → `App::new` returns Err → app exits with a cryptic "Failed to fetch data from the API" and no hint that the real problem is the missing/changed `TOUTUI_SECRET_KEY`.
8. **`pop_message`/`clear_message` panic without a valid config** — `src/utils/pop_up_message.rs` indexes `color[0..2]` on a `Vec` that stays empty when `load_config()` fails; these are called from DB error paths *before* `App` exists, so a missing `config.toml` turns a recoverable error into a panic. Same pattern throughout the render code (`Color::Rgb(bg_color[0], …)`): a user theme with <3 values per color panics everywhere. Validate and clamp at load time.
9. **`alternate_colors` re-reads `config.toml` from disk for every list row, every frame** (`src/ui/tui.rs` `alternate_colors` + `render_list`) — file I/O per row per 200 ms tick. Should be part of the cached `config` on `App`.
10. **Single-row `listening_session` + `DELETE` on insert** (`src/db/crud.rs` `insert_listening_session`) wipes the previous session row while the previous playback loop may still be alive and polling/updating it — the root of the "progress set to 0" class of sync bugs listed in `known_bugs.md`.
11. **`render_info_pod_ep_search` indexes `titles_pod_search[0]`** (`src/ui/tui.rs`) without the empty-check that its sibling `render_info_pod_ep` has — panics when a podcast in search results has a non-empty episode list but missing title metadata.
12. **Unvalidated API responses before parsing** — `src/api/library_items/play_lib_item_or_pod.rs` has no status check (unlike every other API call); a 401/error JSON still produces an `info_item` of `"N/A"`s that flows into `start_vlc`, and `info_item[0].parse::<f64>().unwrap()`-style parses (`handle_l_*.rs`) will panic on any non-numeric value. Same for `duration.parse::<f32>().unwrap()` in `src/api/me/update_media_progress.rs` (4 places) — a zero/empty duration yields `Inf` progress sent to the server.
13. **`search_mode` dead/confused state** — `/` in `handle_key` calls `search_active()` directly, while `render_search_book` *also* calls `self.search_active()` (a blocking event loop!) from inside the render pass whenever `search_mode` is true. `search_mode` is reset on both paths so the render path is (accidentally) dead — but a blocking input loop living inside `frame.render_widget` is a trap waiting for a re-enable.
14. **`exec_nc` hard-codes `kitty`** (`src/player/vlc/exec_nc.rs`) — the `cvlc_term = "1"` option only works in the kitty terminal and panics (`expect`) in any other terminal.
15. **macOS config path inconsistency** — code uses `~/Library/Preferences` (a non-standard location) while the README comment says `Library/Application Support` (`main.rs` vs. README). Users following the README will not find their files.
16. **`check_update` blocks app startup** on an unauthenticated, **timeout-less** GitHub API call every launch and every refresh (60 req/h rate limit → noisy after heavy use).

### Install script (all verified in `hello_toutui.sh`)

17. **L471 typo `pseudo_escape_line`** (undefined) → `grep -E "^"` matches every line → the "preserve user-only config lines" loop never matches → **user config lines not present in `config.example.toml` are silently deleted on every update** (this is known bug `255b86` "losing config after an update").
18. **L71 `shasum -a 256`** doesn't exist on most Linux → checksum verification never works on Linux.
19. **L419–421**: `.env` written 644; key read with plain `read` (visible on screen).
20. **L481 `echo -e`** on merged config content → backslash corruption of user config.
21. **L10/L48 ordering**: `load_exit_codes` runs *after* the root guard and config-dir checks fire → `exit $EXIT_ROOT` with an unset variable is `exit ""` → **exit code 0 on fatal errors** (the README `&&` chain reports success for failed installs).
22. **L646 `EXIT_FAIL` executed as a command**; **L663** no `curl -f`; **L77–79** `rm -rf $tmpdir` on an out-of-scope variable (rm of user's `$tmpdir` if set); **L642–677** old binary removed *before* new one copied → failed update leaves no binary; **L36/182** documented `install_directory` argument is ignored (hardcoded `/usr/local/bin`).
23. **L15 `check_shasum $tmpfile`** unquoted; no `set -u` anywhere so unset variables silently empty out.

---

## 🛠 Design / things that could be better

1. **`App` is a 90-field god struct** holding ~40 parallel lists per view, with three nearly identical `handle_l_*` functions (~250 lines each, copy-pasted with subtle differences) and 15+ CRUD functions that each re-derive the DB path. Extract a `PlayerSession` type (id_session, id_item, duration, …), a single `handle_play()` parameterized by view, and a `Db` handle opened once.
2. **N+1 API calls on every refresh** — `App::new` fetches `get_pod_ep` for *every* podcast sequentially, plus one `get_book_progress` per continue-listening book, plus `check_update`. Use the personalized view's `recent_episode` data already fetched, and run concurrent fetches (`futures::stream::buffer_unordered`) or cache the episode lists.
3. **No HTTP timeouts anywhere** and a fresh `reqwest::Client` per request — share one client with timeouts; a hung server currently freezes startup/refresh indefinitely.
4. **Blocking calls on the async runtime** — `start_vlc`/`exec_nc` call `Command::output()` (blocks until VLC exits, i.e. the whole playback) inside `tokio::spawn`; `wait_prev_session_finished` and `handle_key_player` use `std::thread::sleep`/`thread::blocking`-prone paths. Use `tokio::process` or `spawn_blocking`.
5. **`select_default_usr` flattens a row into a `Vec<String>` consumed by magic indices** (`get(0)`, `get(2)`, `get(5)`…) in `main.rs` and `app.rs` — return the `User` struct; it's one line and removes a whole class of index-drift bugs.
6. **No migrations**: schema changes require deleting the DB (per changelogs). Even a `user_version` + `ALTER TABLE` ladder would save users their data.
7. **`player_info()` rebuilds 10+ strings and opens the DB (×3: session, speed rate, key bindings) every 200 ms** just to draw the player bar; cache and poll less often.
8. **Dead code**: `AppView::SettingsAbout`/`SettingsUpdateUninstall` are unreachable states (render matches to `{}`), `select_last`/`Settings` special-casing, commented-out blocks throughout, unused `should_exit`, `users` field on `Database` never populated.
9. **Error handling philosophy**: most `let _ =` swallowed API/DB errors mean a failed sync is invisible except in the log file the user will never open; surface failures in the TUI (you already have `pop_message` for this).
10. **No tests, no CI for `cargo test`/`clippy`** (the workflow only builds release binaries). The copy-pasted handle/collect code would be trivial to unit-test once deduplicated.
11. `known_bugs.md` lists 5 minor open bugs — several are directly explainable by the root causes above (single-row session table #10, the 1-second login race #6, `render_player` fragility #1).

---

## Suggested priority if you ever un-archive this

1. Token out of the VLC command line (Security #1) and fix the install-script trust chain + Linux `sha256sum` (Security #3, #18) — user-facing security.
2. Kill the panic paths: `render_player` short-vec, `select_last` underflow, `unwrap()`/`parse().unwrap()` on API strings, config color validation (Bugs #1, #2, #8, #12).
3. Fix the `pseudo_escape_line` typo (#17) — it silently destroys user configs on every update.
4. Make Q robust (read `should_exit` in `main`, or return from the sync task) and bound `wait_prev_session_finished` with a timeout (#3, #4).
5. Then the structural work: shared HTTP client with timeouts, dedupe playback handlers, DB handle + migrations, and unit tests.
