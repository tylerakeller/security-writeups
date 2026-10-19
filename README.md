# Securing a personal Android fleet: two audits and a signed update chain

*Tyler Keller · October 2026*

I write about twenty Android apps for my own phones and tablets and ship them through an update server I run at home. One of them is a password vault. In 2026 I ran two structured security audits across this code and rebuilt how updates are signed and checked before they install. This write-up covers what we found, what I fixed, and what is still open.

## The setup

- **Apps:** Kotlin apps for my own devices, plus a few Python and C++ tools on Windows. Nothing is on Google Play. Every app is sideloaded.
- **Update server:** a small FastAPI service that builds, signs and publishes APKs and serves a manifest per app. It listens only on loopback and is reachable only over my private Tailscale network, through a TLS proxy, with a bearer token.
- **On-device updater:** each app can update itself, and a separate guardian app installs, updates and rolls back the others. The guardian also keeps known-good copies offline.

The threats I care about are a lost or stolen phone, secrets leaking through screenshots or the clipboard, a compromised update server or network path pushing a malicious APK, and my own mistakes destroying data.

## How the audits ran

Both audits ran as Claude Code multi-agent workflows. Independent finder agents reviewed each area in parallel, and a second pass tried to disprove every finding before it counted. In the fleet-wide audit, 92 findings went in and 91 survived verification. I decided what to fix. Each fix was built, installed on my devices and committed.

## Part 1: The password-vault audit (34 findings)

The vault stores credentials encrypted on the phone and types them into a paired computer as a Bluetooth keyboard, so passwords never pass through a network or the computer's clipboard. That makes the phone-side protections everything.

The audit confirmed 34 findings, 6 of them high. I fixed all 6 high findings and all 4 nits, plus most of the medium and low ones, and accepted two low-risk findings with a documented reason (below).

The high-severity findings:

1. **One decrypt error could wipe the vault.** If decryption failed, the store read that as "no credentials." The next save then overwrote the encrypted blob with an empty list. *Fix:* a failed decrypt is now a load error, never an empty vault. Writes are refused while the error stands, the user sees a warning banner, and every read-modify-write runs under a mutex.
2. **The biometric check failed open.** On a device with no screen lock or biometric enrolled, the unlock prompt reported success. *Fix:* it now fails closed and tells the user to set a screen lock. It also stays locked if the prompt can't be shown on the current screen.
3. **The re-lock could be bypassed.** A flag that skips re-lock during the app's own system dialogs could stay set forever if the dialog was abandoned. *Fix:* the skip now expires after 60 seconds, and it's read and cleared atomically.
4. **Long passwords were silently truncated.** Characters with no US-keyboard mapping were dropped while typing, so the wrong password went in with no error. *Fix:* the app refuses before it connects if any character can't be typed, and it reports an error if any keystroke is dropped.
5. **Copied passwords stayed on the clipboard.** *Fix:* the clipboard clears after 45 seconds and whenever the app goes to the background. It clears only if it still holds the copied password.
6. **The vault appeared in screenshots and the app switcher.** *Fix:* the whole app is marked secure, so screenshots, screen recordings and the recents thumbnail come out blank.

Selected medium and low fixes:

- **Updates are checked for who signed them, not just their hash.** The updater compared the downloaded APK's SHA-256 with the manifest. But the manifest comes from the same server, so a matching hash only shows the bytes match what the server claimed. The updater now also requires the APK to have our package name and exactly the running app's signing-certificate set before it opens an install session. It fails closed on any read error.
- **No redirects.** The update client no longer follows HTTP redirects. A redirect could have moved the download off the private network and sent the bearer token with it.
- **The key can't be used while the phone is locked.** The vault's AES-256-GCM key now requires an unlocked device. It falls back from StrongBox to the TEE only when StrongBox is missing, not on any key-generation error.
- **Dropped keystrokes are retried and reported.** Each keystroke retries once, and the key is always released.

Later hardening (the four nits):

- The re-lock grace period is capped at 5 minutes (it was up to 1 hour), and turning the screen off forces a re-lock.
- APK downloads are capped at 300 MB, both by the advertised size and during the stream, and stale partial downloads are deleted first.
- Settings changes are atomic, and the updater reads its settings from one consistent snapshot.

**Accepted, not fixed:** a cosmetic connection-status issue that can't send a password to the wrong device, and the update token sitting unencrypted in app-private storage. The app sandbox protects that storage, and the screenshot risk is closed by fix 6. Moving the token to Keystore-backed storage is on the open list below.

## Part 2: A signed update chain

Sideloading means I am the app store. If the update server is compromised, so is every device, unless each device checks what it installs.

### On the device

- **Signer identity before install.** The guardian app and each self-updating app refuse an APK unless its signing certificates match the expected fleet signer. An empty or unreadable certificate set means no install.
- **Checked twice.** The check runs when an APK is stored as a known-good copy and again right before a rollback installs it. A crafted APK can claim any package name, but it can't fake the signature over its bytes.
- **Fail-closed paths need testing on old Android too.** On Android 9 and 10, the system call I used to read an APK file's certificates returns nothing unless a legacy flag is also requested. The guardian then saw every APK as unsigned and refused all installs on my two oldest phones. Failing closed did its job, but it blocked legitimate updates. The fix requests the legacy flag on those versions only.

### Rotating off the debug key

Every app originally shipped signed with the default Android debug key, and that key's password is public. Trusting it is weak, because anyone can sign an APK with an identical-looking default debug key. I moved the fleet to a dedicated release key with **APK Signature Scheme v3 key rotation**. The new signature carries a lineage that proves the release key succeeds the old one, so each phone accepts the update in place, with no uninstall and no data loss.

### The publish gate

A regression made the gate necessary. Two builds of the vault shipped with the correct new key but **no rotation lineage**. Any phone still on the old key would have refused the update, and the only way forward would have been an uninstall that wipes the vault. I caught it before it hit a device. The server now refuses to publish unless all of these hold:

1. The APK's signing certificate matches the one registered for the app.
2. It's a certificate the on-device guardian actually trusts. An incident showed why: a tool defaulted to a per-app key, and the guardian would have rejected that build as foreign even after a reinstall.
3. A rotation lineage is present and contains the app's old certificate, and the current key is the newest entry.
4. Checked against the server's record of which certificate each device has installed: if any device is still on the old certificate, the build must cover it.

Every tooling failure blocks publishing too, such as a missing signing tool, an unreadable APK, or a database read error. An unverified build never ships because a check couldn't run.

### Key custody

Signing keys are never committed to source control. Before an AI agent session's changes ship, the server scans the staged diff for private keys, common token formats and the fleet's own bearer token, and blocks the ship if it finds any. The keys are escrowed as an AES-256 encrypted archive, with hash-verified copies on two separate devices, and the passphrase lives in my password manager.

**Still open:** the guardian still trusts the old debug certificate for apps not yet confirmed on the release key, so older known-good copies can still restore. A per-app release-only list removes that trust as each app is confirmed.

## Part 3: Selected fixes from the fleet-wide audit

The fleet-wide audit covered 13 repositories and confirmed 91 findings: 6 high, 35 medium, 42 low, 8 informational, and no criticals. I shipped 40 fixes, covering every high and every security-relevant medium.

- **A private-network check that could be fooled.** The guardian decided whether a server URL was on my private network by matching part of the hostname. A lookalike host could pass and receive the bearer token. It now checks the full host: the exact private-network name, or an address inside the private range.
- **SSRF in a data importer.** An importer fetched whatever URL the data pointed to. It now requires HTTPS, a host allow-list and a size cap. The same app stopped allowing cleartext traffic globally and caps how deep it walks JSON.
- **A shell-injection boundary in an on-device build tool.** The name of the folder inside an imported project zip reached a shell command. It's now cleaned at that boundary. Package downloads also moved to HTTPS with per-package SHA-256 checks, and build output is capped.
- **Device strings in an elevated PowerShell script.** A Windows tray app embedded the GPU name and device ID that Windows reports into a script run as administrator. They now go through the app's quoting helper, so a crafted device string can't inject commands.
- **A USB-over-network bridge.** It now refuses to listen until a host is paired, bounds-checks USB write sizes, caps frame and buffer sizes, redacts its WireGuard private key from logs, and limits its Windows named pipe to the service, SYSTEM and Administrators.
- **Server-side path traversal.** An authenticated route that creates new app projects now validates the package name against a strict grammar and checks that every write stays inside the target folder.
- **A local power-monitor API** now binds to loopback by default and requires a shared secret before it will listen on any other address.
- **Clipboard hygiene in a form-filling helper.** Values it pastes are flagged sensitive and cleared from the clipboard right after use.

## Part 4: Around the edges

- **Remote desktop** is limited by firewall rule to my private network's address ranges and requires Network Level Authentication. The disk is BitLocker-encrypted, and failed logons are audited.
- **AI agent sessions** that can change code are started from my phone. Every session writes a full transcript to an audit log. I built a challenge-response approval step for them: the phone holds a StrongBox P-256 key that only signs after a fingerprint, and the server checks that signature on a one-time challenge before a session starts. It's been tested against the live server and one phone is enrolled, but it isn't required for every session yet.

## Open items

- Several apps still keep tokens in app-private storage without encrypting them. I'm moving them to Keystore-backed storage one key at a time, testing on each device, because a mistake here breaks the fleet's own sign-in.
- Retiring trust in the old debug certificate, app by app.
- The USB bridge's per-peer pairing check in native code is only partly done. Its native code also wasn't recompiled in the audit pass.
- The 42 low and 8 informational fleet findings are documented but not fixed.

## What I'd tell someone running their own sideloaded apps

- **A hash proves integrity, not authenticity.** If the hash comes from the same server as the file, check the signer too.
- **Fail closed, then test the closed path on the oldest OS you support.** Mine locked out two phones before I noticed.
- **Make the unsafe build impossible to publish.** Don't rely on remembering a flag. The missing-lineage builds passed every check that existed at the time.
- **In a vault, "can't decrypt" and "empty" must be different states.** Treating them the same turns one bad read into permanent data loss.
