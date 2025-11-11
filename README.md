# How-to-install-exe-files-on-android-using-termux-
In this repository you can learn how to install exe files like application and games which are made for windows but installing them on android 
Quick summary (what this does)

You’ll create a Linux environment in Termux, install Box64/Box86 (x86 → ARM translators), install Wine (Windows compatibility layer), and run .exe via an X server (Termux-X11) so windows appear on-screen.


---

Prerequisites (what you need)

Android device (64-bit ARM recommended).

Termux (official) installed.

Enough free storage (20–30 GB recommended if you plan to install full Windows ISOs).

Good battery/charging while doing heavy work.

Patience — building from source is CPU + RAM heavy.



---

0 — Decide route

Recommended (fastest & simplest): Use prebuilt Box64+Wine bundles (search ptitSeb/box64 releases or box64 wine bundle on GitHub) and extract them inside Debian proot.

Fallback / if you want to compile: clone box64 and box86 and make them in Debian. (Slow on phone but doable.)



---

1 — Prepare Termux & storage

# update Termux packages
pkg update -y && pkg upgrade -y

# install tools
pkg install proot-distro wget tar unzip git -y

# grant storage access (runs an Android prompt)
termux-setup-storage

Verify the Download dir:

ls /sdcard/Download


---

2 — Install Debian (proot) and enter it

proot-distro install debian
proot-distro login debian
# now you should see root@localhost:~#

Inside Debian:

apt update && apt upgrade -y
apt install wget curl git build-essential cmake python3 -y


---

3 — (Recommended) Get prebuilt Box64 + Wine bundle

Why: avoids heavy compile and common build errors.
How: open the project release page in your phone browser (GitHub) and download a box64 + wine bundle that matches ARM64. If you prefer CLI, do it from Debian once you have a correct URL.

Example steps (replace URL with the real release tar.xz you downloaded from GitHub):

# from Debian shell
cd ~
wget -O box64-wine.tar.xz "URL_OF_PREBUILT_BUNDLE"   # (use the correct URL you verified)
tar -xf box64-wine.tar.xz -C ~/
# assume it extracts to ~/box64-wine
ls ~/box64-wine
# test
~/box64-wine/box64 ~/box64-wine/wine64 --version

If it prints a Wine version, great — you have working binaries.


---

4 — (If no prebuilt available) Build Box64 & Box86 (fallback)

Warning: heavy and long on phones.

# box64
git clone https://github.com/ptitSeb/box64.git
cd box64
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
make install
cd ~

# box86 (if you need 32-bit)
git clone https://github.com/ptitSeb/box86.git
cd box86
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
make install
cd ~

After builds:

box64 --version
box86 --version   # if installed


---

5 — Install / place Wine (64-bit recommended)

Option A — prebuilt Wine in the bundle: use the wine64 shipped in the bundle:

# sample: run wine from the bundle using box64
~/box64-wine/box64 ~/box64-wine/wine64 --version

Option B — download Wine build (if you have a proper ARM/Box64 build): download and extract into ~/wine and use box64 ~/wine/bin/wine64.

Option C — do NOT try to apt install wine blindly inside proot if it pulls x86 packages that your system cannot handle. If apt tries to install wine32:i386 and breaks, you’ll need to fix dpkg (see troubleshooting below).


---

6 — Configure X display (Termux-X11)

On the Android side install Termux-X11 (from its GitHub releases). Then inside Termux start it:

# in Termux (not in Debian proot)
termux-x11 :0 &    # starts server; open the Termux-X11 app and click Connect if needed
export DISPLAY=:0  # in the shell where you'll run QEMU/Wine

If you’re inside the Debian proot, also export:

export DISPLAY=:0

Test with:

# install x11 apps if needed
apt install x11-apps -y    # provides xclock, xeyes (optional)
xclock

If a window pops in Termux-X11 → display works.


---

7 — Run a test EXE (Notepad example)

Place your Notepad.exe or any small exe in /sdcard/Download/.

From Debian:

# if wine is in bundle and box64 is at ~/box64-wine/box64
~/box64-wine/box64 ~/box64-wine/wine64 /sdcard/Download/Notepad.exe

If wine is system binary (rare) you can:

box64 wine64 /sdcard/Download/Notepad.exe

You should see the Notepad window open via Termux-X11.


---

8 — Running larger installers & Windows ISOs

Small EXE apps: run directly with box64 wine64 app.exe.

Full Windows (setup ISO): use QEMU (heavy) — qemu-system-x86_64 -m 4096 -cdrom /sdcard/Download/Win.iso -hda /sdcard/Download/win.img -vga std -usbdevice tablet
(QEMU is separate; you run a full VM, not Wine.)



---

Troubleshooting — errors you likely saw & fixes

Illegal instruction

Means binary was built for a CPU instruction set your SoC doesn’t support (e.g., x86 binary or wrong ARM target).

Fix: use Box64+Wine bundle that contains ARM translations or build box64/box86 locally and run box64 /path/to/wine64 ….


dpkg errors & corrupt .deb (“unexpected end of file or stream”)

If a package download got corrupted:

rm -f /var/cache/apt/archives/libwine_*.deb
apt clean
dpkg --configure -a
apt update
apt --fix-broken install -y

If apt continues to try to install wine32:i386 and fails, either add i386 architecture only if you have box86 and i386 support, or avoid apt wine and use prebuilt bundle.

404 when wget from GitHub

GitHub release filenames change. Open the project releases page in a browser, find the correct asset, long-press → copy link address, then wget "URL". Do not save HTML pages.


X server says --shm-helper or GTK pixbuf errors

Harmless warnings: GUI might look plain. Install adwaita-icon-theme/librsvg2 in Debian if available — but Termux repos may not have them. Ignore if apps run.


No termux-setup-storage (after you wiped environment)

Reinstall Termux core utilities:


pkg install termux-tools -y

Then run termux-setup-storage again.


---

Performance & safety tips

Use -smp N and -m flags sensibly: -smp 2-4 -m 2048 for phones. Too many cores or memory can crash/overheat phone.

Keep phone plugged in. Disable aggressive battery/CPU limits.

Don’t run heavy games directly — many won’t be playable. Lightweight GUI apps and small games may work.

Windows license: installing Microsoft Windows via ISO requires proper license—respect EULAs.

Backup important files before running large QEMU images.



---

Quick commands cheat-sheet (copy/paste)

# Termux (host)
pkg update -y
pkg install proot-distro wget tar git termux-x11 -y
termux-setup-storage
termux-x11 :0 &

# Debian (proot)
proot-distro install debian
proot-distro login debian
apt update && apt upgrade -y
apt install wget curl git build-essential cmake -y

# If you have a prebuilt bundle file (downloaded via browser to /sdcard/Download)
cp /sdcard/Download/box64-wine.tar.xz ~/
tar -xf ~/box64-wine.tar.xz -C ~/
# Example run:
~/box64-wine/box64 ~/box64-wine/wine64 /sdcard/Download/Notepad.exe
