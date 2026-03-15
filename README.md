# Autodesk Fusion 360 on Fedora 43 + KDE Plasma 6 + Wayland using Steam and GE-Proton10-32

> **Status:** Unsupported workaround  
> **Host OS:** Fedora 43  
> **Desktop:** KDE Plasma 6  
> **Session:** Wayland  
> **Runtime:** native Steam + GE-Proton10-32  
> **Installer:** Fusion Client Downloader.exe

---

## Disclaimer

This is **not** an officially supported Autodesk installation path.

Autodesk’s current position is that Fusion cannot be directly installed on Linux and is only available on Windows and macOS. This document records a **working workaround** for Fedora 43 on KDE Plasma Wayland using Steam and GE-Proton.

This procedure was built from a real working install and includes the fixes that were actually required, including:

- Steam + Proton setup
- Autodesk installer download
- Autodesk browser callback handling
- fixing broken `xdg-open` behavior
- solving Plasma Wayland modal dialog z-order problems
- final Wine settings that made the app usable

---

## References

- Autodesk Linux support position:  
  <https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/Is-there-any-way-to-install-Fusion-360-in-Linux.html>

- Autodesk deployment/install article:  
  <https://www.autodesk.com/support/technical/article/caas/sfdcarticles/sfdcarticles/How-to-deploy-Fusion-360.html>

- GE-Proton project:  
  <https://github.com/GloriousEggroll/proton-ge-custom>

- GE-Proton releases:  
  <https://github.com/GloriousEggroll/proton-ge-custom/releases>

- Fedora Proton/Steam docs:  
  <https://docs.fedoraproject.org/en-US/gaming/proton/>

- Fusion installer direct download URL:  
  <https://dl.appstreaming.autodesk.com/production/installers/Fusion%20Client%20Downloader.exe>

---

## What this procedure installs

By the end of this process, you will have:

- native Steam installed
- GE-Proton10-32 installed in Steam
- Autodesk Fusion 360 installed into a Steam-managed Proton prefix
- a reusable launcher script for Fusion 360
- a reusable launcher script for Autodesk Identity Manager
- KDE URI handlers registered for:
  - `adsk://`
  - `adskidmgr://`
- the Wine graphics settings that actually worked:
  - **Allow the window manager to decorate the windows** = **ON**
  - **Allow the window manager to control the windows** = **OFF**
  - **Emulate a virtual desktop** = **OFF**
- Fusion configured to use **OpenGL**

**Wine 10+ for tooling**

Even though Fusion 360 itself runs on **GE-Proton10-32**, the setup and repair workflow assumes that **Wine 10 or newer** is available on the host for supporting tools and compatibility work.

On Fedora 43, this requirement is satisfied by the current `wine` package, which is version **10.15-1.fc43** at the time of writing. Fedora 43 also provides a current `winetricks` package. :contentReference[oaicite:1]{index=1}

Install these before continuing:

~~~bash
sudo dnf install -y wine winetricks
~~~

---

## Known-good final configuration

If you only want the end state before reading the full procedure, this is the working configuration:

- **native Steam**
- **GE-Proton10-32**
- **Fusion Client Downloader.exe**
- installer launched as a **non-Steam game**
- Steam launch options set to:

~~~text
PROTON_LOG=1 PROTON_DUMP_DEBUG_COMMANDS=1 WINEDLLOVERRIDES="mscoree,mshtml,webview2=disabled" DXVK_ASYNC=1 NO_AT_BRIDGE=1 %command% --quiet
~~~

- `xdg-open` must resolve to:

~~~text
/usr/bin/xdg-open
~~~

- Autodesk URI handlers must be registered:
  - `adsk://`
  - `adskidmgr://`

- in `winecfg -> Graphics`:
  - **Allow the window manager to decorate the windows** = ON
  - **Allow the window manager to control the windows** = OFF
  - **Emulate a virtual desktop** = OFF

- inside Fusion:
  - **Preferences -> General -> Graphics Driver -> OpenGL**

---

# 1. Prerequisites

This guide assumes:

- Fedora 43 is already installed
- KDE Plasma 6 is being used
- the session is Wayland
- internet access is available
- you have `sudo` access
- you want a **Steam-based GE-Proton install**
- you are **not** using Bottles, Lutris, Docker, Podman, distrobox, or WinBoat for this procedure

---

# 2. Update Fedora and install basic tools

Open a terminal and run:

~~~bash
sudo dnf upgrade --refresh -y
sudo dnf install -y curl tar
~~~

These are needed for:

- downloading GE-Proton from GitHub
- extracting the release tarball
- general shell setup

---

# 3. Enable RPM Fusion and install native Steam

Install Steam through RPM Fusion.

Run:

~~~bash
sudo dnf install -y \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

sudo dnf install -y steam
~~~

After installation:

1. Launch **Steam** once from the application menu
2. Let Steam update itself
3. Sign in if needed
4. Close Steam completely

Now verify Steam created one of its normal directories:

~~~bash
ls -ld ~/.steam/steam ~/.local/share/Steam 2>/dev/null
~~~PREFIX="$(cat ~/.config/fusion360-proton/prefix)"

if [ -d "$HOME/.steam/steam" ]; then
  STEAMROOT="$HOME/.steam/steam"
else
  STEAMROOT="$HOME/.local/share/Steam"
fi

PROTON="$STEAMROOT/compatibilitytools.d/GE-Proton10-32/proton"
if [ ! -x "$PROTON" ]; then
  PROTON="$(find "$STEAMROOT/compatibilitytools.d" -maxdepth 2 -path '*/GE-Proton*/proton' | sort -V | tail -n1)"
fi

env STEAM_COMPAT_DATA_PATH="$PREFIX" \
    STEAM_COMPAT_CLIENT_INSTALL_PATH="$STEAMROOT" \
    "$PROTON" run winecfg

At least one of those paths should exist.

---

# 4. Verify `xdg-open` is not hijacked

This matters because Fusion’s sign-in flow depends on:

1. Fusion launching the browser
2. browser authentication succeeding
3. the browser handing a callback back into Fusion

If `xdg-open` is broken or replaced with a user wrapper, sign-in callbacks can fail.

Check what `xdg-open` actually is:

~~~bash
which xdg-open
readlink -f "$(which xdg-open)"
~~~

Expected output:

~~~text
/usr/bin/xdg-open
/usr/bin/xdg-open
~~~

If you instead get something like:

~~~text
/home/youruser/.local/bin/xdg-open
~~~

inspect it:

~~~bash
sed -n '1,200p' ~/.local/bin/xdg-open
~~~

If it is a wrapper you do not want, move it out of the way:

~~~bash
mv ~/.local/bin/xdg-open ~/.local/bin/xdg-open.bak
hash -r
which xdg-open
~~~

Test that browser launching works:

~~~bash
xdg-open https://example.com
~~~

Your default browser should open.

---

# 5. Download and install GE-Proton10-32

This guide uses **GE-Proton10-32** specifically because that is the version that matched the working setup.

The script below:

- downloads GE-Proton10-32 from GitHub
- downloads the checksum
- verifies the checksum
- installs it into Steam’s compatibility tools directory

Run:

~~~bash
set -euo pipefail

TAG="GE-Proton10-32"
API_URL="https://api.github.com/repos/GloriousEggroll/proton-ge-custom/releases/tags/$TAG"

rm -rf /tmp/proton-ge-custom
mkdir -p /tmp/proton-ge-custom
cd /tmp/proton-ge-custom

tarball_url="$(curl -fsSL "$API_URL" | grep browser_download_url | cut -d\" -f4 | grep "${TAG}\.tar\.gz$")"
checksum_url="$(curl -fsSL "$API_URL" | grep browser_download_url | cut -d\" -f4 | grep "${TAG}\.sha512sum$")"

curl -fL "$tarball_url" -o "${TAG}.tar.gz"
curl -fL "$checksum_url" -o "${TAG}.sha512sum"

sha512sum -c "${TAG}.sha512sum"

mkdir -p ~/.steam/steam/compatibilitytools.d
tar -xf "${TAG}.tar.gz" -C ~/.steam/steam/compatibilitytools.d/
~~~

Verify it installed:

~~~bash
find ~/.steam/steam/compatibilitytools.d -path '*/GE-Proton10-32/proton' -print
~~~

Expected output will look like:

~~~text
/home/youruser/.steam/steam/compatibilitytools.d/GE-Proton10-32/proton
~~~

Now restart Steam so it detects the new compatibility tool.

---

# 6. Download the Autodesk Fusion installer

Use the **Fusion Client Downloader.exe** installer.

Direct download URL:

~~~bash
wget https://dl.appstreaming.autodesk.com/production/installers/Fusion%20Client%20Downloader.exe
~~~

Download it to a simple path with a simple filename:

~~~bash
mkdir -p ~/fusion-test
curl -fL 'https://dl.appstreaming.autodesk.com/production/installers/Fusion%20Client%20Downloader.exe' \
  -o ~/fusion-test/fusion-client-downloader.exe
ls -lh ~/fusion-test/fusion-client-downloader.exe
~~~

You should see the file listed at the end.

> **Do not use** `Fusion Admin Install.exe` for this guide.

---

# 7. Launch Steam from a terminal

For the initial install run, start Steam from a terminal so you can inspect its console output if the installer exits immediately.

Run:

~~~bash
unset MANGOHUD LD_PRELOAD WINEDEBUG
steam > ~/steam-fusion-console.log 2>&1
~~~

Leave this terminal open.

---

# 8. Add the Fusion installer to Steam as a non-Steam game

Inside Steam:

1. Click **Games**
2. Click **Add a Non-Steam Game to My Library**
3. Click **Browse**
4. Select:

~~~text
~/fusion-test/fusion-client-downloader.exe
~~~

5. Add it to your library

Then:

1. Right-click the new library entry
2. Open **Properties**
3. Open **Compatibility**
4. Enable **Force the use of a specific Steam Play compatibility tool**
5. Select **GE-Proton10-32**

---

# 9. Configure the Steam launch options

In the same Steam shortcut properties, set **Launch Options** to:

~~~text
PROTON_LOG=1 PROTON_DUMP_DEBUG_COMMANDS=1 WINEDLLOVERRIDES="mscoree,mshtml,webview2=disabled" DXVK_ASYNC=1 NO_AT_BRIDGE=1 %command% --quiet
~~~

What this does:

- `PROTON_LOG=1` enables Proton logging when possible
- `PROTON_DUMP_DEBUG_COMMANDS=1` helps debugging
- `WINEDLLOVERRIDES="mscoree,mshtml,webview2=disabled"` matches the known working recipe
- `DXVK_ASYNC=1` and `NO_AT_BRIDGE=1` match the working KDE/Wayland recipe
- `--quiet` uses Autodesk’s quiet installer mode

---

# 10. Run the installer

Click **Play** in Steam.

What to expect:

- a window may appear blank
- it may look unhelpful
- do **not** close it immediately
- let it sit for several minutes

If it flashes and exits immediately, check the Steam terminal log:

~~~bash
tail -n 200 ~/steam-fusion-console.log
~~~

---

# 11. Find the actual Steam prefix after installation

Because Fusion is being installed through Steam, the working prefix is **not** `~/.fusion360-proton2`.

It will live under Steam’s `compatdata` tree.

Find the installed Fusion executable:

~~~bash
find ~/.local/share/Steam/steamapps/compatdata ~/.steam/steam/steamapps/compatdata \
  -name Fusion360.exe 2>/dev/null
~~~

You should see something like:

~~~text
/home/youruser/.local/share/Steam/steamapps/compatdata/1234567890/pfx/drive_c/Program Files/Autodesk/webdeploy/production/<hash>/Fusion360.exe
~~~

Now save the prefix path:

~~~bash
FUSION_EXE="$(find ~/.local/share/Steam/steamapps/compatdata ~/.steam/steam/steamapps/compatdata -name Fusion360.exe 2>/dev/null | head -n1)"
PREFIX="${FUSION_EXE%%/pfx/*}"

mkdir -p ~/.config/fusion360-proton
printf '%s\n' "$PREFIX" > ~/.config/fusion360-proton/prefix
cat ~/.config/fusion360-proton/prefix
~~~

The final command should print the compatdata directory, for example:

~~~text
/home/youruser/.local/share/Steam/steamapps/compatdata/1234567890
~~~

> If `find` returns more than one result, use the one that matches the install you just performed and write that prefix path into `~/.config/fusion360-proton/prefix`.

---

# 12. Create the Fusion launcher script

Run:

~~~bash
mkdir -p ~/.local/bin ~/.local/share/applications

cat > ~/.local/bin/fusion360-launch <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if [ -d "$HOME/.steam/steam" ]; then
  STEAMROOT="$HOME/.steam/steam"
else
  STEAMROOT="$HOME/.local/share/Steam"
fi

PREFIX="$(cat "$HOME/.config/fusion360-proton/prefix")"

PROTON="$STEAMROOT/compatibilitytools.d/GE-Proton10-32/proton"
if [ ! -x "$PROTON" ]; then
  PROTON="$(find "$STEAMROOT/compatibilitytools.d" -maxdepth 2 -path '*/GE-Proton*/proton' | sort -V | tail -n1)"
fi

if [ -z "${PROTON:-}" ] || [ ! -x "$PROTON" ]; then
  echo "No GE-Proton binary found under $STEAMROOT/compatibilitytools.d" >&2
  exit 1
fi

FUSION_EXE="$(find "$PREFIX/pfx/drive_c/Program Files/Autodesk/webdeploy/production" -name Fusion360.exe -print -quit 2>/dev/null)"

if [ -z "${FUSION_EXE:-}" ]; then
  echo "Fusion360.exe not found under prefix: $PREFIX" >&2
  exit 1
fi

exec env PROTON_USE_WINED3D=0 DXVK_ASYNC=1 NO_AT_BRIDGE=1 \
  WINEDLLOVERRIDES="bcp47langs=" \
  STEAM_COMPAT_DATA_PATH="$PREFIX" \
  STEAM_COMPAT_CLIENT_INSTALL_PATH="$STEAMROOT" \
  "$PROTON" run "$FUSION_EXE" "${1:-}"
EOF

chmod +x ~/.local/bin/fusion360-launch
~~~

---

# 13. Create the Autodesk Identity Manager launcher

Run:

~~~bash
cat > ~/.local/bin/fusion360-idmgr <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

if [ -d "$HOME/.steam/steam" ]; then
  STEAMROOT="$HOME/.steam/steam"
else
  STEAMROOT="$HOME/.local/share/Steam"
fi

PREFIX="$(cat "$HOME/.config/fusion360-proton/prefix")"

PROTON="$STEAMROOT/compatibilitytools.d/GE-Proton10-32/proton"
if [ ! -x "$PROTON" ]; then
  PROTON="$(find "$STEAMROOT/compatibilitytools.d" -maxdepth 2 -path '*/GE-Proton*/proton' | sort -V | tail -n1)"
fi

if [ -z "${PROTON:-}" ] || [ ! -x "$PROTON" ]; then
  echo "No GE-Proton binary found under $STEAMROOT/compatibilitytools.d" >&2
  exit 1
fi

IDM_EXE="$(find "$PREFIX/pfx" -name AdskIdentityManager.exe -print -quit 2>/dev/null)"

if [ -z "${IDM_EXE:-}" ]; then
  echo "AdskIdentityManager.exe not found under prefix: $PREFIX" >&2
  exit 1
fi

exec env PROTON_USE_WINED3D=0 DXVK_ASYNC=1 NO_AT_BRIDGE=1 \
  WINEDLLOVERRIDES="bcp47langs=" \
  STEAM_COMPAT_DATA_PATH="$PREFIX" \
  STEAM_COMPAT_CLIENT_INSTALL_PATH="$STEAMROOT" \
  "$PROTON" run "$IDM_EXE" "${1:-}"
EOF

chmod +x ~/.local/bin/fusion360-idmgr
~~~

---

# 14. Register the Autodesk URI handlers

This is what makes browser sign-in return to Fusion.

Run:

~~~bash
cat > ~/.local/share/applications/adsk-fusion360-proton.desktop <<EOF
[Desktop Entry]
Name=Autodesk Fusion 360 Proton URI Handler
Exec=$HOME/.local/bin/fusion360-launch %u
Type=Application
MimeType=x-scheme-handler/adsk;
NoDisplay=true
EOF

cat > ~/.local/share/applications/adskidmgr-fusion360-proton.desktop <<EOF
[Desktop Entry]
Name=Autodesk Identity Manager Proton URI Handler
Exec=$HOME/.local/bin/fusion360-idmgr %u
Type=Application
MimeType=x-scheme-handler/adskidmgr;
NoDisplay=true
EOF

cat > ~/.local/share/applications/fusion360.desktop <<EOF
[Desktop Entry]
Name=Autodesk Fusion 360
Comment=Autodesk Fusion 360 via GE-Proton
Exec=$HOME/.local/bin/fusion360-launch
Terminal=false
Type=Application
Categories=Engineering;Graphics;
EOF

xdg-mime default adsk-fusion360-proton.desktop x-scheme-handler/adsk
xdg-mime default adskidmgr-fusion360-proton.desktop x-scheme-handler/adskidmgr
update-desktop-database ~/.local/share/applications
~~~

Verify registration:

~~~bash
xdg-mime query default x-scheme-handler/adsk
xdg-mime query default x-scheme-handler/adskidmgr
~~~

Expected output:

~~~text
adsk-fusion360-proton.desktop
adskidmgr-fusion360-proton.desktop
~~~

---

# 15. Launch Fusion for the first time

Run:

~~~bash
~/.local/bin/fusion360-launch
~~~

Click **Sign In** and complete the Autodesk login in your browser.

If `xdg-open` is sane and the URI handlers are registered, the callback should return to Fusion.

---

# 16. Fix broken or hard-to-read fonts

If Fusion launches but some UI text looks ugly, thin, jagged, or difficult to read, install additional Windows fonts and `gdiplus` into the **same existing Fusion Proton prefix**.

## 16.1 Locate the correct existing Fusion prefix and Proton install

Run:

~~~bash
FUSION_EXE="$(find ~/.local/share/Steam/steamapps/compatdata ~/.steam/steam/steamapps/compatdata -name Fusion360.exe 2>/dev/null | head -n1)"
PREFIX="${FUSION_EXE%%/pfx/*}"

if [ -d "$HOME/.steam/steam" ]; then
  STEAMROOT="$HOME/.steam/steam"
else
  STEAMROOT="$HOME/.local/share/Steam"
fi

PROTON="$STEAMROOT/compatibilitytools.d/GE-Proton10-32"
if [ ! -x "$PROTON/proton" ]; then
  PROTON="$(dirname "$(find "$STEAMROOT/compatibilitytools.d" -maxdepth 2 -path '*/GE-Proton*/proton' | sort -V | tail -n1)")"
fi

echo "Using prefix: $PREFIX"
echo "Using Proton: $PROTON"
~~~

You should see:

- a real Fusion compatdata path under Steam
- a real GE-Proton path

## 16.2 Install Winetricks if it is not already installed

Run:

~~~bash
sudo dnf install -y winetricks
~~~

## 16.3 Kill any leftover Wine / Proton processes for the Fusion prefix

If Fusion or a Wine helper was left open, `winetricks` may block waiting for the prefix to become idle.

Run:

~~~bash
env \
  WINE="$PROTON/files/bin/wine" \
  WINESERVER="$PROTON/files/bin/wineserver" \
  WINEPREFIX="$PREFIX/pfx" \
  "$PROTON/files/bin/wineserver" -k || true

sleep 2

pkill -f Fusion360.exe || true
pkill -f AdskIdentityManager.exe || true
pkill -f winetricks || true
pkill -f wineserver || true
sleep 2
~~~

If `winetricks` says it is waiting for `wineserver -w`, something from the Fusion prefix is still running.

## 16.4 Install the additional fonts and gdiplus into the existing Fusion prefix

Run:

~~~bash
env \
  WINE="$PROTON/files/bin/wine" \
  WINESERVER="$PROTON/files/bin/wineserver" \
  WINEPREFIX="$PREFIX/pfx" \
  winetricks -q allfonts gdiplus fontsmooth=rgb
~~~

This installs:

- the broader Winetricks font collection
- `gdiplus`

Using the Proton-bundled `wine` and `wineserver` avoids the Wine version mismatch problems that happen when plain system Wine binaries try to touch a Proton-managed prefix.

> This command can take a while.
>
> If it appears to pause, make sure it is not waiting on leftover Wine processes from the Fusion prefix.

## 16.5 Launch Fusion again

After the font install completes, launch Fusion normally:

~~~bash
~/.local/bin/fusion360-launch
~~~

At this point, the broken text should render correctly.

---

# 17. Apply the Wine graphics fix

This was the setting combination that solved the bad modal dialog z-order problem on Fedora 43 + Plasma Wayland.

Open `winecfg` for the Fusion prefix:

~~~bash
PREFIX="$(cat ~/.config/fusion360-proton/prefix)"

if [ -d "$HOME/.steam/steam" ]; then
  STEAMROOT="$HOME/.steam/steam"
else
  STEAMROOT="$HOME/.local/share/Steam"
fi

PROTON="$STEAMROOT/compatibilitytools.d/GE-Proton10-32/proton"
if [ ! -x "$PROTON" ]; then
  PROTON="$(find "$STEAMROOT/compatibilitytools.d" -maxdepth 2 -path '*/GE-Proton*/proton' | sort -V | tail -n1)"
fi

env STEAM_COMPAT_DATA_PATH="$PREFIX" \
    STEAM_COMPAT_CLIENT_INSTALL_PATH="$STEAMROOT" \
    "$PROTON" run winecfg
~~~

In `winecfg`, open the **Graphics** tab and set:

- **Allow the window manager to decorate the windows** = **checked**
- **Allow the window manager to control the windows** = **unchecked**
- **Emulate a virtual desktop** = **unchecked**

This was the final working combination.

# 18. Switch Fusion to OpenGL inside the app

Once Fusion is running:

1. Open **Preferences**
2. Open **General**
3. Find **Graphics Driver**
4. Change it to **OpenGL**
5. Restart Fusion

---

# 19. Daily-use launcher

After installation is complete, launch Fusion with:

~~~bash
~/.local/bin/fusion360-launch
~~~

or use the desktop menu entry created earlier.

Do **not** keep launching the original installer through Steam once Fusion is installed.

---

# 20. Quick validation checklist

Run these commands:

~~~bash
which xdg-open
xdg-mime query default x-scheme-handler/adsk
xdg-mime query default x-scheme-handler/adskidmgr
cat ~/.config/fusion360-proton/prefix
~~~

Expected results:

- `which xdg-open` returns `/usr/bin/xdg-open`
- `adsk` handler returns `adsk-fusion360-proton.desktop`
- `adskidmgr` handler returns `adskidmgr-fusion360-proton.desktop`
- the saved prefix points to the Steam `compatdata` directory containing Fusion

Then confirm in the UI:

- Fusion opens
- browser sign-in returns to Fusion
- save/open dialogs are clickable
- preferences opens
- graphics driver is OpenGL

---

# 21. Troubleshooting

## A. The installer flashes and exits immediately

Check all of the following:

1. You used **Fusion Client Downloader.exe**, not the admin installer
2. Steam is **native**, not Flatpak
3. The shortcut is forced to **GE-Proton10-32**
4. Launch options still include `--quiet`

Then inspect the terminal log:

~~~bash
tail -n 200 ~/steam-fusion-console.log
~~~

---

## B. Browser sign-in does not return to Fusion

Re-check the callback pieces:

~~~bash
which xdg-open
xdg-mime query default x-scheme-handler/adsk
xdg-mime query default x-scheme-handler/adskidmgr
~~~

Expected:

- `which xdg-open` -> `/usr/bin/xdg-open`
- `adsk` handler -> `adsk-fusion360-proton.desktop`
- `adskidmgr` handler -> `adskidmgr-fusion360-proton.desktop`

---

## C. Links inside Fusion do nothing when clicked

Check whether `xdg-open` has been overridden again:

~~~bash
which xdg-open
readlink -f "$(which xdg-open)"
~~~

If it points to `~/.local/bin/xdg-open`, rename that wrapper again and re-test.

---

## D. Modal dialog boxes turn gray and cannot be clicked

Re-open `winecfg` for the Fusion prefix and verify:

- **Allow the window manager to decorate the windows** = ON
- **Allow the window manager to control the windows** = OFF
- **Emulate a virtual desktop** = OFF

If a dialog still behaves badly, try pressing **Esc** first before killing Fusion.

---

## E. You accidentally created multiple Steam compatdata prefixes

Find all Fusion installs:

~~~bash
find ~/.local/share/Steam/steamapps/compatdata ~/.steam/steam/steamapps/compatdata \
  -name Fusion360.exe 2>/dev/null
~~~

Pick the correct one and rewrite the saved prefix:

~~~bash
printf '%s\n' "/full/path/to/the/correct/compatdata/number" > ~/.config/fusion360-proton/prefix
cat ~/.config/fusion360-proton/prefix
~~~

---

## F. `Fusion360.exe` cannot be found later

Check whether Autodesk changed the install layout within the same prefix:

~~~bash
PREFIX="$(cat ~/.config/fusion360-proton/prefix)"
find "$PREFIX/pfx/drive_c/Program Files/Autodesk" -name Fusion360.exe 2>/dev/null
~~~

If it exists elsewhere under the same prefix, update the launcher logic or reinstall using this guide.

---

# 22. Summary of the one setting that mattered most

If you only remember one technical fix from this entire process, remember this:

In `winecfg -> Graphics`, the setting that made Fusion usable on Fedora 43 + KDE Plasma Wayland was:

- **Allow the window manager to control the windows** = **OFF**

The full working set was:

- decorate = ON
- control = OFF
- virtual desktop = OFF

---

# 23. Final launch command

Once setup is complete, this is the normal launch command:

~~~bash
~/.local/bin/fusion360-launch
~~~

---

# 24. Optional cleanup

After Fusion is installed and working, you may want to remove the non-Steam installer shortcut from Steam to avoid launching the downloader again by accident.

You do **not** need the downloader entry anymore once Fusion is installed and your launcher script works.

---

# 25. Short version

If you need the shortest accurate summary of the whole process:

1. Install native Steam on Fedora 43
2. Install GE-Proton10-32 into `~/.steam/steam/compatibilitytools.d/`
3. Download `Fusion Client Downloader.exe`
4. Add it to Steam as a non-Steam game
5. Force `GE-Proton10-32`
6. Use launch options:

~~~text
PROTON_LOG=1 PROTON_DUMP_DEBUG_COMMANDS=1 WINEDLLOVERRIDES="mscoree,mshtml,webview2=disabled" DXVK_ASYNC=1 NO_AT_BRIDGE=1 %command% --quiet
~~~

7. Install Fusion
8. Save the correct Steam compatdata prefix
9. Create Fusion and Identity Manager launcher scripts
10. Register `adsk://` and `adskidmgr://` URI handlers
11. In `winecfg -> Graphics`, set:
    - decorate = ON
    - control = OFF
    - virtual desktop = OFF
12. In Fusion, set graphics driver to **OpenGL**

That is the working Fedora 43 + Plasma Wayland recipe.
