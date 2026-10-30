# Brother MFC-J4540DW network printer & scanner on openSUSE Tumbleweed

Install notes for the **MFC-J4540DW** on **openSUSE Tumbleweed** using Brother's `linux-brprinter-installer`: printer driver (CUPS) + scanner (brscan5 / brscan-skey) over the network.

`#Brother` `#Drucker` `#OpenSUSE` `#Tumbleweed`

## Contents

1. [Download & unpack the installer](#1-download--unpack-the-installer)
2. [Run the installer](#2-run-the-installer)
3. [Confirm the packages](#3-confirm-the-packages)
4. [Set the Device URI (network IP)](#4-set-the-device-uri-network-ip)
5. [Test print](#5-test-print)
6. [Scanner drivers](#6-scanner-drivers)
7. [Verify](#7-verify)
8. [Notes & troubleshooting](#8-notes--troubleshooting)

---

## 1. Download & unpack the installer

From the Brother support page for your model (here: **MFC-J4540DW**), download `linux-brprinter-installer-2.2.6-0.gz` (Linux → "Driver Install Tool"), then unpack it:

```bash
cd ~/Downloads
gunzip linux-brprinter-installer-2.2.6-0.gz
```

Background reading: [openSUSE SDB: Brother – How to set up a networked printer/scanner](https://en.opensuse.org/SDB:Brother_How_to_setup_a_Networked_Printer_Scanner).

## 2. Run the installer

Pass the model name as the argument:

```bash
sudo bash linux-brprinter-installer-2.2.6-0 mfc-j4540dw
```

## 3. Confirm the packages

The installer lists what it will install — answer `y`:

```text
You are going to install following packages.
  mfcj4540dwpdrv-3.5.0-1.i386.rpm
  brscan5-1.x.x-x.x86_64.rpm
  brscan-skey-0.3.x-x.x86_64.rpm
OK? [y/N] ->y
```

It then installs the printer driver with `rpm --nodeps --replacefiles --ignorearch`, adds a CUPS queue `MFCJ4540DW`, relaxes AppArmor for cupsd and sets SELinux labels for `/opt/brother`.

<details>
<summary>Expected harmless messages</summary>

```text
ln: failed to create symbolic link '/etc/init.d/cupsys': No such file or directory
ln: failed to create symbolic link '/etc/init.d/cups': No such file or directory
ln: failed to create symbolic link '/etc/init.d/lpd': No such file or directory
ln: failed to create symbolic link '/etc/init.d/lprng': No such file or directory
lpadmin: Printer drivers are deprecated and will stop working in a future version of CUPS.
Profile for /usr/sbin/cupsd not found, skipping
```

Tumbleweed uses systemd, so there is no `/etc/init.d` — the `ln` errors can be ignored. The AppArmor line just means no cupsd profile is loaded (new Tumbleweed installs use SELinux by default), so the installer's SELinux labelling is what matters. The CUPS deprecation warning is about PPD drivers in general and doesn't stop the install.

</details>

## 4. Set the Device URI (network IP)

Answer `Y`, then choose **9 (I): Specify IP address** and enter the printer's IP:

```text
Will you specify the Device URI? [Y/n] ->Y

0: beh
1: cups-brf:/
2: smb
3: http
4: lpd
5: ipp
6: socket
7: ipps
8: https
9 (I): Specify IP address.
10 (A): Auto. (usb://dev/usblp0)

select the number of destination Device URI. ->9
enter IP address ->192.168.178.58

lpadmin -p MFCJ4540DW -v lpd://192.168.178.58/BINARY_P1 -E
```

> [!TIP]
> Give the printer a fixed IP (DHCP reservation in your router, e.g. FRITZ!Box → Heimnetz → Netzwerk) so the queue doesn't break when the address changes.

## 5. Test print

```text
Test Print? [y/N] ->y
wait 5s.
lpr -P MFCJ4540DW /usr/share/cups/data/testprint
```

## 6. Scanner drivers

The MFC-J4540DW uses the newer **brscan5** scanner driver (not brscan4). The installer downloads `brscan5` and `brscan-skey` from `download.brother.com`. For each one it shows the Brother License Agreement — accept with `Y`:

```text
Do you agree? [Y/n] ->Y
wget ... brscan5-1.x.x-x.x86_64.rpm      → installed

Do you agree? [Y/n] ->Y
wget ... brscan-skey-0.3.x-x.x86_64.rpm  → installed

brsaneconfig5 -a name=MFC-J4540DW model=MFC-J4540DW ip=192.168.178.58
```

The last line registers the network scanner with SANE.

## 7. Verify

| Check | Command |
|---|---|
| Printer queue exists | `lpstat -v` |
| Print a file | `lpr -P MFCJ4540DW file.pdf` |
| Scanner registered | `brsaneconfig5 -q \| grep MFC` |
| SANE sees the scanner | `scanimage -L` |
| Scan a page | `scanimage --format=png -o scan.png` (or Skanlite / Simple Scan) |
| Scan-button daemon | `brscan-skey` then `brscan-skey -l` |

## 8. Notes & troubleshooting

> [!NOTE]
> Package versions (`x.x`) change over time — the installer always fetches the current ones from Brother. Older Brother models use separate `…lpr` + `…cupswrapper` RPMs and `brscan4`/`brsaneconfig4`; the J4540DW uses the combined `…pdrv` package and `brscan5`/`brsaneconfig5`.

- **Scanner not found:** check the IP with `brsaneconfig5 -q`; remove/re-add with `brsaneconfig5 -r MFC-J4540DW` and the `-a` line above.
- **Firewall:** network scanning may need UDP 54925 (scan-to-PC button) open in `firewalld`.
- **Driverless alternative:** most current Brother MFCs support IPP Everywhere / AirPrint and eSCL —
  ```bash
  lpadmin -p MFCJ4540DW-ipp -E -v ipp://192.168.178.58/ipp/print -m everywhere
  ```
  and the `sane-airscan` package often work without Brother's RPMs.

---

<sub>Personal setup notes · Brother MFC-J4540DW · openSUSE Tumbleweed</sub>
