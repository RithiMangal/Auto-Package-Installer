# Auto Package Installer

**Repository description:** Native Linux DEB/RPM dependency installer with local-bundle scanning, multi-manager resolution, an internal WebView fallback, direct-link downloading, and verified system installation.

[![Platform](https://img.shields.io/badge/platform-Linux-blue?style=flat-square&logo=linux)](https://www.kernel.org/)
[![Language](https://img.shields.io/badge/language-Rust-orange?style=flat-square&logo=rust)](https://www.rust-lang.org/)
[![UI](https://img.shields.io/badge/UI-egui-blue?style=flat-square)](https://github.com/emilk/egui)
[![Format](https://img.shields.io/badge/package-DEB%20%7C%20RPM-purple?style=flat-square)](https://www.debian.org/)
[![License](https://img.shields.io/badge/license-GPL--3.0--or--later-green?style=flat-square)](https://www.gnu.org/licenses/gpl-3.0.html)
![AppImage](https://img.shields.io/badge/distro-AppImage-red?style=for-the-badge)

Auto Package Installer is a native Linux desktop application for installing local `.deb` and `.rpm` packages together with the dependencies they require. It is designed for vendor driver bundles, offline package archives, manually collected repositories, and systems where a conventional package-manager workflow is inconvenient or incomplete.

The application keeps local packages first, inspects package metadata before acting, searches every package-manager frontend available on the host when network access is required, and verifies the selected package after installation. It provides a graphical workflow without hiding command output or elevation requirements.

## Table of Contents

- [Highlights](#highlights)
- [Supported Package Formats](#supported-package-formats)
- [How It Works](#how-it-works)
- [Dependency Resolution](#dependency-resolution)
- [Internal Browser and Direct Downloads](#internal-browser-and-direct-downloads)
- [Privilege Model](#privilege-model)
- [Supported Package Managers](#supported-package-managers)
- [System Requirements](#system-requirements)
- [Quick Start](#quick-start)
- [Usage Guide](#usage-guide)
- [Security Model](#security-model)
- [Known Limitations](#known-limitations)
- [Building from Source](#building-from-source)
- [Quality Assurance](#quality-assurance)
- [Project Structure](#project-structure)
- [Contributing](#contributing)
- [License](#license)

## Highlights

- Unified DEB and RPM workflow with package inspection through `dpkg-deb` and `rpm`.
- Recursive local-bundle scanning for dependency packages stored in the selected package folder and its subdirectories.
- Direct-dependency presentation for the selected package while retaining recursive dependency planning for installation.
- Linux compatibility checks for package format, CPU architecture, and required local installers before installation.
- Local-first dependency matching with `[LOCAL]`, `[INSTALLED]`, and `[NETWORK]` states.
- Version-aware dependency matching with support for common Debian and RPM comparison operators.
- Shared-library and file-provider discovery through distro-specific `provides` tools.
- Universal host-tool fallback across APT, DNF/DNF5, YUM, Pacman, Zypper, APK, XBPS, EOPKG, and Emerge when available.
- Direct HTTP and HTTPS package-link downloading into the selected package folder.
- Built-in GTK/WebKit browser with no dependency on an external browser.
- Automatic browser-download interception, package validation, folder rescanning, and installation continuation.
- Explicit administrator or `sudo` authentication before privileged installation.
- Live command output, installation verification, and clear download and privilege status popups.
- AppImage packaging with an integrated desktop entry and application icon.

## Supported Package Formats

### Debian packages

DEB metadata is read with `dpkg-deb`. The application extracts the package name, version, architecture, declared dependencies, version constraints, and alternative dependencies. Installation uses `dpkg -i`; APT systems can perform an `apt-get install -f` repair pass when local package dependencies require it.

### RPM packages

RPM metadata is read with `rpm`. The application queries the package name, version and release, architecture, and required capabilities. Installation uses the native package manager when appropriate, with `rpm -Uvh` as a direct fallback.

Native-format packages are strongly recommended. Cross-format operation is intentionally conservative and may require a direct installer that does not participate in normal repository dependency resolution.

## How It Works

```text
Select package
      |
      v
Inspect metadata and Linux compatibility
      |
      v
Scan direct dependencies and matching local files
      |
      v
Build a recursive installation plan
      |
      v
Resolve missing packages and libraries
      |
      v
Authenticate privileged installation
      |
      v
Install dependencies, then the selected package
      |
      v
Verify against dpkg-query or rpm
```

1. The user selects a `.deb` or `.rpm` file.
2. The application validates the file and reads its package metadata.
3. The selected package format and architecture are checked against the current Linux system.
4. The containing directory tree is scanned recursively for same-format package files.
5. Direct dependencies of the selected package are shown with their current state.
6. Matching local packages are used before any network resolver is invoked.
7. Missing package dependencies and library/file requirements are resolved into a dependency-first plan.
8. A download-complete notification is displayed when required files have been saved.
9. Privilege is verified before any system-changing command is attempted.
10. Dependencies are installed before the selected package.
11. The installed package is verified with the system package database.

## Dependency Resolution

The resolver operates in three layers.

### 1. Local package matching

The selected package folder is scanned recursively, with bounded traversal depth and file count. Candidate packages are matched by:

- Package format
- Normalized package name
- Architecture qualification where applicable
- Version constraint where applicable

The user interface shows only dependencies declared directly by the selected package. Unrelated packages found in the same folder are hidden from the matching-files list. Transitive dependencies are still considered internally when the installation plan is built.

### 2. Installed-package detection

The application queries the local package database before attempting a download. Debian systems are checked through `dpkg-query`; RPM systems are checked through `rpm --whatprovides` or the selected package manager.

### 3. Repository and provider resolution

When a dependency is not available locally, the application tries the detected package manager first and then every other supported package-manager frontend found on the host. This means it uses all repository sources configured for tools installed on the system; it does not claim access to package indexes that are not configured on that machine.

Library and file requirements such as `libc.so.6()(64bit)` or `/usr/lib/example.so` are resolved through available provider indexes, including:

- `dnf repoquery`
- `repoquery`
- `yum provides`
- `zypper what-provides`
- `apt-file`
- `pacman -F`
- `apk info -W`
- `eopkg search-file`
- `xbps-query`

All downloaded files are saved beside the selected package, not in an unrelated temporary folder. After download completion, the folder is rescanned automatically.

## Internal Browser and Direct Downloads

### Direct package links

The **Direct Package URL** field accepts `http://` and `https://` links. The built-in download manager:

1. Streams the response into the selected package folder.
2. Rejects HTML pages when a package file was expected.
3. Preserves a safe package filename and avoids overwriting existing files.
4. Validates the downloaded file with the DEB or RPM inspector.
5. Recursively scans the folder and continues installation automatically.

Direct downloads are limited to a 2 GB response and do not accept `file://`, `javascript:`, or other URL schemes.

### Internal WebView fallback

If repository and provider resolution cannot satisfy a dependency, the application displays a confirmation popup. Selecting **YES** opens the built-in GTK/WebKit browser; selecting **NO** only closes the popup.

The internal browser:

- Opens inside the application workflow rather than launching an external browser.
- Starts searches for the unresolved dependency.
- Denies WebKit permission requests such as camera, microphone, geolocation, and notifications.
- Denies new-window requests.
- Redirects browser-initiated package downloads into the selected package folder.
- Validates each completed download before continuing.
- Automatically rescans the folder and resumes installation after a valid package arrives.

CAPTCHA challenges are not bypassed or solved automatically. The user must complete any challenge manually in the internal browser.

## Privilege Model

The graphical application does not run as root. Privileged commands are isolated to the installation phase.

When an administrator or `sudo` password is entered:

- The password is held in memory only for the active installation.
- The password is never written to disk, application configuration, or the activity log.
- `sudo -S -k` is used for explicit password authentication.
- A privilege preflight executes `id -u` before package installation.
- Installation is blocked if authentication fails.

When the password field is blank:

- `pkexec` is preferred for a desktop authentication prompt.
- A non-interactive cached `sudo` session is used when available.
- Privileged commands are never silently executed as the current unprivileged user.

On Ubuntu and many other distributions, the root account is locked. The application therefore expects the password of the logged-in administrator or a user authorized through `sudo`, not the root-account password.

## Supported Package Managers

| Package manager | Typical distributions | Resolution approach | Installation approach |
|---|---|---|---|
| APT | Debian, Ubuntu, Linux Mint, Kali, Pop!_OS | `apt-get download`, `apt download` | `dpkg -i`, optional privileged `apt-get install -f` |
| DNF / DNF5 | Fedora, Rocky Linux, AlmaLinux, RHEL | `dnf download --resolve --alldeps` | `dnf install -y` |
| YUM | RHEL and compatible systems | `yumdownloader`, `yum download` | `yum install -y` |
| Pacman | Arch, Manjaro, EndeavourOS | `pacman -Sw` cache retrieval | `pacman -U --noconfirm` |
| Zypper | openSUSE and SUSE systems | `zypper download` | non-interactive `zypper install` |
| APK | Alpine Linux | recursive `apk fetch` | `apk add --allow-untrusted` |
| XBPS | Void Linux | native repository resolver | `xbps-install` |
| EOPKG | Solus | native package downloader | `eopkg install` |
| Emerge | Gentoo | `emerge --fetch` | direct RPM fallback |
| Unknown | Other Linux systems | explicit unsupported-resolver error | direct installer only when available |

Repository commands and package-manager behavior vary between distribution versions. A command being listed here indicates an implemented strategy, not a guarantee that every repository or package version is available.

## System Requirements

### Current release artifact

- Linux x86_64
- GTK 3
- WebKitGTK 4.1 with JavaScriptCore and Soup 3
- X11 or Wayland desktop session
- `dpkg-deb` for DEB inspection
- `rpm` for RPM inspection
- `sudo` or `pkexec` for privileged installation
- Network access when dependencies are not already present locally

The AppImage contains the application binary, desktop entry, launcher, and icon. System GUI and WebKit runtime libraries are dynamically linked and must exist on the target system.

## Quick Start

### AppImage

1. Download `Auto-Package-Installer-x86_64.AppImage` from the GitHub release.
2. Make it executable:

```bash
chmod +x Auto-Package-Installer-x86_64.AppImage
```

3. Run it:

```bash
./Auto-Package-Installer-x86_64.AppImage
```

4. Select a `.deb` or `.rpm` package.
5. Review compatibility and dependency status.
6. Enter the administrator or `sudo` password, or leave it blank to use the system prompt.
7. Press **INSTALL**.

### Portable archive

If a portable archive is provided, extract it and run the bundled launcher or binary. The executable retains the internal name `gufw-total-controller` for compatibility.

## Usage Guide

| Control | Behavior |
|---|---|
| **BROWSE *.deb / *.rpm** | Selects a package and starts metadata and local-bundle scanning. |
| **INSTALL** | Starts the complete dependency and installation pipeline. |
| **INSTALL (N ONLINE)** | Indicates unresolved package or library requirements that may require network access. |
| **ADMIN / SUDO PASSWORD** | Provides transient password authentication for the current installation. |
| **DIRECT PACKAGE URL** | Starts a direct HTTP or HTTPS package download in the selected folder. |
| **DOWNLOAD & INSTALL** | Downloads, validates, scans, and installs a directly linked package. |
| Dependency list | Shows direct dependencies of the selected package and their current state. |
| **PROGRESS OUTPUT** | Streams repository, privilege, download, and installation output. |
| Window controls | Minimize, maximize or restore, and close the application. |

Recommended workflow for a local bundle:

1. Place the selected package and all known companion packages in one folder.
2. Select the primary package.
3. Review `[LOCAL]` matches before downloading anything.
4. Install using a trusted administrator or `sudo` password.
5. Verify the final result in the activity log.

## Security Model

- Package arguments are passed directly to process APIs; no package name is interpolated into a shell command string.
- Dependency names are validated against a conservative allow-list before repository commands are created.
- Downloaded files are inspected as DEB or RPM packages before they can enter the installation plan.
- Downloaded files are stored in the user-selected package folder.
- The application never runs as root merely to start its user interface.
- A privilege preflight blocks installation if root access cannot be confirmed.
- Password input is excluded from logs and is cleared after the installation session.
- Browser permission and new-window requests are denied.
- CAPTCHA bypass is not implemented.
- The activity log exposes command output for auditability.

Important source-trust boundary: repository downloads inherit the trust configuration of the host package manager. Direct-link and WebView downloads are validated for package structure but are not independently signature-verified by this application. Only use links and package sources you trust.

## Known Limitations

- The current published AppImage targets x86_64.
- WebKitGTK 4.1 and GTK 3 must be installed on the target system.
- Repository availability depends on the package managers and repositories configured on the host.
- CAPTCHA challenges require manual user interaction.
- Browser-downloaded and directly downloaded packages are not independently signature-verified by the application.
- Cross-format installation is best-effort and should be avoided when a native package is available.
- Some package managers expose download and installation as a combined operation; behavior therefore varies by distribution version.
- The interface is English-only.
- There is no drag-and-drop package input in this release.
- The internal executable name remains `gufw-total-controller` for compatibility.

## Building from Source

### Rust toolchain

Rust 1.85 or newer is recommended.

### Debian and Ubuntu build dependencies

```bash
sudo apt update
sudo apt install \
  build-essential \
  pkg-config \
  libgl1-mesa-dev \
  libxkbcommon-dev \
  libgtk-3-dev \
  libwebkit2gtk-4.1-dev
```

### Build and test

Run these commands from the repository root:

```bash
cargo build --release --locked
cargo test
cargo fmt -- --check
cargo check --locked
```

### Build the AppImage

```bash
chmod +x build-appimage.sh
./build-appimage.sh
```

The script builds the release executable, creates the AppDir layout, installs the desktop entry and icon, obtains `appimagetool` when necessary, and produces:

```text
Auto-Package-Installer-x86_64.AppImage
```

If AppImage tooling cannot be obtained, the script can produce a portable archive instead.

## Quality Assurance

The test suite currently contains 18 tests covering:

- Debian alternatives and version constraints
- RPM library and capability filtering
- Package-name normalization
- Numeric version comparison
- Architecture normalization and compatibility
- Direct-only dependency presentation
- DEB and RPM installation-strategy selection
- Universal package-manager ordering
- Package-manager download attempt generation
- Safe browser download filenames
- Browser search URL encoding
- Direct-link URL validation

Recommended pre-submission checks:

```bash
cargo fmt -- --check
cargo check --locked
cargo test
```

## Project Structure

```text
auto-package-installer-backup/
├── src/
│   └── main.rs
├── assets/
│   ├── gufw-total-controller.desktop
│   └── icon.png
├── Cargo.toml
├── Cargo.lock
├── build-appimage.sh
├── README.md
└── RELEASE_NOTES.md
```

The application is intentionally organized as a compact Rust desktop project. UI, package inspection, dependency planning, repository resolution, elevation, and tests are currently maintained in `src/main.rs`.

## Contributing

Contributions are welcome. Before opening a pull request:

1. Run `cargo fmt`.
2. Run `cargo check --locked`.
3. Run `cargo test`.
4. Keep privilege elevation isolated to the installation phase.
5. Do not log, persist, or transmit administrator passwords.
6. Add tests for new dependency parsing, version comparison, and manager-selection behavior.
7. Update `README.md` and `RELEASE_NOTES.md` when user-visible behavior changes.

## License

The project metadata declares the GNU General Public License, version 3 or later (`GPL-3.0-or-later`). Distribution should include the corresponding GPL license text.
