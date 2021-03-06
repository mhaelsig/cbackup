# cbackup

cbackup is a simple backup/restore script for rooted Android devices. It backs up apps, full app data, and some Android metadata.

## Quickstart

Run the following command in Termux to take a backup:

```bash
bash -c "$(curl -L https://git.io/cbackup-quick)"
```

You can also specify a destination folder as an argument, which will be cleared if it already exists and created otherwise:

```bash
bash -c "$(curl -L https://git.io/cbackup-quick)" backup /data/local/tmp/cbackup
```

To restore a backup located at `/sdcard/cbackup`:

```bash
bash -c "$(curl -L https://git.io/cbackup-quick)" restore
```

Or restore a backup from a custom location:

```bash
bash -c "$(curl -L https://git.io/cbackup-quick)" restore /data/local/tmp/cbackup
```

Using a custom path outside of `/sdcard`, e.g. `/data/local/tmp/cbackup`, will work significantly faster on Android 11 and newer.

If you're worried about piping and running a random script, the [quickstart script](https://github.com/kdrag0n/cbackup/blob/master/termux-quickstart.sh) is short and thoroughly commented, so feel free to download it and read it yourself before running it.

## Scope

cbackup includes the following data for each app:

- App packages (APKs, including split APKs from Android App Bundles)
- App private data (excluding cache and data explicitly marked as no-backup)
- Per-app Android device IDs (SSAIDs; only restored if you reboot immediately after restoring)
- Granted dangerous runtime permissions
- Whether battery optimization is enabled
- Name of the installer app that installed the app in question, e.g. Play Store

cbackup **does not** include:

- Account information registered with Android's AccountManager service
- App external data stored in `/sdcard/Android/data`
- OBBs stored in `/sdcard/Android/obb`, e.g. game resources
- Device-bound keystore encryption keys (not possible to extract by design)

Some security-centric apps that use device-bound keystore keys to encrypt their data are included in a builtin blacklist, so their data will not be backed up.

Only apps installed in the primary user profile (Android user ID 0) will be backed up. Work profiles and other users will be excluded.

## Usage

If you run the script with no arguments, it will default to creating a new backup located at `backup_dir` its settings, which is set to `/sdcard/cbackup` by default.

cbackup accepts the following optional positional arguments:

- First argument: mode — `backup` or `restore`
- Second argument: backup path

Example to restore from a backup located at `/data/local/tmp/cbackup`:

```bash
./cbackup.sh restore /data/local/tmp/cbackup
```

And to back up to `/data/local/tmp/cbackup`:

```bash
./cbackup.sh backup /data/local/tmp/cbackup
```

## Security

Encryption is mandatory.

App data archives are encrypted with AES-256-CTR using the OpenSSL command-line tool. The encryption key is derived from the provided password with 200,001 iterations of PBKDF2-HMAC-SHA256, with a random salt for each encrypted file. Because the salt fed into PBKDF2 is random and not tracked across different runs of the OpenSSL tool, the encryption key and IV change for every file.

Each app also contains a password canary, which is a file encrypted with the same password and parameters that just contains a constant string in it to verify whther the given password is correct.

App packages and names are not encrypted, so anyone can see which apps you have backed up. It is also possible for an adversary to tamper with the app packages and inject spyware or other data exfiltration code into them, as the signatures are not verified to match the ones that were used when the app was initially backed up.

AES-256-CTR does not have error propagation or any form of MAC, so an adversary can manipulate the data archives to corrupt them. Future versions of cbackup will use an AEAD such as AES-256-GCM or ChaCha20-Poly1305 to prevent this from happening.

None of the supplementary metadata such as SSAIDs, battery optimization status, granted permissions, or installer names are encrypted or verified, so adversaries can also tamper with those and grant more permissions than desired.

All of the security loopholes are planned to be addressed in later versions of cbackup.

Summary:

- AES-256-CTR encryption
- PBKDF2-HMAC-SHA256 key derivation
- No AEAD or integrity verification
- Only data is encrypted

## Storage format

The preliminary v0 storage format is currently used by the cbackup shell script.

### v0 (preliminary)

All data related to each app is stored in a folder with the app's package name. The following files will always be present:

- `backup_version.txt`: plain-text file containing `0\n`
- `password_canary.enc`: encrypted file containing `cbackup-valid`
- `base.apk`: unencrypted base APK file

The following files may or may not be present, depending on what data is associated with the app that is being backed up:

- `split_config.*.apk`: unencrypted additional split APKs with resources
- `data.tar.zst.enc`: encrypted tarball archive containing app data, compressed with Zstandard before encrypting
- `permissions.list`: unencrypted list of granted runtime permissions
- `ssaid.xml`: system-generated SSAID entry from the SSAID settings XML
- `battery_opt_disabled`: empty file, the presence of this file indicates that battery optimization is disabled
- `installer_name.txt`: unencrypted plain-text file containing the installer app's package name

All encrypted files are encrypted using the OpenSSL CLI tool with the AES-256-CTR block cipher and 32-byte key derived from the user's password using 200,001 iterations of PBKDF2 and a random salt stored in the file's header.

App data includes both main CE-encrypted data as well as DE-encrypted (Direct Boot) data. App, code, and shader caches are always excluded, as well as files placed by apps in the dedicated no-backup folder.
