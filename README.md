# Hardware Refresh with USMT

Step-by-step procedure for migrating user data and settings from an old Windows device to a new one using the User State Migration Tool (USMT).

---

## Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Repository layout](#repository-layout)
4. [Step 1: Obtain USMT](#step-1-obtain-usmt)
5. [Step 2: Prepare the migration store](#step-2-prepare-the-migration-store)
6. [Step 3: Pre-migration checks](#step-3-pre-migration-checks)
7. [Step 4: Capture data from the old device](#step-4-capture-data-from-the-old-device-scanstate)
8. [Step 5: Prepare the new device](#step-5-prepare-the-new-device)
9. [Step 6: Restore data to the new device](#step-6-restore-data-to-the-new-device-loadstate)
10. [Step 7: Validate the migration](#step-7-validate-the-migration)
11. [Step 8: Clean up and decommission](#step-8-clean-up-and-decommission)
12. [Custom XML rules](#custom-xml-rules)
13. [Troubleshooting](#troubleshooting)
14. [Quick reference](#quick-reference)
15. [Refresh checklist](#refresh-checklist)

---

## Overview

USMT consists of two command-line tools:

| Tool | Runs on | Purpose |
|------|---------|---------|
| `ScanState.exe` | Old (source) device | Collects user profiles, files and settings into a migration store |
| `LoadState.exe` | New (destination) device | Restores the migration store to the new device |

Migration behaviour is controlled by XML rule files:

| File | Purpose |
|------|---------|
| `MigDocs.xml` | Finds user documents by file type and location |
| `MigApp.xml` | Migrates application settings |
| `Config.xml` | Optional. Generated with `/genconfig`; includes or excludes specific components |
| Custom `.xml` | Optional. Organisation-specific folders, registry keys or exclusions |

> **Note:** Do not use `MigUser.xml` together with `MigDocs.xml`. Use one or the other. `MigDocs.xml` is recommended.

---

## Prerequisites

- Local administrator rights on both devices. Run all commands from an **elevated** Command Prompt.
- A technician workstation with the Windows ADK installed (to obtain USMT).
- A migration store location with enough space (network share, external drive, or local hard-link store).
- The USMT version must be **the same as or newer than** the Windows version on the destination device.
- ScanState must use the architecture that matches the **source** OS (`amd64`, `x86` or `arm64`).
- Source and destination devices should be on a reliable network connection if a network share is used.
- Applications must be **installed on the new device before** LoadState runs. USMT migrates settings, not applications.
- For domain users, the new device must be **joined to the domain** before LoadState runs, and the domain must be reachable.

---

## Repository layout

```
.
├── README.md
├── usmt/
│   ├── amd64/            # Copied from the Windows ADK
│   ├── x86/
│   └── arm64/
├── xml/
│   ├── CustomRules.xml   # Organisation-specific rules
│   └── Config.xml        # Generated config (optional)
├── scripts/
│   ├── capture.cmd
│   └── restore.cmd
└── logs/                 # Collected logs per device (do not commit)
```

> **Tip:** Add `logs/` and `usmt/` to `.gitignore`. Do not commit USMT binaries or migration logs, as logs can contain usernames and file paths.

---

## Step 1: Obtain USMT

1. Download the **Windows ADK** matching your target Windows version from Microsoft.
2. Run the installer and select **only** *User State Migration Tool (USMT)*.
3. After installation, the files are located at:

   ```
   C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\User State Migration Tool\
   ```

4. Copy the folder matching the source architecture (for example `amd64`) to a deployment share or USB drive. This folder contains `ScanState.exe`, `LoadState.exe`, `MigDocs.xml` and `MigApp.xml`.

---

## Step 2: Prepare the migration store

### Option A: Network share (most common)

1. Create a share, for example `\\fileserver\MigStore$`.
2. Grant the technician account (or a technician group) **Full Control** on the share and NTFS.
3. Remove default access for standard users.
4. Plan for roughly the size of the data being migrated per device. Compression usually reduces this.

### Option B: External drive

Use an NTFS-formatted drive with enough free space. Keep the drive physically secure, as it will hold user data.

### Option C: Hard-link migration (in-place refresh only)

Used when the device is wiped and reinstalled in place. It is **not** suitable for moving to different hardware.

---

## Step 3: Pre-migration checks

On the **old** device:

1. Confirm the user is logged off, or that you are signed in as an administrator other than the profile being migrated.
2. Check free disk space on the source and the store location.
3. Run a disk space estimate (optional):

   ```cmd
   scanstate.exe \\fileserver\MigStore$\%COMPUTERNAME% /i:migdocs.xml /i:migapp.xml /nocompress /p:C:\Temp\spacecheck.xml /v:5
   ```

4. Review the profiles on the device and decide which should be migrated.
5. Note the application list. Make sure all required applications are available for the new device.
6. If **BitLocker** is enabled on the source, no action is needed for an online capture. Ensure you hold the recovery key for the device in case it is needed.
7. If **OneDrive Known Folder Move** is in use, confirm that files are fully synced and not online-only. USMT will not retrieve cloud-only files.
8. Close or disable applications that lock files, such as email clients and sync tools.

---

## Step 4: Capture data from the old device (ScanState)

1. Sign in to the old device with an administrator account.
2. Open **Command Prompt as administrator**.
3. Change to the USMT folder:

   ```cmd
   cd /d D:\usmt\amd64
   ```

4. Run ScanState.

### Standard capture (domain users)

```cmd
scanstate.exe \\fileserver\MigStore$\%COMPUTERNAME% ^
  /i:migdocs.xml /i:migapp.xml ^
  /o /c /vsc /v:13 ^
  /ue:*\* /ui:CONTOSO\* ^
  /uel:90 ^
  /l:%COMPUTERNAME%_scanstate.log ^
  /progress:%COMPUTERNAME%_scanstate_progress.log
```

### Common parameters

| Parameter | Meaning |
|-----------|---------|
| `/i:<file>` | XML rule file to include. Can be used multiple times |
| `/o` | Overwrite an existing store |
| `/c` | Continue on non-fatal errors and log them |
| `/vsc` | Use Volume Shadow Copy to capture locked files |
| `/v:13` | Maximum verbosity logging |
| `/ue:*\*` | Exclude all users... |
| `/ui:CONTOSO\*` | ...except users in the CONTOSO domain |
| `/uel:90` | Exclude profiles not used in the last 90 days |
| `/localonly` | Exclude data on removable drives and network shares |
| `/efs:copyraw` | Copy EFS-encrypted files in encrypted form |
| `/encrypt /key:"<key>"` | Encrypt the store. The same key is required for LoadState |
| `/nocompress` | Store uncompressed (faster, larger; useful for troubleshooting) |
| `/config:<file>` | Apply a Config.xml |

### Capture with an encrypted store

```cmd
scanstate.exe \\fileserver\MigStore$\%COMPUTERNAME% /i:migdocs.xml /i:migapp.xml /o /c /vsc /v:13 /encrypt /key:"YourStrongKeyHere"
```

> **Warning:** Do not hard-code keys in scripts committed to GitHub. Use `/keyfile` with a protected file, or pass the key at run time.

### Check the result

1. Review the exit code:

   ```cmd
   echo %ERRORLEVEL%
   ```

   `0` indicates success. Any other value must be investigated (see [Troubleshooting](#troubleshooting)).

2. Open the log and search for `error` and `warning`.
3. Confirm the store folder exists and contains `USMT\USMT.MIG`.
4. Copy the log to the `logs/` folder or central location.

---

## Step 5: Prepare the new device

1. Image or deploy Windows to the new device.
2. Apply drivers and Windows updates.
3. Join the device to the domain.
4. Install **all required applications**, in the same or newer versions than on the old device. LoadState only restores settings for applications that are present.
5. Confirm the network connection and access to the migration store.
6. Sign in with a local administrator account (not one of the users being migrated).
7. Copy the USMT folder to the device or run it from the deployment share.

> **Important:** Do not sign in as the end user before LoadState. Doing so creates a new profile, which can result in a profile folder such as `username.CONTOSO` and mapping issues.

---

## Step 6: Restore data to the new device (LoadState)

1. Open **Command Prompt as administrator**.
2. Change to the USMT folder:

   ```cmd
   cd /d D:\usmt\amd64
   ```

3. Run LoadState.

### Standard restore (domain users)

```cmd
loadstate.exe \\fileserver\MigStore$\OLDPC01 ^
  /i:migdocs.xml /i:migapp.xml ^
  /c /v:13 ^
  /l:%COMPUTERNAME%_loadstate.log ^
  /progress:%COMPUTERNAME%_loadstate_progress.log
```

Use the **same XML files** as were used with ScanState.

### Encrypted store

```cmd
loadstate.exe \\fileserver\MigStore$\OLDPC01 /i:migdocs.xml /i:migapp.xml /c /v:13 /decrypt /key:"YourStrongKeyHere"
```

### Local user accounts

Local accounts do not exist on the new device by default. Create and enable them during restore:

```cmd
loadstate.exe \\fileserver\MigStore$\OLDPC01 /i:migdocs.xml /i:migapp.xml /c /v:13 /lac:"TempP@ssw0rd" /lae
```

| Parameter | Meaning |
|-----------|---------|
| `/lac` | Create local accounts that do not exist. Disabled unless `/lae` is also used. Optionally takes a password |
| `/lae` | Enable the created accounts |

### Mapping users (if usernames or domains change)

```cmd
loadstate.exe \\fileserver\MigStore$\OLDPC01 /i:migdocs.xml /i:migapp.xml /c /v:13 /mu:OLDDOMAIN\jsmith:NEWDOMAIN\jsmith
```

### Check the result

1. Check `%ERRORLEVEL%` is `0`.
2. Review the LoadState log for errors and warnings.
3. Restart the device.

---

## Step 7: Validate the migration

1. Sign in as the migrated user.
2. Confirm that the following are present:
   - Desktop, Documents, Downloads, Pictures and other known folders
   - Browser favourites and settings
   - Outlook profile and signatures (Outlook may need to reconfigure)
   - Application settings for the key business applications
   - Mapped network drives and printers (these may need to be re-added)
   - Wi-Fi profiles (if migrated or deployed by policy)
3. Ask the user to confirm that their files and applications work as expected.
4. Record the migration outcome in the ticket or refresh tracker.

---

## Step 8: Clean up and decommission

1. Keep the migration store for an agreed retention period (for example 14 to 30 days) in case anything is missing.
2. After the user has signed off, delete the store from the share.
3. Securely wipe the old device according to your organisation's data sanitisation policy before disposal, reuse or return.
4. Remove the old device from Active Directory, Intune, SCCM/MECM and any asset management records.
5. Archive logs centrally and remove them from technician devices.

---

## Custom XML rules

Use a custom XML file to migrate folders that the default rules do not cover.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<migration urlid="http://www.microsoft.com/migration/1.0/migxmlext/customrules">
  <component type="Documents" context="System">
    <displayName>Custom data folders</displayName>
    <role role="Data">
      <rules>
        <include>
          <objectSet>
            <pattern type="File">C:\Data\* [*]</pattern>
          </objectSet>
        </include>
        <exclude>
          <objectSet>
            <pattern type="File">C:\Data\Temp\* [*]</pattern>
          </objectSet>
        </exclude>
      </rules>
    </role>
  </component>
</migration>
```

Use it with both tools:

```cmd
scanstate.exe <store> /i:migdocs.xml /i:migapp.xml /i:CustomRules.xml ...
loadstate.exe <store> /i:migdocs.xml /i:migapp.xml /i:CustomRules.xml ...
```

### Generating a Config.xml

```cmd
scanstate.exe /genconfig:Config.xml /i:migdocs.xml /i:migapp.xml
```

Edit the file and set components to `migrate="no"` to exclude them, then pass it with `/config:Config.xml` to both ScanState and LoadState.

---

## Troubleshooting

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| Exit code `27` or `28` | Invalid command line or store problem | Check the syntax and the store path and permissions |
| Exit code `26` | Store path error | Confirm the path exists and is accessible from the device |
| Exit code `71` | Store is corrupt or incomplete | Re-run ScanState with `/o` |
| Exit code `3` or `11` | Not run as administrator, or the store is invalid | Open an elevated prompt |
| Access denied on the share | Share or NTFS permissions | Grant the technician account Full Control |
| Locked files in the log | Files in use | Use `/vsc` and close applications |
| `/vsc` fails | Shadow copy not available | Run without `/vsc` and close applications |
| Profile restored to `user.DOMAIN` | User signed in before LoadState | Remove the new profile, or use `/mu` and re-run |
| Users not migrated | `/ue` and `/ui` filters too strict | Review the filter parameters |
| Application settings missing | Application not installed or a different version | Install it before LoadState |
| Cloud-only OneDrive files missing | Files not downloaded locally | Set the folders to "Always keep on this device" before capture |
| Store larger than expected | Large media or temp files | Use `/uel`, `/localonly` or exclusions in XML |
| Local user cannot sign in | Account created but disabled | Use `/lae`, or enable the account manually |

> **Note:** Exit codes vary by USMT version. Always check the log first, then the Microsoft documentation for your version.

### Useful options for diagnosis

```cmd
/v:13                  :: maximum log detail
/listfiles:files.txt   :: list the files that will be collected (ScanState)
/nocompress            :: uncompressed store, can be browsed
```

---

## Quick reference

```cmd
:: CAPTURE (old device)
scanstate.exe \\server\share\%COMPUTERNAME% /i:migdocs.xml /i:migapp.xml /o /c /vsc /v:13 /ue:*\* /ui:DOMAIN\* /uel:90 /l:scanstate.log

:: RESTORE (new device)
loadstate.exe \\server\share\OLDPC /i:migdocs.xml /i:migapp.xml /c /v:13 /l:loadstate.log

:: GENERATE CONFIG
scanstate.exe /genconfig:Config.xml /i:migdocs.xml /i:migapp.xml
```

---

## Refresh checklist

**Before**
- [ ] Ticket raised and user informed of the date and downtime
- [ ] Old and new device serial numbers recorded
- [ ] Application list reviewed
- [ ] OneDrive sync confirmed
- [ ] Store location ready and permissions checked
- [ ] Free disk space confirmed on source and store

**Capture**
- [ ] ScanState run as administrator
- [ ] Exit code `0`
- [ ] Log reviewed and archived
- [ ] Store verified

**Build**
- [ ] Windows deployed and updated
- [ ] Device joined to the domain
- [ ] Applications installed
- [ ] User has **not** signed in

**Restore**
- [ ] LoadState run as administrator
- [ ] Exit code `0`
- [ ] Log reviewed and archived
- [ ] Device restarted

**After**
- [ ] User sign-in tested
- [ ] Data and applications confirmed by the user
- [ ] Store deleted after the retention period
- [ ] Old device wiped and removed from all systems

---

## References

- [User State Migration Tool (USMT) overview, Microsoft Learn](https://learn.microsoft.com/en-us/windows/deployment/usmt/usmt-overview)
- [ScanState syntax, Microsoft Learn](https://learn.microsoft.com/en-us/windows/deployment/usmt/usmt-scanstate-syntax)
- [LoadState syntax, Microsoft Learn](https://learn.microsoft.com/en-us/windows/deployment/usmt/usmt-loadstate-syntax)
- [USMT exit codes, Microsoft Learn](https://learn.microsoft.com/en-us/windows/deployment/usmt/usmt-return-codes)

## Licence

Add a licence of your choice (for example MIT) in a `LICENSE` file.
