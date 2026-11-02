---
name: scoop-manifest-generator
description: Inspect Windows EXE installers or official installer download URLs and create, explain, and validate Scoop JSON manifests. Use when asked to package an EXE with Scoop, reverse-engineer its installer layout, determine portable extraction, or generate a bucket manifest.
---

# Scoop Manifest Generator

Create trustworthy Scoop manifests from Windows `.exe` installers. Prefer a portable, versioned application installation, with no machine-wide changes. Never claim a manifest works without testing it.

## Inputs

Accept any of:
- A local `.exe` path and an official, stable download URL for that same binary.
- An official installer download URL.
- The product name and official release page, if the user needs the installer located.

Optional: desired manifest filename, target CPU architecture, version, bucket, portable-data requirements, and whether validation may be performed on a Windows test machine.

A local EXE path alone does **not** create a distributable manifest: Scoop needs a download source accessible to its intended users. Ask for the URL, or clearly label the result a local-only prototype using an explicitly scoped local source if supported and verified.

## Workflow

### 1. Establish provenance and inspect without executing

1. Use official vendor/GitHub release sources; avoid untrusted mirrors.
2. Determine app name, exact version, publisher, download URL, license and architecture from authoritative metadata.
3. For a local file, compute SHA256 with PowerShell: `Get-FileHash -Algorithm SHA256 -LiteralPath 'C:\path\setup.exe'`.
4. Examine Authenticode signature where present: `Get-AuthenticodeSignature -LiteralPath 'C:\path\setup.exe'`. A signature is evidence, not a guarantee of safety.
5. Inspect EXE metadata using PowerShell `Get-Item ... | Select-Object -ExpandProperty VersionInfo`. Version strings may be unrelated to product version; corroborate independently.
6. With 7-Zip available, inspect archive contents with `7z l 'C:\path\setup.exe'`. This does not establish that extraction alone produces a working app.
7. Determine whether the EXE is Inno Setup, NSIS, WiX/Burn, InstallShield, Squirrel/Electron, self-extracting archive, a standalone executable, or unknown. Report uncertainty.
8. Do not execute downloaded installers or run arbitrary embedded scripts just to inspect them. Run installation tests only with user's permission in an appropriate isolated Windows environment.

### 2. Choose the safest workable strategy

Prioritize, in order:
1. Official portable ZIP/7z release (if available and functionally equivalent).
2. Extractable Inno Setup EXE: use `"innosetup": true` after verifying Scoop-compatible extraction and resulting app files.
3. Other extractable EXE: determine whether Scoop's actual extractor handles it; if not, use a verified extraction script with declared dependencies, or provide a supported portable release instead. Test file layout. Do **not** assume every NSIS EXE is portable after `7z x`.
4. Installer that cannot be extracted into a functional app: consider documented silent-install switches only when Scoop compatibility, output location, user permissions, uninstallation and update behavior have been tested. Prefer to explain that conventional setup may be inappropriate for a Scoop package.

Never invent silent arguments (e.g., `/S`, `/VERYSILENT`, `/quiet`) or use them without identifying the actual installer. Never silently install globally, add background services, modify machine PATH or registry, or require administrative privileges without explaining why.

### 3. Construct the manifest

Use supported Scoop fields only as warranted:
- `version`, `description`, `homepage`, `license`, `url`, `hash`.
- `architecture` with distinct 32bit/64bit/arm64 URLs and hashes when actual releases differ.
- `innosetup: true` only when verified suitable for Inno Setup extraction.
- `extract_dir` / `extract_to` only if they match actual archive/extraction layout.
- `bin` only for commands that should be callable from shell.
- `shortcuts` only for verified graphical executables and desired Start Menu names.
- `persist` only for genuine writable config/data paths; verify initial state and migration on upgrades.
- `pre_install`, `installer`, `post_install`, `uninstaller` only where unavoidable and validated.
- `checkver` and `autoupdate` only if the upstream version endpoint/regex and versioned URLs are demonstrably correct.

Match SHA256 to the **exact downloaded bytes**. Never fabricate a hash or use placeholder hexadecimal as if verified. If the hash cannot be computed, mark `hash` as unverified in the report and withhold a release-ready manifest until it is obtained. Prefer canonical HTTPS download links, and mention if URL content may change without its version string changing.

Keep manifest scripts minimal, deterministic, idempotent, and constrained to Scoop-managed directories. Guard missing files with clear errors. Avoid assumptions about user-profile paths. Quote PowerShell paths appropriately. Avoid unnecessary post-install registration, forced launches and network calls.

### 4. Check output correctness

- Parse the JSON with PowerShell `Get-Content -Raw .\app.json | ConvertFrom-Json` or an equivalent JSON parser.
- Where available, run Scoop's manifest validation / `scoop checkup` or the appropriate repository validation tooling; distinguish schema checks from actual installation tests.
- Check that all relative `bin`, `shortcuts`, `persist`, and `extract_dir` paths exist after extraction.
- Verify download and hash against the URL on a test system.
- On an isolated Windows system with permission, test `scoop install .\app.json`, launch the app/CLI, `scoop update`, and `scoop uninstall`. Check no unexpected system changes and that persistent data survives update.
- Never claim the above tests passed unless they were run and the outcome was observed.

### 5. Deliverables

Provide:
1. `<app-name>.json` — valid JSON manifest, when sufficient facts are verified.
2. A short inspection report listing: EXE provenance, installer type (and evidence), version, architecture, extraction strategy, hash status, selected executable, persistent-data location, and unsupported assumptions.
3. Test commands and explicit validation status: `not tested`, `JSON validated`, `Scoop installation tested`, or `install/update/uninstall tested`.
4. Outstanding questions or blockers, particularly missing stable URL, checksum, silent-install support, or functional portability.

If the environment lacks Windows utilities, report the limitation, supply reproducible PowerShell commands, and never pretend an inspection occurred.

## Manifest patterns

### Portable application extracted from an Inno Setup EXE

Use this pattern only after verifying installer type, payload and application paths:

```json
{
  "version": "1.2.3",
  "description": "Short factual description",
  "homepage": "https://example.org",
  "license": "MIT",
  "url": "https://example.org/releases/app-1.2.3-setup.exe",
  "hash": "<verified-sha256>",
  "innosetup": true,
  "shortcuts": [
    ["App.exe", "App"]
  ]
}
```

The hash value above is a **placeholder**, not valid release-ready data. The other fields are illustrative and must not be assumed true of any real application.

### Standalone executable distributed directly

When the executable itself is the application and Scoop keeps it in the version directory, the downloaded filename and actual location must be verified before declaring a shim:

```json
{
  "version": "1.2.3",
  "description": "Short factual description",
  "homepage": "https://example.org",
  "url": "https://example.org/releases/tool-1.2.3.exe#/tool.exe",
  "hash": "<verified-sha256>",
  "bin": "tool.exe"
}
```

Verify the Scoop URL filename-fragment behavior with the installed Scoop version before relying on it.

## Special cases and failure modes

- **NSIS:** Some are extractable with 7-Zip, others require a custom extraction strategy or are not safely portable. Verify file layout and runtime behavior.
- **WiX/Burn / InstallShield:** May wrap MSIs or chained packages; do not assume that extracting embedded files reproduces registration, dependencies or services.
- **Squirrel / Electron:** Look for an official portable release or an extractable app payload; avoid retaining updater integration that interferes with Scoop updates.
- **Drivers, services, shell extensions and COM registration:** Often unsuitable for a simple portable Scoop manifest; explicitly describe required system changes.
- **Mutable latest-release URLs:** Download the exact target and verify the computed checksum; add `checkver`/`autoupdate` only after verification.
- **App data:** Inspect whether configuration lives in `%APPDATA%`, `%LOCALAPPDATA%`, beside the binary, or elsewhere. Don't claim `persist` works for app data outside Scoop's install directory without a validated migration/redirection method.
- **Terms and licenses:** Don't redistribute or repackage binaries in violation of their terms.

## Response style

Prefer verified facts over optimistic automation. Briefly explain *why* each manifest field is needed. If the user provides an EXE only, provide a meaningful inspection and ask for the official URL only if needed to finish a distributable manifest. Mark assumptions and untested behavior clearly.

## Official references

- Scoop manifest reference: https://github.com/ScoopInstaller/Scoop/wiki/App-Manifests
- Creating a manifest: https://github.com/ScoopInstaller/Scoop/wiki/Creating-an-app-manifest
- Scoop hooks and scripts: https://github.com/ScoopInstaller/Scoop/wiki/Pre-Post-%28un%29install-scripts
- Scoop implementation: https://github.com/ScoopInstaller/Scoop
