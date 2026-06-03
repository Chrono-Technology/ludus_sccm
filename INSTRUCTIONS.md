# Chrono Ludus SCCM Deployment Instructions

Use this file as the handoff prompt/runbook for an LLM or operator that needs to deploy this SCCM hierarchy lab from the Chrono-Technology fork.

The source repo is `https://github.com/Chrono-Technology/ludus_sccm`, but the Ansible collection namespace remains `mayyhem.ludus_sccm`. Do not rename the namespace unless you also update every role reference in `new-config.yml`.

## What Gets Deployed

`new-config.yml` deploys a `mayyhem.com` SCCM hierarchy:

- `CAS`: central administration site on `cas-pss`, database on `cas-db`, service connection point on `cas-scp`
- `PS1`: child primary site on `ps1-pss`, database on `ps1-db`, SMS provider on `ps1-sms`, management point on `ps1-mp`, PXE distribution point on `ps1-dp`, content library on `ps1-lib`, passive site server on `ps1-psv`
- `SEC`: child secondary site on `ps1-sec`, with secondary SQL Express, management point, and distribution point roles
- `ps1-dev`: Windows 11 workstation for validation and lab use

Expected VM count: 14 including the Ludus router.

## Inputs Required

Set these before starting:

```powershell
$LUDUS_URL = "https://<ludus-server>:8080"
$LUDUS_API_KEY = "<ludus-api-key-for-deploy-user>"
$REPO = "$env:USERPROFILE\Documents\GitHub\ludus_sccm"
```

Use a dedicated Ludus user for large SCCM deployments. If you have an admin/root Ludus API key and need a fresh user:

```powershell
$ADMIN_API_KEY = "<admin-or-root-ludus-api-key>"
$NEW_USER = "SCCMADM$(Get-Date -Format MMdd)"

$env:LUDUS_API_KEY = $ADMIN_API_KEY
ludus --url $LUDUS_URL users add -i $NEW_USER -n "SCCM Admin $(Get-Date -Format yyyy-MM-dd)" -a
ludus --url $LUDUS_URL users apikey --user $NEW_USER --no-prompt
```

Store the returned deploy-user API key locally:

```powershell
$env:LUDUS_API_KEY = $LUDUS_API_KEY
ludus --url $LUDUS_URL apikey
ludus --url $LUDUS_URL version
```

## Prerequisites

The Ludus host must already have:

- Ludus server and client working
- Proxmox capacity for a large lab, roughly 16 CPU cores, 64 GB RAM, and 256 GB free disk as a practical minimum
- Templates named `win2022-server-x64-template` and `win11-22h2-x64-enterprise-template`
- Internet access from the VMs during deployment unless you have mirrored all packages
- The Windows 11 22H2 Enterprise eval ISO available to the SCCM role for OSD import

Known validated versions:

- Ludus `1.9.6+118a007`
- Collection version `1.0.6`
- Config file `new-config.yml`

## Clone Or Update The Fork

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\Documents\GitHub" | Out-Null
if (Test-Path $REPO) {
  git -C $REPO fetch --all --prune
  git -C $REPO checkout main
  git -C $REPO pull --ff-only
} else {
  git clone https://github.com/Chrono-Technology/ludus_sccm.git $REPO
}
cd $REPO
git status --short --branch
```

## Install The Collection

Fast path from Ansible Galaxy:

```powershell
ludus --url $LUDUS_URL ansible collection add mayyhem.ludus_sccm --version 1.0.6 -f
```

Source-fork path, if you need the exact code in this repo:

```powershell
cd $REPO
ansible-galaxy collection build --force
$artifact = Get-ChildItem $REPO -Filter "mayyhem-ludus_sccm-*.tar.gz" |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

python -m http.server 8081
```

In another shell, install the artifact into Ludus. Replace `<operator-ip>` with an address reachable from the Ludus server:

```powershell
ludus --url $LUDUS_URL ansible collection add "http://<operator-ip>:8081/$($artifact.Name)" -f
```

Verify:

```powershell
ludus --url $LUDUS_URL ansible collection list
```

## Deploy

```powershell
cd $REPO
ludus --url $LUDUS_URL range config set -f .\new-config.yml
ludus --url $LUDUS_URL range deploy
ludus --url $LUDUS_URL range logs -f
```

If VM provisioning is already complete and only the user-defined roles need to be rerun:

```powershell
ludus --url $LUDUS_URL range deploy -t user-defined-roles
```

Check deployment state:

```powershell
ludus --url $LUDUS_URL range list
ludus --url $LUDUS_URL range errors
```

## Validation Checklist

Do not stop validation as soon as Ludus says `SUCCESS`. SCCM continues installing some passive and secondary-site components asynchronously.

Minimum checks:

```powershell
ludus --url $LUDUS_URL range list
ludus --url $LUDUS_URL range errors
```

On the Ludus host, run Ansible checks from `/opt/ludus/ansible/range-management`. Replace `<lowercase-ludus-user>` and `<proxmox-user>` as needed:

```bash
cd /opt/ludus/ansible/range-management
export PROXMOX_URL=https://127.0.0.1:8006
export PROXMOX_USERNAME='<proxmox-user>@pam'
export PROXMOX_PASSWORD="$(cat /opt/ludus/users/<lowercase-ludus-user>/proxmox_password)"
export PROXMOX_INVALID_CERT=1
export ANSIBLE_HOST_KEY_CHECKING=False

ansible -i proxmox.py '<LUDUS_USER>-*:!<LUDUS_USER>-router-debian11-x64' -m ansible.windows.win_ping -a '' -o
```

Domain join:

```bash
ansible -i proxmox.py '<LUDUS_USER>-*:!<LUDUS_USER>-router-debian11-x64' \
  -m ansible.windows.win_shell \
  -a '$cs=Get-CimInstance Win32_ComputerSystem; "$env:COMPUTERNAME Domain=$($cs.Domain) PartOfDomain=$($cs.PartOfDomain)"' -o
```

SCCM provider and site status, run as `MAYYHEM\domainadmin`:

```bash
ansible -i proxmox.py '<LUDUS_USER>-cas-pss:<LUDUS_USER>-ps1-pss' \
  -m ansible.windows.win_shell \
  -a '$siteCode=if($env:COMPUTERNAME -like "CAS-*"){"CAS"}else{"PS1"}; $ns="root\sms\site_$siteCode"; Get-CimInstance -Namespace root\sms -ClassName SMS_ProviderLocation | Select SiteCode,Machine,NamespacePath; Get-CimInstance -Namespace $ns -ClassName SMS_Site | Select SiteCode,SiteName,Type,ServerName,ReportingSiteCode,Status,RequestedStatus,Version,BuildNumber; Get-CimInstance -Namespace $ns -ClassName SMS_SecondarySiteStatus -ErrorAction SilentlyContinue | Sort MessageTime | Select -Last 8 SiteCode,MessageTime,StatusID,Status,Description' \
  -b --become-method runas --become-user 'MAYYHEM\domainadmin' \
  -e 'ansible_become_password=password ansible_become_flags=logon_type=interactive logon_flags=with_profile' -o
```

Expected final SCCM site status:

- `CAS` status `1`
- `PS1` status `1`
- `SEC` status `1`
- Secondary status ends with `ConfigMgr Setup - Installation success` and `Bootstrap service completed successfully`

ConfigMgr PowerShell provider test on `ps1-pss`:

```powershell
$module = "C:\Program Files (x86)\Microsoft Configuration Manager\AdminConsole\bin\ConfigurationManager.psd1"
Import-Module $module -Force
New-PSDrive -Name PS1 -PSProvider CMSite -Root "ps1-pss.mayyhem.com"
Set-Location PS1:
Get-CMSite
Get-CMTaskSequence -Name "Deploy-OS"
Get-CMOperatingSystemImage -Name "Windows11"
```

Management point HTTP checks from `ps1-dev`:

```powershell
Invoke-WebRequest -UseBasicParsing http://ps1-mp.mayyhem.com/sms_mp/.sms_aut?mplist
Invoke-WebRequest -UseBasicParsing http://ps1-mp.mayyhem.com/sms_mp/.sms_aut?mpcert
Invoke-WebRequest -UseBasicParsing http://ps1-sec.mayyhem.com/sms_mp/.sms_aut?mplist
Invoke-WebRequest -UseBasicParsing http://ps1-sec.mayyhem.com/sms_mp/.sms_aut?mpcert
```

PXE DP check on `ps1-dp`:

```powershell
Get-Service WDSServer,W3SVC,CCMEXEC
Test-Path C:\RemoteInstall
Test-Path C:\SMS_DP$
Test-Path C:\SCCMContentLib
```

OSD objects expected in WMI/ConfigMgr:

- OS image `Windows11`, package `PS100005`, image OS version `10.0.22621.525`
- Task sequence `Deploy-OS`, package `PS100006`
- Deployment `Deploy-OS_PS100006_AllUnknownComputers`, advertisement `PS120001`

## Known Recovery Notes

If Windows package installation fails because Chocolatey is missing on a host, confirm whether another host has `C:\ProgramData\chocolatey` and repair the broken host before rerunning `ludus range deploy -t user-defined-roles`.

If OSD import fails because the expected Windows ISO is missing, check the Ludus host under `/var/lib/vz/template/iso`. The previously validated lab expected:

```text
/var/lib/vz/template/iso/22621.525.220925-0207.ni_release_svc_refresh_CLIENTENTERPRISEEVAL_OEMRET_x64FRE_en-us.iso
```

If the same Win11 22H2 Enterprise eval ISO exists under another name, create a symlink to the expected filename and rerun user-defined roles.

If deployment stalls after initial VM setup:

```powershell
ludus --url $LUDUS_URL power off -n all
Start-Sleep -Seconds 300
ludus --url $LUDUS_URL power on -n all
Start-Sleep -Seconds 300
ludus --url $LUDUS_URL range deploy -t user-defined-roles
ludus --url $LUDUS_URL range logs -f
```

## Power Control

Power off without destroying:

```powershell
ludus --url $LUDUS_URL power off -n all
ludus --url $LUDUS_URL range list
```

Power on again:

```powershell
ludus --url $LUDUS_URL power on -n all
ludus --url $LUDUS_URL range list
```

Destroy only when explicitly requested:

```powershell
ludus --url $LUDUS_URL range rm
```
