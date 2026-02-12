# windows_vmware_tools

Installs VMware Tools on Windows via WinRM.

This role:

- Downloads the VMware Tools installer (`.exe`) and installs it silently.
- Optionally discovers the latest VMware Tools build from VMware's public directory listing.
- Applies a Windows 7 / Windows Server 2008 R2 prerequisite hotfix (KB4474419) when needed.
- Installs the Visual C++ runtime required for VMware guest customization on Windows 7 / 2008 R2.

## Supported platforms

- Windows: works on modern Windows Server versions.
- Windows 7 / Windows Server 2008 R2 (6.1): supported by this role, but note that `ansible.windows` prints a warning that Windows 6.1 will not be supported/tested in future releases.

## Requirements

### Ansible collections

This role uses modules from these collections:

- `ansible.windows`
- `community.windows`
- `chocolatey.chocolatey` (used only as a fallback in some cases)

Install them with:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### Control node Python packages

You need WinRM support (commonly `pywinrm`) on the control node. Exact package set depends on the WinRM transport you use.

## What the role does

### Main flow

The main task flow is in `tasks/main.yml`:

1. **Windows 7 / 2008 R2 only**: apply KB4474419 prerequisite hotfix (see below).
2. Optionally **discover latest** VMware Tools download URL.
3. **Install VMware Tools**.
4. **Windows 7 / 2008 R2 only**: install Visual C++ runtime.
5. Reboot if required.

### VMware Tools installation behavior

Implemented in `tasks/install.yml`:

- Downloads the installer to `temp_directory`.
- Installs silently via `ansible.windows.win_package`.
- If `win_package` fails, falls back to `win_shell` execution, then verifies installation by checking for `vmtoolsd.exe`.
- If all else fails, attempts installation via Chocolatey (`vmware-tools`).
- Always deletes the downloaded installer.

### Windows 7 / 2008 R2 prerequisite hotfix (KB4474419)

Implemented in `tasks/security_update_2008.yml`:

- Downloads KB4474419 (`win_61_security_hot_fix`) to `temp_directory`.
- If the DISM PowerShell module is available, can use `community.windows.win_hotfix`.
- Otherwise, extracts the `.msu` using `wusa.exe` and installs contained `.cab` packages using `dism.exe /add-package`.
- Reboots if the hotfix installation changes the system.

Notes:

- This role **does not install Windows ADK by default** (ADK installers are frequently incompatible on 2008 R2 and may fail to download). If you truly need ADK-provided components, you can opt in.
- On Windows 6.1, the role uses `shutdown.exe` + `wait_for_connection` reboot logic to avoid known `win_reboot` issues in some environments.

## Role variables

Defaults are in `defaults/main.yml`. Architecture mapping is in `vars/main.yml`.

### VMware Tools download selection

| Variable | Default | Description |
|---|---:|---|
| `vmware_tools_find_latest` | `true` | If `true`, scrapes VMware's “latest” directory listing to pick the newest installer. |
| `vmware_tools_download_base_url` | `https://packages.vmware.com/tools/releases/latest/windows/{{ vmware_tools_arch_mapped }}/` | Base URL used by the “find latest” logic. |
| `vmware_tools_download_urls.amd64` | (pinned version) | Pinned download URL for amd64 if `vmware_tools_find_latest: false`. |
| `vmware_tools_download_urls.arm64` | (pinned version) | Pinned download URL for arm64 if `vmware_tools_find_latest: false`. |
| `vmware_tools_download_url` | derived | Effective URL used for download/install. Override to use an internal mirror or local URL. |

### Architecture mapping

| Variable | Derived from | Description |
|---|---|---|
| `win_architecture` | `ansible_architecture` | Maps Ansible arch strings to `amd64`, `x86`, `arm`, `arm64`. |
| `vmware_tools_arch_mapped` | `win_architecture` | Maps to VMware path segment (`x64`, `x86`, `arm`). |

### Installation verification / product ID

| Variable | Default | Description |
|---|---:|---|
| `vmware_tools_install_location` | derived | Path used to verify VMware Tools is installed (`vmtoolsd.exe`). |
| `vmware_tools_product_id` | derived | MSI product ID for some versions (not currently used by the install tasks). |

### Temp / downloads

| Variable | Default | Description |
|---|---:|---|
| `temp_directory` | `{{ ansible_env.TEMP }}` | Where installers/hotfixes are downloaded/extracted. |

### Windows 7 / 2008 R2 hotfix settings

| Variable | Default | Description |
|---|---:|---|
| `win_61_security_hot_fix.kb` | `kb4474419` | Hotfix KB identifier. |
| `win_61_security_hot_fix.file` | (msu filename) | Target filename under `temp_directory`. |
| `win_61_security_hot_fix.url` | Windows Update URL | Download URL for the `.msu`. |

### Optional ADK install (disabled by default)

| Variable | Default | Description |
|---|---:|---|
| `windows_vmware_tools_install_adk` | `false` | If `true`, the role will attempt installing ADK in `security_update_2008.yml` (only if built-in `dism.exe` is not present). |
| `windows_adk_download_url` | `https://go.microsoft.com/fwlink/?linkid=873065` | ADK installer URL used when opting in. |
| `windows_adk_choco_version` | `10.0.17134.0` | Version used by the Chocolatey fallback when opting in. |

## Example usage

### Basic usage

```yaml
- name: Install VMware Tools
  hosts: windows
  roles:
    - oatakan.windows_vmware_tools
```

### Pin a specific VMware Tools version (do not scrape “latest”)

```yaml
- name: Install pinned VMware Tools
  hosts: windows
  vars:
    vmware_tools_find_latest: false
    vmware_tools_download_url: https://packages.vmware.com/tools/releases/12.3.5/windows/x64/VMware-tools-12.3.5-22544099-x86_64.exe
  roles:
    - oatakan.windows_vmware_tools
```

### Using an internal mirror

```yaml
- name: Install VMware Tools from internal mirror
  hosts: windows
  vars:
    vmware_tools_find_latest: false
    vmware_tools_download_url: https://artifacts.example.local/vmware/VMware-tools-x64.exe
  roles:
    - oatakan.windows_vmware_tools
```

### WinRM inventory example

This role does not enforce a specific WinRM transport. Example inventory vars:

```ini
[windows]
winhost ansible_host=windows1.example.local

[windows:vars]
ansible_connection=winrm
ansible_port=5985
ansible_winrm_scheme=http
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
```

Security note: prefer WinRM over HTTPS (5986) and avoid Basic auth over HTTP unless you are in a trusted network.

## Development / testing

- Syntax check: `ansible-playbook tests/test.yml -i tests/inventory --syntax-check`

If you run Ansible on macOS and see fork-safety crashes, setting `OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES` can help.

## License

MIT

## Author

Orcun Atakan
