# my_bucket
my scoop bucket

Install Delta 0.19.2 (Windows x64) from a regular PowerShell terminal:

```powershell
scoop install .\delta-dev.json
```

The manifest extracts the WiX bundle and its MSI using `dark` and `lessmsi`,
which Scoop installs automatically. It does not run the installer or require an
administrator password. Launch Delta from the Start menu or run `delta`.

The package is named `delta-dev` because Scoop's main bucket already uses `delta`
for the Git diff viewer. Both applications use the `delta` command, so their
command-line shims can conflict if both are installed.
