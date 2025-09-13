# 📦 deb-to-rpm-via-fpm

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![Method: FPM](https://img.shields.io/badge/Method-FPM-informational)
![Target: Fedora RPM](https://img.shields.io/badge/Target-Fedora%20RPM-blue)

A clean, general method for converting Debian `.deb` packages into RPMs on Fedora using **FPM (Effing Package Management)**.  
Includes a real-world case study: installing **CodeTantra Secure Exam App (SEA)** on Fedora.

---

## 📌 Why This Repo?

Fedora users often face the same roadblock:  
> *“This app only ships a `.deb` package — how do I install it?”*

- Tools like **alien** often fail (broken spec files, wrong dependency names).  
- Debian and Fedora use **different package naming conventions** (`libgtk-3-0` vs `gtk3`).  
- The web is full of partial fixes, but no clean, repeatable method.

This repo documents a **professional, repeatable recipe** using FPM that works broadly, not just for CodeTantra.

---

## ⚠️ Disclaimer

- Works best for apps that **bundle their runtime** (Electron, Java, single binaries).  
- Apps needing **Debian-specific drivers/kernel modules/init scripts** may still fail.  
- You may need to resolve **runtime libraries** manually (`ldd` + `dnf provides`).  
- `--nogpgcheck` is used here because the RPM is locally built and unsigned. In production, **sign your RPM** or install from a trusted, signed repository.  
- This is **not official Fedora or vendor support**.

---

## 📚 Table of Contents

1. [📌 Why This Repo?](#-why-this-repo)  
2. [⚠️ Disclaimer](#%EF%B8%8F-disclaimer)  
3. [🛠 Prerequisites](#-prerequisites)  
4. [🔄 Conversion Steps](#-conversion-steps)  
5. [📥 Install the RPM](#-install-the-rpm)  
6. [📦 Runtime Dependencies](#-runtime-dependencies)  
7. [🐛 Troubleshooting](#-troubleshooting)  
8. [🖥 Case Study: CodeTantra SEA](#-case-study-codetantra-sea)  
9. [❌ Why Not Alien?](#-why-not-alien)  
10. [🗑 Uninstall](#-uninstall)  
11. [📜 License](#-license)  

---

## 🛠 Prerequisites

Install FPM and build tools:

```bash
sudo dnf install -y ruby ruby-devel gcc make rpm-build rpmdevtools
sudo gem install --no-document fpm
```

---

## 🔄 Conversion Steps

Convert `.deb` → `.rpm` (ignore Debian dependencies)

```bash
fpm -s deb -t rpm --no-depends yourapp_x.y.z_amd64.deb
```

Flags explained:
- `-s deb` → source format = Debian package
- `-t rpm` → target format = RPM package
- `--no-depends` → drop Debian dependency metadata (avoids mismatched package names on Fedora)

Resulting file will look like: `yourapp-x.y.z-1.x86_64.rpm`.

---

## 📥 Install the RPM

First change into the directory where the original file is downloaded, with `cd`:
```bash
cd your file directory
```

Then install with:
```bash
sudo dnf install -y --nogpgcheck ./yourapp-x.y.z-1.x86_64.rpm
```

> **Note:** `--nogpgcheck` is acceptable for local, self-built RPMs.
For production or team use, sign your RPM with GPG or host it in a signed repository.

---

## 📦 Runtime Dependencies

Most apps need some system libraries (GTK, NSS, etc.). On Fedora, install them with:

```bash
sudo dnf install -y \
  gtk3 libXScrnSaver libXtst nss libnotify \
  at-spi2-core libsecret libuuid xdg-utils
```

> If your binary path contains spaces (common for Electron apps under `/opt`), **quote** it when running:
  `"/opt/My App/app-binary"`

---

## 🐛 Troubleshooting

List installed files for a package:

```bash
rpm -ql yourapp
```

Inspect the contents of an RPM *before* installing:

```bash
rpm -qlp ./yourapp-x.y.z-1.x86_64.rpm
```

Check what the RPM claims it requires:

```bash
rpm -q --requires yourapp
```

Check missing runtime libs:

```bash
ldd /path/to/yourapp | grep 'not found'
```

Find and install missing libs:

```bash
sudo dnf provides '*/libNAME.so*'
sudo dnf install -y <matching-package>
```

**If a GUI launcher didn’t appear:**
Check for a `.desktop` file:

```bash
rpm -ql yourapp | grep '\.desktop$'
```

If none, create one manually in `~/.local/share/applications/`.

---

## 🖥 Case Study: CodeTantra SEA

I needed CodeTantra Secure Exam App on Fedora.
The vendor only provided `codetantra-sea_3.1.0_amd64.deb`.
- **Alien attempt:** Failed with
  
```bash
Empty tag: Summary
nothing provides libgtk-3-0 needed by codetantra-sea
```

- **FPM method (worked):**

```bash
fpm -s deb -t rpm --no-depends codetantra-sea_3.1.0_amd64.deb
sudo dnf install -y --nogpgcheck ./codetantra-sea-3.1.0-1.x86_64.rpm
```

- **Result:**
    - Installed to `/opt/CodeTantra SEA/`
    - Binary at `/opt/CodeTantra SEA/codetantra-sea`
    - Launcher auto-created in GNOME
✅ App runs perfectly on Fedora.

---

## ❌ Why Not Alien?

- Generates broken RPM spec files.
- Copies Debian dependency names (`libgtk-3-0`) that don’t exist in Fedora (`gtk3`).
- Often fails with errors like *"Empty tag: Summary"*.

  **FPM** is simpler:
  - It just repackages the payload.
  - Lets you control metadata (`--no-depends`, `--depends`).
  - Works across multiple packaging formats.

---

## 🗑 Uninstall

Remove the installed RPM cleanly:

```bash
sudo dnf remove -y yourapp
```

---

## 📜 License

This project is licensed under the [MIT License](LICENSE).
If you reuse this guide, please credit the author.
