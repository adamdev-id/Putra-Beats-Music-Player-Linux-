# Putra Beats Music Player (Linux / Debian)

A Discord music bot tuned for **Linux deployment**, especially **Debian / Ubuntu VPS servers**.

> Windows setup is usually easier. On Linux, YouTube playback is stricter because of cookies, PO tokens, ffmpeg behavior, and yt-dlp extraction differences. This README documents the **working Linux path**.

---

## Features

* `/play <query>` for songs, URLs, and playlists
* `/playsearch <query>` with dropdown search results
* Queue system with persistence to disk
* Pause, resume, skip, stop, queue view
* Filter effects:

  * Bass Boost
  * Nightcore
  * Slow + Reverb
  * Slow
* Button controls in Discord embeds
* Linux-safe streaming pipeline:

  * `yt-dlp` handles YouTube extraction and download
  * `ffmpeg` transcodes to Discord-friendly audio
  * bot streams to voice channel

---

## Why Linux needs a special setup

On Windows, playback often works with less setup.

On Linux, YouTube may block playback unless you have:

* a valid **cookies.txt** file
* `yt-dlp` configured with the correct **client**
* a **PO token provider** for current YouTube restrictions
* **system ffmpeg** instead of some bundled static builds
* a streaming pipeline where **yt-dlp fetches media** and pipes it into ffmpeg

This project uses the working Linux approach.

---

## Requirements

### System

* Debian 12 / 13 or Ubuntu server
* Node.js **20+**
* ffmpeg installed from the OS package manager
* internet access to YouTube

### Node packages

Make sure your project has these dependencies installed:

* `discord.js`
* `@discordjs/voice`
* `yt-dlp-exec`
* `ffmpeg-static` is **not required** for Linux in the final working setup

---

## Recommended Linux architecture

This is the flow used by the Linux version:

```text
yt-dlp -> stdout -> ffmpeg -> Discord voice connection
```

This is important.

Do **not** rely on:

```text
yt-dlp extracts URL -> ffmpeg fetches YouTube URL directly
```

That direct-fetch approach can fail on Linux with **403 Forbidden** even when yt-dlp itself works.

---

## Install system packages

```bash
sudo apt update
sudo apt install -y ffmpeg git curl
```

Check ffmpeg:

```bash
which ffmpeg
ffmpeg -version
```

Expected path:

```bash
/usr/bin/ffmpeg
```

---

## Install Node.js

Use Node.js **20+**.

Check version:

```bash
node -v
npm -v
```

---

## Install project dependencies

From your project folder:

```bash
npm install
```

If you use `yt-dlp-exec`, verify the binary exists:

```bash
ls -la node_modules/yt-dlp-exec/bin
```

---

## Cookies setup (required)

Linux playback may fail with messages like:

* `Sign in to confirm you're not a bot`
* `LOGIN_REQUIRED`
* `403 Forbidden`

To fix this, export a fresh `cookies.txt` file from a logged-in YouTube browser session.

### Best practice

1. Open a fresh **Incognito / Private** browser window
2. Sign in to **YouTube** with your Google account
3. Open:

```text
https://www.youtube.com/robots.txt
```

4. Export cookies using a browser extension such as:

   * **Get cookies.txt LOCALLY**
   * **cookies.txt** (Firefox)
5. Save as:

```text
youtube-cookies.txt
```

6. Upload it to the server:

```bash
/opt/Putra-Beats-Music-Player/secrets/youtube-cookies.txt
```

7. Lock permissions:

```bash
chmod 600 /opt/Putra-Beats-Music-Player/secrets/youtube-cookies.txt
```

### Important notes

* Do **not** commit this file to GitHub
* Re-export cookies if playback stops working later
* Cookies can expire or become invalid

---

## PO Token provider setup (recommended)

Modern YouTube playback on Linux may require a **PO token provider**.

This project uses:

* `bgutil-ytdlp-pot-provider`

### Plugin install

```bash
mkdir -p ~/.config/yt-dlp/plugins
cd /tmp
curl -L -o bgutil-ytdlp-pot-provider.zip \
  https://github.com/Brainicism/bgutil-ytdlp-pot-provider/releases/latest/download/bgutil-ytdlp-pot-provider.zip
cp bgutil-ytdlp-pot-provider.zip ~/.config/yt-dlp/plugins/
```

### Provider server install

```bash
cd ~
git clone --single-branch --branch 1.3.1 https://github.com/Brainicism/bgutil-ytdlp-pot-provider.git
cd ~/bgutil-ytdlp-pot-provider/server
npm ci
npx tsc
nohup node build/main.js > ~/bgutil-provider.log 2>&1 &
```

Check it:

```bash
ps aux | grep build/main.js
tail -n 50 ~/bgutil-provider.log
```

---

## Environment variables

Use these when starting the bot on Linux:

```bash
export YTDLP_COOKIES_FILE=/opt/Putra-Beats-Music-Player/secrets/youtube-cookies.txt
export YTDLP_EXTRACTOR_ARGS=youtube:player_client=mweb
export FFMPEG_PATH=/usr/bin/ffmpeg
unset YTDLP_USER_AGENT
```

### Why `unset YTDLP_USER_AGENT`?

In testing, adding a custom user agent caused mismatches between working manual yt-dlp commands and bot playback behavior.

---

## Start the bot

```bash
node index.js
```

Expected startup log:

```text
✅ Ready: Putra Beats#5511
Using cookies file: /opt/Putra-Beats-Music-Player/secrets/youtube-cookies.txt
Using extractor args: youtube:player_client=mweb
ffmpeg path: /usr/bin/ffmpeg
```

---

## Recommended Linux test command

Before running the bot, test yt-dlp manually:

```bash
/opt/Putra-Beats-Music-Player/node_modules/yt-dlp-exec/bin/yt-dlp \
  -v \
  --js-runtimes node \
  --cookies /opt/Putra-Beats-Music-Player/secrets/youtube-cookies.txt \
  --extractor-args "youtube:player_client=mweb" \
  "https://www.youtube.com/watch?v=1ekZEVeXwek"
```

If this fails, the bot will fail too.

---

## Linux-specific implementation notes

### 1. Use system ffmpeg

Use:

```js
const ffmpegPath = process.env.FFMPEG_PATH || 'ffmpeg';
```

Avoid relying on `ffmpeg-static` for Linux production.

### 2. Use yt-dlp as the network client

Linux was more reliable when:

* `yt-dlp` downloaded/streamed audio itself
* ffmpeg read from `pipe:0`

Instead of letting ffmpeg fetch Googlevideo URLs directly.

### 3. Use `mweb` extractor client

The Linux version is configured with:

```text
youtube:player_client=mweb
```

### 4. Use Node as JS runtime

The working Linux configuration includes:

```js
jsRuntimes: 'node'
```

---

## Common Linux problems and fixes

### Problem: `--no-no-warnings`

Cause:

* passing `noWarnings: false` into `yt-dlp-exec`

Fix:

* do not set `noWarnings: false`
* omit the option entirely unless you want `true`

---

### Problem: `No supported JavaScript runtime could be found`

Fix:

* install Node 20+
* use:

```js
jsRuntimes: 'node'
```

---

### Problem: `Sign in to confirm you're not a bot`

Fix:

* export a fresh working `cookies.txt`
* verify the cookie file path is correct
* use the logged-in YouTube account cookies

---

### Problem: `LOGIN_REQUIRED`

Fix:

* same as above: refresh cookies
* ensure the bot reads the correct cookie file
* keep `player_client=mweb`

---

### Problem: `HTTP 403 Forbidden` from ffmpeg

Cause:

* ffmpeg tried to fetch Googlevideo URLs directly

Fix:

* use `yt-dlp -> stdout -> ffmpeg -> Discord`

---

### Problem: `SIGSEGV` from ffmpeg-static

Fix:

* switch to system ffmpeg:

```bash
/usr/bin/ffmpeg
```

---

### Problem: `Option reconnect not found`

Cause:

* passing `-reconnect` options while ffmpeg input is `pipe:0`

Fix:

* remove reconnect flags when using piped input

---

### Problem: `write EPIPE`

Cause:

* ffmpeg exited early and yt-dlp kept writing into the closed pipe

Fix:

* handle pipe shutdown cleanly
* ignore expected `EPIPE` on ffmpeg stdin

---

## Security notes

* Never commit `youtube-cookies.txt`
* Keep it outside your repo if possible
* Restrict permissions:

```bash
chmod 600 /opt/Putra-Beats-Music-Player/secrets/youtube-cookies.txt
```

* Consider using a dedicated YouTube account for automation

---

## Suggested project layout

```text
/opt/Putra-Beats-Music-Player
├── index.js
├── config.json
├── queue.json
├── package.json
├── node_modules/
└── secrets/
    └── youtube-cookies.txt
```

---

## Example run script

Create `run.sh`:

```bash
#!/usr/bin/env bash
export YTDLP_COOKIES_FILE=/opt/Putra-Beats-Music-Player/secrets/youtube-cookies.txt
export YTDLP_EXTRACTOR_ARGS=youtube:player_client=mweb
export FFMPEG_PATH=/usr/bin/ffmpeg
unset YTDLP_USER_AGENT
cd /opt/Putra-Beats-Music-Player
node index.js
```

Make executable:

```bash
chmod +x run.sh
```

Run:

```bash
./run.sh
```

---

## Optional: systemd service

Example `/etc/systemd/system/putra-beats.service`:

```ini
[Unit]
Description=Putra Beats Music Player
After=network.target

[Service]
Type=simple
User=admin
WorkingDirectory=/opt/Putra-Beats-Music-Player
Environment=YTDLP_COOKIES_FILE=/opt/Putra-Beats-Music-Player/secrets/youtube-cookies.txt
Environment=YTDLP_EXTRACTOR_ARGS=youtube:player_client=mweb
Environment=FFMPEG_PATH=/usr/bin/ffmpeg
ExecStart=/usr/bin/node /opt/Putra-Beats-Music-Player/index.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable putra-beats
sudo systemctl start putra-beats
sudo systemctl status putra-beats
```

---

## Final notes

If Windows feels easier, that is normal.

Linux is more sensitive because YouTube playback depends on:

* cookies
* extractor client selection
* PO tokens
* ffmpeg behavior
* how media is fetched and piped

Once those are configured correctly, Linux becomes stable and production-friendly.

---

## Credits

Built with:

* Discord.js
* @discordjs/voice
* yt-dlp
* ffmpeg
* bgutil-ytdlp-pot-provider
