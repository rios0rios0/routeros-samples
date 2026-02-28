# Contributing

Contributions are welcome. By participating, you agree to maintain a respectful and constructive environment.

For coding standards, testing patterns, architecture guidelines, commit conventions, and all
development practices, refer to the **[Development Guide](https://github.com/rios0rios0/guide/wiki)**.

## Prerequisites

- [MikroTik RouterOS](https://mikrotik.com/software) v6.36+ device or [CHR](https://mikrotik.com/download) virtual machine for testing
- [WinBox](https://mikrotik.com/download) v3.x+ or SSH client for device access
- A text editor with RouterOS scripting language support (e.g., [VS Code](https://code.visualstudio.com/) with a RouterOS syntax extension)

## Development Workflow

1. Fork and clone the repository
2. Create a branch: `git checkout -b feat/my-change`
3. Edit or create `.rsc` script files in the appropriate directory (`Layer7 Filter/` or `LoadBalancer/`)
4. Validate your script by importing it into a test RouterOS device:
   ```
   # Upload the .rsc file to the device via WinBox (Files) or SCP
   scp my-script.rsc admin@<router-ip>:/

   # SSH into the router and import the script
   /import file-name=my-script.rsc
   ```
5. Test the auto-balance script if applicable:
   ```
   /system script run <SCRIPT_NAME>
   ```
6. Verify the configuration was applied correctly:
   ```
   /ip firewall filter print
   /ip firewall mangle print
   /ip route print
   ```
7. Commit following the [commit conventions](https://github.com/rios0rios0/guide/wiki/Life-Cycle/Git-Flow)
8. Open a pull request against `main`
