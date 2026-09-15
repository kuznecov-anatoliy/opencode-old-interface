<p align="center">
  <a href="https://opencode.ai">
    <picture>
      <source srcset="packages/console/app/src/asset/logo-ornate-dark.svg" media="(prefers-color-scheme: dark)">
      <source srcset="packages/console/app/src/asset/logo-ornate-light.svg" media="(prefers-color-scheme: light)">
      <img src="packages/console/app/src/asset/logo-ornate-light.svg" alt="OpenCode logo">
    </picture>
  </a>
</p>
<p align="center">The open source AI coding agent.</p>
<p align="center">
  <a href="https://opencode.ai/discord"><img alt="Discord" src="https://img.shields.io/discord/1391832426048651334?style=flat-square&label=discord" /></a>
  <a href="https://www.npmjs.com/package/opencode-ai"><img alt="npm" src="https://img.shields.io/npm/v/opencode-ai?style=flat-square" /></a>
  <a href="https://github.com/anomalyco/opencode/actions/workflows/publish.yml"><img alt="Build status" src="https://img.shields.io/github/actions/workflow/status/anomalyco/opencode/publish.yml?style=flat-square&branch=dev" /></a>
</p>

> [!WARNING]
> **Unofficial fork.** This repository is a personal, non-commercial fork of
> [opencode](https://github.com/anomalyco/opencode). It is **not affiliated with,
> endorsed by, or supported by Anomaly or the OpenCode team**. It ships an unsigned
> Windows desktop build with local patches (fixed 60s retry on free-tier limits,
> extended legacy-interface availability) and serves its updates from this fork.
> No warranty of any kind; keep this disclaimer in derivative builds.

# OpenCode — old interface (legacy UI) build · unofficial fork

> [!WARNING]
> **Unofficial fork.** This repository is a personal, non-commercial fork of
> [opencode](https://github.com/anomalyco/opencode). It is **not affiliated with,
> endorsed by, or supported by Anomaly or the OpenCode team**. It ships an unsigned
> Windows desktop build with local patches (fixed 60s retry on free-tier limits,
> extended legacy-interface availability) and serves its updates from this fork.
> No warranty of any kind; keep this disclaimer in derivative builds.

## What is this?

An unofficial Windows build of [OpenCode](https://github.com/anomalyco/opencode) that:

- **Restores the old (legacy) UI** — OpenCode upstream removed the legacy interface on 2026-09-14 (`oldInterfaceSunset`). This build brings it back.
- **Retries free-tier rate limits** — when you hit the free usage cap (`FreeUsageLimitError` / HTTP 429), the build automatically retries every 60 seconds with no cap on attempts.
- **Auto-updates from this repository** — future updates are served directly from this fork. Download once and receive updates automatically.

## Download & install

**Latest release:** https://github.com/kuznecov-anatoliy/opencode-old-interface/releases/latest

**Direct exe download:** https://github.com/kuznecov-anatoliy/opencode-old-interface/releases/latest/download/opencode-desktop-win-x64.exe

### Steps

1. Download the `.exe` file from the link above.
2. Run it. Windows SmartScreen will show a warning because the build is unsigned — click **More info** → **Run anyway**.
3. Done. All future updates will be downloaded automatically from this repository.

## What's different from upstream

| Feature | Upstream | This fork |
|---------|----------|-----------|
| Legacy UI | Removed 2026-09-14 (`oldInterfaceSunset`) | ✅ Restored |
| Free-tier limits | Error + stop | ✅ Retry every 60s, no attempt cap |
| Updates | Official releases only | ✅ Served from this repository |
| Signing | Signed Windows build | ❌ Unsigned (SmartScreen warning) |

## FAQ

**Why did the interface change / where did the old UI go?**
OpenCode upstream removed the legacy interface on September 14, 2026 via the `oldInterfaceSunset` flag. This fork restores it.

**How do I get the old interface back?**
Download and install this fork. The legacy UI is enabled by default.

**What does this build do when I hit the free-tier limit (429 / FreeUsageLimitError)?**
It automatically retries the request every 60 seconds. There is no limit on the number of retry attempts.

**Is this official?**
No. This is a personal, non-commercial fork. It is **not affiliated with, endorsed by, or supported by Anomaly or the OpenCode team**.

**How does it update?**
Updates are served directly from this GitHub repository. When a new release is published, the desktop app will download and install it automatically.

**How do I go back to the official build?**
Uninstall this build and download the official release from https://opencode.ai/download or the upstream GitHub releases.

---
Keywords: opencode old interface, opencode legacy UI, restore old interface, FreeUsageLimitError, free tier retry, rate limit 429.

## По-русски

**Что это?** Неофициальная Windows-сборка OpenCode, которая возвращает старый интерфейс и автоматически повторяет запросы при лимите бесплатных моделей.

**Установка:**
1. Скачайте `.exe` по ссылке выше.
2. Запустите. Windows SmartScreen покажет предупреждение — нажмите **«Подробнее»** → **«Выполнить в любом случае»**.

**Отличия от официальной сборки:**
- Старый интерфейс OpenCode пропал (upstream отключил его 14.09.2026) — наша сборка возвращает его.
- Лимит на бесплатных моделях: вместо ошибки повтор каждые 60 секунд без ограничения попыток.

**Обновления** приходят автоматически из этого репозитория.

**Внимание:** это неофициальная сборка. Не связана с командой OpenCode.

---

<p align="center">
  <a href="README.md">English</a> |
  <a href="README.zh.md">简体中文</a> |
  <a href="README.zht.md">繁體中文</a> |
  <a href="README.ko.md">한국어</a> |
  <a href="README.de.md">Deutsch</a> |
  <a href="README.es.md">Español</a> |
  <a href="README.fr.md">Français</a> |
  <a href="README.it.md">Italiano</a> |
  <a href="README.da.md">Dansk</a> |
  <a href="README.ja.md">日本語</a> |
  <a href="README.pl.md">Polski</a> |
  <a href="README.ru.md">Русский</a> |
  <a href="README.bs.md">Bosanski</a> |
  <a href="README.ar.md">العربية</a> |
  <a href="README.no.md">Norsk</a> |
  <a href="README.br.md">Português (Brasil)</a> |
  <a href="README.th.md">ไทย</a> |
  <a href="README.tr.md">Türkçe</a> |
  <a href="README.uk.md">Українська</a> |
  <a href="README.bn.md">বাংলা</a> |
  <a href="README.gr.md">Ελληνικά</a> |
  <a href="README.vi.md">Tiếng Việt</a>
</p>

[![OpenCode Terminal UI](packages/web/src/assets/lander/screenshot.png)](https://opencode.ai)

---

### Installation

```bash
# YOLO
curl -fsSL https://opencode.ai/install | bash

# Package managers
npm i -g opencode-ai@latest        # or bun/pnpm/yarn
scoop install opencode             # Windows
choco install opencode             # Windows
brew install anomalyco/tap/opencode # macOS and Linux (recommended, always up to date)
brew install opencode              # macOS and Linux (official brew formula, updated less)
sudo pacman -S opencode            # Arch Linux (Stable)
paru -S opencode-bin               # Arch Linux (Latest from AUR)
mise use -g opencode               # Any OS
nix run nixpkgs#opencode           # or github:anomalyco/opencode for latest dev branch
```

> [!TIP]
> Remove versions older than 0.1.x before installing.

### Desktop App (BETA)

OpenCode is also available as a desktop application. Download directly from the [releases page](https://github.com/anomalyco/opencode/releases) or [opencode.ai/download](https://opencode.ai/download).

| Platform              | Download                           |
| --------------------- | ---------------------------------- |
| macOS (Apple Silicon) | `opencode-desktop-mac-arm64.dmg`   |
| macOS (Intel)         | `opencode-desktop-mac-x64.dmg`     |
| Windows               | `opencode-desktop-windows-x64.exe` |
| Linux                 | `.deb`, `.rpm`, or `.AppImage`     |

```bash
# macOS (Homebrew)
brew install --cask opencode-desktop
# Windows (Scoop)
scoop bucket add extras; scoop install extras/opencode-desktop
```

#### Installation Directory

The install script respects the following priority order for the installation path:

1. `$OPENCODE_INSTALL_DIR` - Custom installation directory
2. `$XDG_BIN_DIR` - XDG Base Directory Specification compliant path
3. `$HOME/bin` - Standard user binary directory (if it exists or can be created)
4. `$HOME/.opencode/bin` - Default fallback

```bash
# Examples
OPENCODE_INSTALL_DIR=/usr/local/bin curl -fsSL https://opencode.ai/install | bash
XDG_BIN_DIR=$HOME/.local/bin curl -fsSL https://opencode.ai/install | bash
```

### Agents

OpenCode includes two built-in agents you can switch between with the `Tab` key.

- **build** - Default, full-access agent for development work
- **plan** - Read-only agent for analysis and code exploration
  - Denies file edits by default
  - Asks permission before running bash commands
  - Ideal for exploring unfamiliar codebases or planning changes

Also included is a **general** subagent for complex searches and multistep tasks.
This is used internally and can be invoked using `@general` in messages.

Learn more about [agents](https://opencode.ai/docs/agents).

### Documentation

For more info on how to configure OpenCode, [**head over to our docs**](https://opencode.ai/docs).

### Contributing

If you're interested in contributing to OpenCode, please read our [contributing docs](./CONTRIBUTING.md) before submitting a pull request.

### Building on OpenCode

If you are working on a project that's related to OpenCode and is using "opencode" as part of its name, for example "opencode-dashboard" or "opencode-mobile", please add a note to your README to clarify that it is not built by the OpenCode team and is not affiliated with us in any way.

---

**Join our community** [Discord](https://discord.gg/opencode) | [X.com](https://x.com/opencode)
