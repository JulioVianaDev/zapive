<p align="center"><img src="ui/zapive.png" alt="ZapZap logo: a crab eating a tomato" width="160"></p>

<h1 align="center">ZapZap</h1>

<p align="center"><strong>Several WhatsApp accounts at once, in one native Linux app.</strong><br>
A fork of <a href="https://github.com/wilssola/zapive">Zapive</a>, written in Rust. One executable, no browser engine.</p>

> [!IMPORTANT]
> ZapZap is an independent project. It is not affiliated with or endorsed
> by WhatsApp or Meta. Unofficial clients may break WhatsApp's terms of
> service; if account safety matters to you, use a secondary number.

ZapZap links to your phone as a WhatsApp companion device, the same way
WhatsApp Web does. It is built on [Zapive](https://github.com/wilssola/zapive)
by [wilssola](https://github.com/wilssola); everything Zapive does
(encrypted storage, calls, offline translation, search, see
[the original documentation](#original-zapive-documentation) below) is
still here. This fork adds what is listed next.

## What this fork adds

### Every account connected at the same time

Upstream Zapive keeps up to five accounts but connects **one at a time**.
ZapZap keeps **every linked account connected simultaneously**:

- Messages, history and notifications keep arriving for all accounts,
  whichever one is on screen.
- The other accounts sit as small avatars in the left rail with their
  unread count; one click switches instantly, without reconnecting.
- A dot on your own avatar means another account has unread messages.
- With more than one account, notifications name the account; clicking
  one switches to it and opens the chat.
- Calls can be answered only on the account on screen; a call to another
  account shows a notification and keeps ringing on the phone.

How it works is described in
[docs/architecture.md](docs/architecture.md#accounts).

### Runs in the background (Linux)

- **Closing the window does not quit.** It hides, and every account stays
  connected, so no message is missed.
- **Open ZapZap again** from the app menu (or click a notification) to
  bring the window back.
- **Quit for real:** right-click the ZapZap icon in the dock → **Quit
  ZapZap**, or run `~/.local/bin/Zapive.AppImage --quit`.
- With **Settings → Start at login** on, ZapZap starts hidden in the
  background after you sign in.

### Notifications that show who and what

- Each banner shows the **contact or group name**, the **message** (in
  groups, `Sender: message`) and the **contact's or group's photo**; a
  photo or video shows its own thumbnail instead.
- Messages in the chat that is open still notify when the window is
  closed or behind another app; a chat only counts as seen while the
  window is visible and focused.
- Muted and archived chats stay quiet unless someone mentions you.
- Previews can be turned off in **Settings → Notifications**.

### Read receipts on

When you open a conversation, the sender sees the blue ticks. Upstream
Zapive keeps receipts off by default; ZapZap turns them on. Turn them off
in **Settings → Privacy**. Messages that arrive while the window is
hidden are not marked as read until you actually open them.

### You choose what downloads

- **Settings → Downloads → In groups** (off by default): with it off,
  nothing in groups downloads by itself, voice notes included. Each item
  keeps its download button, so you fetch only what you want. Private chats
  follow the photo / audio / video / document / sticker switches.
- **Per conversation:** open a chat's or group's info → **More** →
  **Download media automatically here** (or **Stop downloading…**). This
  overrides the default for that one conversation, either way. Turning it
  on fetches what the open chat was holding back.
- Media you send yourself and anything already downloaded always shows.

### Linking a phone: Reload QR code

WhatsApp only hands out QR codes for about two and a half minutes. Before,
the last code stayed on screen after it expired, and scanning it made the
phone say it could not connect. Now an expired code disappears with a
message, and **Reload QR code** under it starts a fresh round at any time.

### Smoother with a lot of history

The first sync of a busy account brings thousands of chats, and the
window could freeze long enough for Ubuntu to say *"not responding"*.
Several causes were fixed:

- The search index is written in one database transaction per batch
  instead of one per message.
- The database's write-ahead log is folded back from a background thread,
  not by whichever save happened to trigger it on the UI thread.
- Indexing old chats for search works in short rounds (about 100 ms) and
  resumes where it stopped, even inside a chat with years of history.
- Chats merged long ago (phone number → WhatsApp's new LID identity) are
  no longer re-indexed on every start.
- Any work that holds the UI thread for more than 150 ms is now logged by
  name in `~/.local/share/zapive/zapive.log` (`[ui] slow: …`), so a new
  freeze can be traced.

### Name, logo and dock

The app is called **ZapZap**, with a new logo, and the window now tells
the desktop which app it belongs to, so the Ubuntu dock shows the ZapZap
icon and name instead of an unknown gear.

Internal names stay `zapive` on purpose: the data folder
(`~/.local/share/zapive`), the keyring entry and the app ID. Renaming them
would disconnect linked accounts and lock the encrypted chats.

## Install on Ubuntu

ZapZap is built from source on your machine and installed for your user
only (no root needed for the install itself).

1. Build tools and run-time packages (Ubuntu 24.04+; on 22.04 use
   `libfuse2` instead of `libfuse2t64`):

   ```sh
   sudo apt install libasound2-dev libgtk-3-dev libxdo-dev cmake build-essential \
       libfuse2t64 libsecret-tools
   curl https://sh.rustup.rs -sSf | sh            # Rust, once
   ```

2. Build, package and install:

   ```sh
   git clone https://github.com/JulioVianaDev/zapive.git && cd zapive
   cargo build --release --locked                 # ~20 min the first time
   mkdir -p build && bash packaging/linux/make-appimage.sh 0.32.1-multiaccount
   bash packaging/linux/install-desktop.sh dist/Zapive-0.32.1-multiaccount-x86_64.AppImage
   ```

3. Press the Super key, type **ZapZap** and open it. Scan the QR code
   (WhatsApp → **Linked devices** → **Link a device**), then
   **avatar (bottom left) → Add account** for each other number.

**Updating:** pull, then repeat step 2. Quit the running copy first
(`~/.local/bin/Zapive.AppImage --quit`) so the new one starts.

**Decline the in-app update prompt:** it downloads the upstream Zapive
release, which connects only one account at a time and lacks the changes
above.

**Uninstall:** `bash packaging/linux/install-desktop.sh --uninstall`
removes the launcher; conversations in `~/.local/share/zapive` are kept.

See [docs/building.md](docs/building.md) for other platforms and the
checks run before each change (`cargo fmt`, `cargo clippy`, `cargo test`).

## Troubleshooting

| Problem | What to do |
| --- | --- |
| Phone says it could not connect | The QR code expired. Click **Reload QR code** and scan within about a minute. |
| "Not responding" right after linking | The first history sync is heavy. Choose **Wait**, not **Force quit**. If it repeats, look for `[ui] slow` lines in `~/.local/share/zapive/zapive.log`. |
| No notifications | Check **Settings → Notifications** in Ubuntu (ZapZap appears after its first banner) and the notification switch in ZapZap's Settings. |
| Dock shows an unknown gear icon | Reinstall with step 2 above, then log out and back in once to refresh GNOME's icon cache. |
| Window gone, app still running | That is the background mode: open ZapZap from the app menu. |

## Credits

ZapZap stands on other people's work:

- **[Zapive](https://github.com/wilssola/zapive)** by
  [wilssola](https://github.com/wilssola) and the Zapive contributors:
  the whole application this fork is built on (MIT License; the copyright
  notice in [LICENSE](LICENSE) is theirs and stays as it is).
- **[whatsapp-rust](https://github.com/jlucaso1/whatsapp-rust)** by
  jlucaso1: the WhatsApp protocol implementation.
- **[Slint](https://slint.dev)**: the UI toolkit.
- **[CTranslate2](https://github.com/OpenNMT/CTranslate2)** and
  **[Argos Translate](https://github.com/argosopentech/argos-translate)**:
  offline translation.
- [opus-pure](https://docs.rs/opus-pure),
  [openh264](https://github.com/ralfbiedert/openh264-rs),
  [pdfium-binaries](https://github.com/bblanchon/pdfium-binaries),
  [Tabler Icons](https://github.com/tabler/tabler-icons) (MIT), Noto Color
  Emoji (OFL) and the Unicode emoji data.
- The logo is a take on **Ferris**, the Rust mascot by Karen Rustad
  Tölva, released to the public domain.
- Multi-account, background mode and the other fork changes:
  [JulioVianaDev](https://github.com/JulioVianaDev), developed with the
  help of Claude Code.

## License

MIT, like upstream Zapive: see [LICENSE](LICENSE). Third-party components
keep their own licenses.

---

# Original Zapive documentation

Everything below comes from upstream Zapive and describes the features
ZapZap inherits. Where it says *Zapive*, it means the same app; links to
releases point to upstream builds, which do not include this fork's
changes.

<p align="center">
  <img src="docs/screenshots/chat-dark-glass.png" alt="Zapive showing a conversation in the dark Liquid Glass style (demo data)" width="900">
</p>

### Highlights

- **Offline translation on your machine.** Messages are translated by
  [CTranslate2](https://github.com/OpenNMT/CTranslate2) running inside the
  app with Argos OPUS-MT models. Once a model is installed, no text leaves
  the computer.
- **Voice and video calls.** One-to-one calls with mute, camera on/off and
  your choice of microphone, speaker and camera.
- **Everything on disk is encrypted.** Chats, message history, media and
  avatars are sealed with AES-256-GCM. The data key is wrapped by the OS key
  store (DPAPI, macOS Keychain or Secret Service) and, if you choose, by a
  PIN.
- **Search without plaintext.** Full-text search on SQLite FTS5 over keyed,
  prefix-preserving word hashes, with encrypted snippets.
- **Privacy by default.** Read receipts stay off, and a message the sender
  deletes stays visible here unless you change either setting. You can also
  blur names, photos, message text and media.
- **Built-in viewers.** PDF preview, a status/stories viewer, a streaming
  video player, and voice notes at up to 3x speed with the pitch preserved.
- **Updates itself.** Checks GitHub Releases, verifies the SHA-256 checksum,
  swaps the binary and shows what's new.
- **Optional AI integration.** A local MCP server (off by default, behind a
  token) lets AI assistants such as Claude or Cursor work with your chats.
- **Light on resources.** About 140–180 MB of RAM with roughly 900 chats
  loaded (see [Performance](#performance)).

### Screenshots

All screenshots come from [demo mode](docs/demo-mode.md) and show invented
data only.

| Dark · Liquid Glass | Light · Liquid Glass |
| --- | --- |
| ![Dark glass](docs/screenshots/chat-dark-glass.png) | ![Light glass](docs/screenshots/chat-light-glass.png) |
| **Dark · Opaque** | **Light · Opaque** |
| ![Dark opaque](docs/screenshots/chat-dark-opaque.png) | ![Light opaque](docs/screenshots/chat-light-opaque.png) |
| **Settings** | **Contact / group info** |
| ![Settings](docs/screenshots/settings.png) | ![Info panel](docs/screenshots/info-panel.png) |
| **Polls** | **Creating a poll** |
| ![Poll with results](docs/screenshots/poll.png) | ![Poll composer](docs/screenshots/poll-composer.png) |
| **Starred messages** | **Reaction details** |
| ![Starred messages](docs/screenshots/starred.png) | ![Reactions](docs/screenshots/reactions.png) |
| **Drag and drop** | **Keyboard shortcuts** |
| ![Dropping files](docs/screenshots/drop.png) | ![Shortcuts](docs/screenshots/shortcuts.png) |
| **Linking a phone** | **Loading** |
| ![Login](docs/screenshots/login.png) | ![Splash](docs/screenshots/splash.png) |

### Download and install

Releases are published at
[github.com/wilssola/zapive/releases](https://github.com/wilssola/zapive/releases).

| Platform | File | Notes |
| --- | --- | --- |
| Windows 10/11 (x64) | `Zapive-Setup-<version>.exe` | Inno Setup installer. The binary is self-signed, so SmartScreen may warn on first run. |
| Linux (x86_64) | `Zapive-<version>-x86_64.AppImage` | `chmod +x` it and run. Install `secret-tool` (libsecret) so the vault key goes into your keyring. |
| macOS 11+ (Apple silicon) | `Zapive-<version>.dmg` | Self-signed: the first time, right-click the app and choose **Open**. |

A Flatpak manifest exists but is not published yet. See
[docs/getting-started.md](docs/getting-started.md) for step-by-step
instructions.

### Quick start

1. Start Zapive. The login screen shows a QR code.
2. On your phone, open WhatsApp → **Settings → Linked devices → Link a
   device**.
3. Scan the QR code, **or** choose to link with a phone number and type the
   8-character code Zapive shows.
4. Your chat list appears as soon as the first sync arrives. History keeps
   arriving in the background.

### Features

Full details are in [docs/features.md](docs/features.md).

**Messaging**
- Chat list with pinned, archived and muted chats, unread and mention
  badges, filters (All, Unread, Favorites, Groups, Archived), custom lists,
  channels and communities.
- WhatsApp formatting (`*bold*`, `_italic_`, `~strike~`, ```` ```mono``` ````),
  replies with a quote, reactions, forwarding, editing, starring, pinning
  and deleting (for me or for everyone).
- **Polls:** create them (up to 12 options, single or multiple answers),
  vote, and see live results with who voted for what.
- **Starred messages** panel (all chats or the current one), a **pinned
  messages banner** under the chat header, and **reaction details**
  grouped by emoji.
- Group read receipts turn blue only when **every** member has read the
  message.
- `@` mentions with a member picker, per-chat drafts, link previews, and a
  multi-message selection mode with copy, forward, delete and export.
- Disappearing messages (off, 24 h, 7 days, 90 days), group member
  management for admins, and chat export to `.txt`.
- Messages queue in an outbox while you are offline and are sent when the
  connection comes back.

**Media**
- Photos, videos, GIFs, stickers, documents and voice notes, sent and
  received. You can crop a photo before sending, paste an image from the
  clipboard, and send a photo as view-once.
- **Drag and drop** up to 10 files onto the window and review each one
  before it is sent.
- Voice notes at 1x, 1.5x, 2x, 2.5x and 3x with the pitch preserved (WSOLA)
  and real waveforms. Recording encodes straight to Ogg/Opus.
- Streaming video player with seek, volume and keyboard controls, an image
  lightbox, a media gallery in the info panel, and an in-app PDF viewer.
- GIF search through Openverse, and automatic downloads you can turn off
  per media type.

**Calls**
- One-to-one voice and video calls with mute, camera toggle and device
  selection, plus a call log. Group calls are not supported yet.

**Privacy and security**
- Encrypted vault, an encrypted search index, an optional PIN on launch,
  read receipts off by default, deletions by the sender kept, and blurring
  of names, photos, text and media. Read the
  [privacy model](#privacy-model).

**Translation**
- Translate any message, or translate a language automatically every time,
  between English, Portuguese, Spanish, French, German, Chinese, Japanese,
  Russian, Italian and Arabic. The source language can be detected
  automatically. See [docs/translation.md](docs/translation.md).

**AI / MCP**
- Opt-in MCP server on `127.0.0.1` with seven tools: list, read and search
  chats, get chat info, send a message, mark as read and react. See
  [docs/mcp.md](docs/mcp.md) before you enable it.

**Appearance and desktop**
- Liquid Glass (translucent) or Opaque style; dark, light or system theme;
  English or Portuguese UI.
- **Chat wallpaper** with custom background and doodle colours (presets
  or any hex value) and doodle intensity.
- **Keyboard shortcuts** for switching, archiving and muting chats,
  searching and editing, with a `Ctrl+/` help screen.
- Native notifications where a click opens the chat, inline reply and a
  play button for voice notes on Windows (open and play on Linux), and a
  message chime you can switch off. See [docs/notifications.md](docs/notifications.md).
- System tray on Windows, start at login, single instance, and up to five
  WhatsApp accounts.

### Privacy model

- **Encrypted at rest:** message history, chat data, media and avatar cache,
  the MCP token, translation settings and the search snippets. A random
  256-bit data key encrypts them with AES-256-GCM. The key is wrapped by
  the OS key store and, with a PIN, also by a key derived from the PIN with
  scrypt.
- **Not encrypted:** whatsapp-rust's protocol store (`wa.db`, protected by
  file permissions), basic app preferences (theme, language, toggles, the
  account list), the log file, and decrypted temporary copies of files you
  open in another app, which are deleted at the next launch.
- **Nothing leaves your computer** except WhatsApp traffic itself, link
  previews, GIF searches, update checks, and the one-time downloads of
  translation models and the PDF engine.

The details, including the MCP risks, are in
[docs/privacy-and-security.md](docs/privacy-and-security.md).

### Performance

Measured with the reproducible harness in [`scripts/bench`](scripts/bench)
on the release build in demo mode (offline mock chats), Windows 11 26H2, Ryzen 7
9800X3D, 100 % scale. Full method and raw numbers:
[docs/benchmarks.md](docs/benchmarks.md).

| Metric | Value |
| --- | ---: |
| Time to first window (median of 5 cold starts) | 210 ms |
| Working set 3 s after start | 76.6 MB |
| Private bytes 3 s after start | 44.5 MB |
| Idle CPU, window visible / minimized / hidden | 0 % |
| CPU while scrolling a chat | 0.05 % of one core |
| Liquid Glass vs opaque style | about +10 MB working set |
| Release executable (single file, LTO, stripped) | 104.7 MiB, plus `ctranslate2.dll` on Windows |

The latest optimisation round cut memory by about 12 %:

| Metric | Before | After |
| --- | ---: | ---: |
| Working set at +3 s | 87.6 MB | 76.6 MB (−12.6 %) |
| Private bytes at +3 s | 50.5 MB | 44.5 MB (−11.9 %) |
| Idle working set, glass | 88.2 MB | 76.5 MB |

Demo mode has no account, so a connected account with many chats, avatars
and media uses more memory than these figures.

Chats load lazily. Opening a chat, or resting the pointer on one for
120 ms, loads its latest messages from the vault, and at most 8 chats stay
"warm" in memory.

### Architecture

| Layer | Crates |
| --- | --- |
| WhatsApp protocol and calls | `whatsapp-rust` (Signal sessions, WebSocket, media, VoIP with the pure-Rust MLOW codec) |
| UI | `slint` (winit backend, Skia software renderer, AccessKit accessibility) |
| Storage | `rusqlite` (SQLite + FTS5), `aes-gcm`, `scrypt`, `hmac`/`sha2` |
| Audio | `opus-pure`, `ogg`, `symphonia`, `cpal` |
| Video and camera | `mp4`, `openh264`, `nokhwa`, `image`, `webp` |
| Translation | `ct2rs` (CTranslate2), `sentencepiece`, `lingua` |
| PDF | `pdfium-render` (pdfium downloaded on first use) |
| MCP | `rust-mcp-sdk`, `axum` |
| Runtime | `tokio` next to the Slint event loop, connected by command channels |

Module map: [docs/architecture.md](docs/architecture.md).

### Build from source

```sh
# Windows (PowerShell): fetch the official CTranslate2 DLL once
./scripts/fetch-ctranslate2.ps1

cargo build --release --locked
```

| Platform | Extra requirements |
| --- | --- |
| Windows | MSVC Build Tools; `scripts/fetch-ctranslate2.ps1` |
| Linux | `libasound2-dev libgtk-3-dev libxdo-dev cmake build-essential` |
| macOS | `cmake` (CTranslate2 is built from source) |

See [docs/building.md](docs/building.md) for checks, self-tests and
packaging.

### Demo mode

Demo mode runs the real UI on invented data in a throwaway directory. It
never connects to WhatsApp and never touches your real data. It is only
compiled with the `demo` feature:

```sh
cargo run --features demo -- --demo                       # explore
cargo run --features demo -- --demo-shots-all docs/screenshots   # regenerate screenshots
```

See [docs/demo-mode.md](docs/demo-mode.md).

### Not yet supported

- Group calls, call links and screen sharing; posting statuses.
- Blocking contacts, creating groups, invite links and `wa.me` deep links.
- Locked chats, auto-lock after inactivity, proxy settings, and
  typing/last-seen privacy settings.
- Sticker packs (import or create), interactive/business messages, sending
  locations, broadcast lists, albums.
- Full screen-reader coverage; more UI languages.
- macOS notifications have no actions, and clicking one does not open the
  chat yet. Linux and macOS builds are tested less than Windows.

### Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) (build, checks, i18n and commit
rules) and [SECURITY.md](SECURITY.md) for reporting vulnerabilities
privately. Release notes are generated from Conventional Commits; see
[docs/releases.md](docs/releases.md).

### Documentation

[docs/index.md](docs/index.md): getting started, features, privacy and
security, translation, MCP, notifications, appearance, architecture,
building, demo mode, releases, troubleshooting and FAQ.
