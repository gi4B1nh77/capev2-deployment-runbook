# CAPEv2 Deployment Runbook

> **Validated lab profile:** Ubuntu 22.04.5 + KVM/libvirt + Windows 10 22H2 x64 + CAPEv2

Deployment runbook for building and operating a CAPEv2 malware-analysis sandbox on an **existing Ubuntu 22.04.5 host**. Ubuntu installation itself is out of scope.

> [!WARNING]
> This repository documents a **disposable malware-analysis lab**. Do not run malware samples on the Ubuntu host. Do not commit real credentials, malware binaries, VM disks, ISO files, PCAPs, or analysis reports containing sensitive data.

## Current validated state

| Component | Validated value |
|---|---|
| Host OS | Ubuntu 22.04.5 |
| Virtualization | KVM/libvirt on a VMware-backed host with nested virtualization |
| CAPE root | `/opt/CAPEv2` |
| Bridge | `virbr0 = 192.168.122.1/24` |
| Host uplink | `ens34` |
| Windows guest | `cuckoo1 = 192.168.122.100/24` |
| Guest OS architecture | Windows 10 22H2 x64 |
| CAPE Agent | TCP/8000 |
| ResultServer | `192.168.122.1:2042` |
| Guest Python | Python 3.9.13 x86 |
| Current snapshot | `clean-realistic` |

> [!IMPORTANT]
> The current `clean-realistic` snapshot was created while the Windows and VirtIO ISO media were attached. Keep those ISO files available while this snapshot remains in use; deleting them can make libvirt snapshot restore fail.

## Repository safety

Keep secrets as placeholders such as `CHANGE_ME_DB_PASSWORD` and `<HOST_ADMIN_USER>`. This README intentionally does **not** contain the real PostgreSQL password used by the lab.

---

## 0. Scope, assumptions, and values to set once

This document assumes Ubuntu is already installed on the target host. It includes everything from host preflight, KVM/libvirt, CAPEv2 installation and service validation through Windows guest preparation, snapshot creation, runtime/cron checks, and end-to-end submission.

| Item | Value / requirement |
| --- | --- |
| <HOST_ADMIN_USER> | Existing sudo-capable Linux login used to install KVM; example: itsec01 |
| CHANGE_ME_DB_PASSWORD | Password used by PostgreSQL role cape and the CAPE connection string |
| NETWORK_IFACE | virbr0 |
| IFACE_IP | 192.168.122.1 |
| UPLINK | ens34 |
| GUEST_NAME | cuckoo1 |
| GUEST_IP | 192.168.122.100 |
| SNAPSHOT | clean-realistic |

> Warning: If CHANGE_ME_DB_PASSWORD contains URI-reserved characters such as @, :, /, # or %, URL-encode it before placing it in a PostgreSQL URI.

## 1. Host preflight on the existing Ubuntu machine

### 1.1 Confirm OS, CPU virtualization, nested virtualization and capacity

```bash
cat /etc/os-release
uname -a
lscpu | egrep 'Model name|Virtualization|CPU\(s\)'
egrep -c '(vmx|svm)' /proc/cpuinfo
free -h
df -hT / /opt /var/lib/libvirt 2>/dev/null || true
```

The virtualization count must be greater than zero. If it is 0 on a VMware-backed Ubuntu VM, enable nested virtualization in the outer VMware layer before continuing.

### 1.2 Capture the current network state

```bash
ip -br addr
ip route
ip route | grep '^default'
resolvectl status 2>/dev/null | head -80 || true
```

For this validated host, the intended Internet/default-route uplink is ens34. The CAPE analysis bridge will be virbr0.

### 1.3 Install basic administration/build utilities

```bash
sudo apt-get update
sudo apt-get install -y git curl wget tmux jq unzip p7zip-full ca-certificates gnupg lsb-release build-essential python3 python3-pip python3-venv acpica-tools tcpdump
```

> Note: Run the long installer steps from tmux. Upstream CAPEv2 explicitly recommends tmux because KVM/CAPE installation is lengthy and an SSH disconnect can interrupt it.

## 2. Clone CAPEv2 - do not use the cape user yet

At this point the cape user may not exist. Clone as the current administrative account/root. Do not run chown cape:cape yet.

```bash
sudo mkdir -p /opt
cd /opt
sudo git clone https://github.com/kevoreilly/CAPEv2.git
cd /opt/CAPEv2
git rev-parse --short HEAD
git status --short
```

> Important: If /opt/CAPEv2 already exists and you are intentionally rebuilding, archive the current conf/ and installation logs before replacing it.

## 3. Prepare installer hardware replacement token

### 3.1 Extract ACPI identifiers

```bash
cd ~
sudo acpidump > acpidump.out
sudo acpixtract -a acpidump.out
sudo iasl -d dsdt.dat
grep -iE 'Hardware ID|OEMID|OEM Table ID|Manufacturer' dsdt.dsl | head -30
```

For this lab, use the neutral four-character replacement value:

```bash
ABCD
```

### 3.2 Replace <WOOT> in kvm-qemu.sh

```bash
cd /opt/CAPEv2/installer
grep -n '<WOOT>' kvm-qemu.sh
sudo sed -i 's/<WOOT>/ABCD/g' kvm-qemu.sh
grep -n '<WOOT>' kvm-qemu.sh
grep -n 'ABCD' kvm-qemu.sh | head -30
sudo chmod +x kvm-qemu.sh
```

The second grep for <WOOT> should return no lines.

## 4. Install and validate KVM/QEMU/libvirt

### 4.1 Review the KVM installer before executing it

```bash
cd /opt/CAPEv2/installer
./kvm-qemu.sh -h
```

Use the existing host administrator account here, not the cape service account. Example command pattern:

```bash
cd /opt/CAPEv2/installer
HOST_ADMIN_USER="itsec01"   # change if your sudo login is different
sudo ./kvm-qemu.sh all "$HOST_ADMIN_USER" 2>&1 | tee /root/kvm-qemu-install.log
```

> Important: The exact kvm-qemu.sh CLI can change over time. Always run -h from the checked-out script first. This runbook intentionally does not invent a cape user before cape2.sh creates it.

### 4.2 Reboot after KVM installation

```bash
sudo reboot
```

### 4.3 Validate KVM after reboot

```bash
lsmod | grep -E '^kvm|kvm_intel|kvm_amd'
virsh -c qemu:///system version
virsh -c qemu:///system list --all
systemctl status libvirtd --no-pager 2>/dev/null || systemctl status virtqemud --no-pager
```

### 4.4 Validate or create the libvirt default network

```bash
virsh -c qemu:///system net-list --all
ip -br addr show virbr0 2>/dev/null || true
```

If the default network already exists and virbr0 has 192.168.122.1/24, do not recreate it. If the default network is present but stopped:

```bash
sudo virsh -c qemu:///system net-start default
sudo virsh -c qemu:///system net-autostart default
```

Verify:

```bash
virsh -c qemu:///system net-info default
ip -br addr show virbr0
ip route | grep 192.168.122.0
```

> Validated: Expected bridge address for this deployment: 192.168.122.1/24.

## 5. Configure cape2.sh

### 5.1 Review the help from the script actually present on this host

```bash
cd /opt/CAPEv2/installer
chmod +x cape2.sh
./cape2.sh -h
```

The validated installer on this host showed the syntax:

```bash
./cape2.sh <command> <iface_ip> [options]
```

Therefore this runbook uses the host's actual syntax rather than assuming a different upstream revision.

### 5.2 Edit the three required values

```bash
sudo nano /opt/CAPEv2/installer/cape2.sh
```

Set:

```ini
NETWORK_IFACE=virbr0
IFACE_IP="192.168.122.1"
PASSWD="CHANGE_ME_DB_PASSWORD"
```

Do not manually force INTERNET_IFACE unless there is a specific reason; the script version used here detects the default-route interface.

### 5.3 Verify the edited header

```bash
grep -nE '^NETWORK_IFACE|^IFACE_IP|^PASSWD|^INTERNET_IFACE' /opt/CAPEv2/installer/cape2.sh
```

## 6. Install CAPEv2 Base and create the cape runtime user

### 6.1 Run Base installation

```bash
cd /opt/CAPEv2/installer
sudo bash cape2.sh Base 192.168.122.1 --disable-libvirt 2>&1 | tee /root/cape-base-install.log
```

> Note: --disable-libvirt is used because KVM/libvirt was installed and validated in the previous section. This prevents cape2.sh from unnecessarily replacing that layer.

### 6.2 Now verify that the cape user exists

```bash
id cape
getent passwd cape
getent group cape
```

Only now is it valid to use commands such as sudo -u cape or chown cape:cape.

### 6.3 Verify CAPE ownership and runtime groups

```bash
stat -c '%U:%G %a %n' /opt/CAPEv2
id cape
```

If /opt/CAPEv2 is not owned appropriately after installation:

```bash
sudo chown -R cape:cape /opt/CAPEv2
```

Verify libvirt access for the service account:

```bash
sudo -u cape virsh -c qemu:///system list --all
```

If libvirt permission is denied, inspect groups first:

```bash
id cape
getent group libvirt
getent group kvm
```

Only if cape is missing from those groups:

```bash
sudo usermod -aG libvirt,kvm cape
sudo systemctl restart libvirtd 2>/dev/null || true
```

### 6.4 Verify Poetry environment

```bash
ls -l /etc/poetry/bin/poetry
cd /opt/CAPEv2
sudo -u cape -H /etc/poetry/bin/poetry env list
sudo -u cape -H /etc/poetry/bin/poetry install
```

> Important: Run CAPE application commands as cape. Root is reserved for installer/rooter/privileged networking actions.

## 7. PostgreSQL, MongoDB and application database

### 7.1 Check what Base installed

```bash
systemctl status postgresql --no-pager || true
systemctl status mongod --no-pager || systemctl status mongodb --no-pager || true
psql --version || true
mongod --version 2>/dev/null | head -5 || true
```

If PostgreSQL is missing, use the installer command exposed by ./cape2.sh -h:

```bash
cd /opt/CAPEv2/installer
sudo bash cape2.sh PostgreSQL 192.168.122.1 | tee /root/cape-postgresql-install.log
```

If MongoDB is missing:

```bash
cd /opt/CAPEv2/installer
sudo bash cape2.sh Mongo 192.168.122.1 | tee /root/cape-mongo-install.log
```

### 7.2 Verify PostgreSQL role/database

```bash
sudo -u postgres psql -c "\du"
sudo -u postgres psql -c "\l"
```

Expected role/database: cape. If either is missing, create only the missing object.

```bash
sudo -u postgres psql
```

```bash
CREATE USER cape WITH PASSWORD 'CHANGE_ME_DB_PASSWORD';
CREATE DATABASE cape OWNER cape;
\q
```

> Warning: Do not rerun CREATE USER/CREATE DATABASE if they already exist. This fallback is for a truly incomplete installer run.

### 7.3 Test authentication exactly as CAPE will use it

```bash
PGPASSWORD='CHANGE_ME_DB_PASSWORD' psql -h localhost -U cape -d cape -c "SELECT 1;"
```

### 7.4 Verify MongoDB locally

```bash
mongosh --quiet --eval 'db.runCommand({ping:1})' 2>/dev/null || mongo --quiet --eval 'db.runCommand({ping:1})' 2>/dev/null || true
ss -lntp | grep 27017 || true
```

## 8. Configure CAPEv2 host files

### 8.1 cuckoo.conf - PostgreSQL and ResultServer

Edit:

```bash
sudo -u cape nano /opt/CAPEv2/conf/cuckoo.conf
```

Ensure the database connection points to the cape database and the ResultServer binds to virbr0:

```ini
connection = postgresql://cape:CHANGE_ME_DB_PASSWORD@localhost:5432/cape

[resultserver]
ip = 192.168.122.1
port = 2042
```

### 8.2 routing.conf

```bash
sudo -u cape nano /opt/CAPEv2/conf/routing.conf
```

```ini
[routing]
enable_pcap = yes
route = internet
internet = ens34
nat = yes
```

### 8.3 auxiliary.conf - packet capture

Ensure the sniffer uses virbr0 and /usr/bin/tcpdump. Keep Windows screenshots enabled if desired.

### 8.4 reporting.conf - MongoDB

- Ensure MongoDB reporting is enabled and points to the local service:

```bash
host = 127.0.0.1
port = 27017
db = cuckoo
enabled = yes
```

Install all dependencies and enable Mitre, Bingraph
```bash

cd /opt/CAPEv2

sudo apt-get install -y python3-tk

sudo -u cape -H /etc/poetry/bin/poetry run pip install -r /opt/CAPEv2/extra/optional_dependencies.txt

sudo -u cape -H /etc/poetry/bin/poetry run python -c "from binGraph.binGraph import generate_graphs; print('binGraph OK')"

sudo -u cape -H /etc/poetry/bin/poetry run pip install -U git+https://github.com/CAPESandbox/pyattck/

```
```bash
[bingraph]
```bash
enabled = yes
on_demand = yes
binary = yes
cape = yes
....
[mitre]
enabled = yes
```
```bash
systemctl restart cape-processor cape-web
```

### 8.5 kvm.conf - create the machine entry before snapshot name exists

```bash
sudo -u cape nano /opt/CAPEv2/conf/kvm.conf
```

```ini
[kvm]
machines = cuckoo1
interface = virbr0
dsn = qemu:///system

[cuckoo1]
label = cuckoo1
platform = windows
ip = 192.168.122.100
resultserver_ip = 192.168.122.1
arch = x64
tags = win10,x64
```

The snapshot line is added later after the known-good Windows baseline is created.

### 8.6 Check for stale default IPs

```bash
grep -RniE '192\.168\.1\.1|virbr1' /opt/CAPEv2/conf || true
```

> Validated: The actual failure observed during rebuild was ResultServer still binding to 192.168.1.1:2042. Eliminate stale defaults before starting cape.service.

## 9. Runtime services, systemd, rooter, and cron checks

### 9.1 Reload and inspect installed unit files

```bash
sudo systemctl daemon-reload
systemctl cat cape
systemctl cat cape-rooter
systemctl cat cape-processor
systemctl cat cape-web
```

### 9.2 Enable boot-time runtime services

```bash
sudo systemctl enable cape-rooter cape cape-processor cape-web
sudo systemctl enable postgresql
sudo systemctl enable cron
sudo systemctl enable libvirtd 2>/dev/null || true
sudo systemctl enable mongod 2>/dev/null || sudo systemctl enable mongodb 2>/dev/null || true
```

```bash
sudo systemctl enable libvirtd
```

```bash
sudo systemctl enable libvirtd.socket
```

```bash
virsh -c qemu:///system net-autostart default
```

### 9.3 Start rooter first

```bash
sudo systemctl restart cape-rooter
systemctl status cape-rooter --no-pager
ls -l /tmp/cuckoo-rooter 2>/dev/null || true
```

Expected rooter socket ownership is typically root:cape with cape able to communicate with it.

### 9.4 Run Django migrations after rooter is available

```bash
cd /opt/CAPEv2
sudo -u cape -H /etc/poetry/bin/poetry run python manage.py migrate
```

### 9.5 Start the remaining CAPE services

```bash
sudo systemctl restart cape
sudo systemctl restart cape-processor
sudo systemctl restart cape-web
```

### CAPE WEB
```bash

systemctl cat cape-web

ss -lntp | grep -E ':8000|:8080|:8888'
```

bind Web to localhost: Change ExecStart = ...from 0.0.0.0:8000 to 127.0.0.1:8000

Change:
```bash

systemctl edit --full cape-web

systemctl daemon-reload

systemctl restart cape-web

ssh -L 8000:127.0.0.1:8000 username@172.16.24.42 (run on host)
```

### 9.6 One-command runtime status check

```bash
systemctl --no-pager --full status cape-rooter cape cape-processor cape-web postgresql cron $(systemctl list-unit-files --type=service | awk '/^mongod\.service|^mongodb\.service/{print $1; exit}')
```

### 9.7 Check which Linux user each CAPE service actually runs as

```bash
systemctl show cape -p User -p Group -p ExecStart -p WorkingDirectory
systemctl show cape-processor -p User -p Group -p ExecStart -p WorkingDirectory
systemctl show cape-web -p User -p Group -p ExecStart -p WorkingDirectory
systemctl show cape-rooter -p User -p Group -p ExecStart -p WorkingDirectory
```

Expected design: cape/cape for normal CAPE application services; rooter is the privileged exception.

### 9.8 Validate tcpdump privilege path

```bash
sudo -l -U cape | grep -i tcpdump || true
sudo -u cape sudo -n /usr/bin/tcpdump --version | head -3
```

### 9.9 Cron runtime checks - do not skip

The installer may create root cron entries for components such as MongoDB initialization or optional Suricata rule updates. Confirm cron itself is running and inspect both root and cape crontabs:

```bash
systemctl status cron --no-pager
systemctl is-enabled cron
sudo crontab -l 2>/dev/null || true
sudo -u cape crontab -l 2>/dev/null || true
sudo grep -RniE 'cape|cuckoo|mongodb|mongod|suricata' /etc/cron.d /etc/cron.daily /etc/cron.hourly /var/spool/cron/crontabs 2>/dev/null || true
```

> Important: Do not delete an installer-created @reboot MongoDB entry or Suricata update job just because it is unfamiliar. First identify which installer option created it and whether that component is enabled.

### 9.10 Recommended operational health check via cron

This is an operational addition, not a CAPEv2 requirement. It logs runtime state every five minutes without auto-restarting services, so failures remain visible.

```bash
sudo tee /usr/local/sbin/cape-runtime-check.sh > /dev/null <<'EOF'
#!/bin/bash
TS="$(date -Is)"
SERVICES=(cron postgresql cape-rooter cape cape-processor cape-web)
for s in "${SERVICES[@]}"; do
    if systemctl is-active --quiet "$s"; then
        echo "$TS OK service=$s"
    else
        echo "$TS FAIL service=$s state=$(systemctl is-active "$s" 2>/dev/null)"
    fi
done

for s in mongod mongodb libvirtd virtqemud; do
    if systemctl list-unit-files --type=service | grep -q "^${s}.service"; then
        if systemctl is-active --quiet "$s"; then
            echo "$TS OK service=$s"
        else
            echo "$TS FAIL service=$s state=$(systemctl is-active "$s" 2>/dev/null)"
        fi
    fi
done

ip -br addr show virbr0 2>/dev/null | sed "s/^/$TS NET /"
ss -lnt 2>/dev/null | grep -E '(:2042|:27017)' | sed "s/^/$TS LISTEN /"
virsh -c qemu:///system net-info default 2>/dev/null | awk -v ts="$TS" '/Active:|Autostart:/{print ts " LIBVIRT " $0}'
virsh -c qemu:///system domstate cuckoo1 2>/dev/null | sed "s/^/$TS VM cuckoo1=/"
EOF

sudo chmod 750 /usr/local/sbin/cape-runtime-check.sh
sudo /usr/local/sbin/cape-runtime-check.sh
```

Install the cron entry:

```bash
echo '*/5 * * * * root /usr/local/sbin/cape-runtime-check.sh >> /var/log/cape-runtime-check.log 2>&1' | sudo tee /etc/cron.d/cape-runtime-check
sudo chmod 644 /etc/cron.d/cape-runtime-check
sudo systemctl restart cron
sudo tail -n 50 /var/log/cape-runtime-check.log 2>/dev/null || true
```

Check virtual network:
```bash

systemctl is-active libvirtd

virsh -c qemu:///system net-list --all

ip -br addr show virbr0

systemctl is-active cape-rooter cape cape-processor cape-web postgresql cron

ss -lntp | grep -E '2042|27017'
```

Add log rotation:

```bash
sudo tee /etc/logrotate.d/cape-runtime-check > /dev/null <<'EOF'
/var/log/cape-runtime-check.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```

## 10. Host acceptance before creating the Windows guest

```bash
systemctl is-active cape-rooter cape cape-processor cape-web postgresql cron
ss -lntp | grep 2042
ip -br addr show virbr0
virsh -c qemu:///system net-info default
journalctl -u cape -n 50 --no-pager
journalctl -u cape-rooter -n 50 --no-pager
```

> Warning: Do not proceed if cape.service is crash-looping. Resolve the first CRITICAL traceback before building the guest.

## 11. Prepare ISO files and create Windows VM

### 11.1 Required files

```bash
sudo mkdir -p /var/lib/libvirt/images/iso
ls -lh /var/lib/libvirt/images/iso
```

Place the following files in the ISO directory:

Windows 10 22H2 x64 ISO, validated filename: Win10_22H2_English_x64v1.iso

VirtIO driver ISO, validated filename: virtio-win-0.1.285.iso

### 11.2 Create the VM

```bash
sudo /usr/bin/virt-install --name cuckoo1 --ram 8192 --vcpus 4 --os-variant win10 --disk path=/var/lib/libvirt/images/cuckoo1.qcow2,size=40,bus=virtio --cdrom /var/lib/libvirt/images/iso/Win10_22H2_English_x64v1.iso --disk path=/var/lib/libvirt/images/iso/virtio-win-0.1.285.iso,device=cdrom --network network=default,model=virtio --graphics vnc,listen=127.0.0.1 --clock hpet_present=yes --noautoconsole
```

### 11.3 Verify VM definition before installation

```bash
virsh -c qemu:///system list --all
virsh -c qemu:///system dominfo cuckoo1
virsh -c qemu:///system domblklist cuckoo1
virsh -c qemu:///system domiflist cuckoo1
```

> Warning: Never use 'virsh undefine cuckoo1 --remove-all-storage' as a casual cleanup command. It can remove attached storage including ISO files. Undefine the domain first, then delete only the exact qcow2 path after verification if a rebuild is truly required.

## 12. Install Windows 10 and VirtIO drivers

Connect to this host through port 5905, use Tight VNC to connect (Run on hardware host not VM) – forward port bc Ubuntu Server do not have GUI.
```bash

ssh -L 5905:127.0.0.1:5900 username@172.16.24.42
```

### 12.1 Windows setup

Boot the VM and complete Windows 10 Pro 22H2 x64 installation. When Windows Setup cannot see the VirtIO disk, load the storage driver from the VirtIO ISO appropriate for Windows 10 amd64.
```bash

vioscsi\w10\amd64 OR viostor\w10\amd64
```

### 12.2 NIC driver

After first boot, install the VirtIO network driver from:

```bash
NetKVM\w10\amd64
```

### 12.3 Configure static guest network

Identify the adapter:

```powershell
Get-NetAdapter
```

Then configure the active Ethernet adapter. Replace 'Ethernet' only if the alias differs:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.122.100 -PrefixLength 24 -DefaultGateway 192.168.122.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.122.1
ipconfig /all
```

Validate host reachability:

```powershell
Test-NetConnection 192.168.122.1
```

## 13. Prepare the disposable Windows sandbox baseline

Run the following only inside this isolated malware-analysis guest, from Administrator PowerShell:

Windows Security → Virus & threat protection → Manage settings → Tamper Protection → Off  All features (Can not type in the commandline, need to interact GUI)

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

Stop-Service wuauserv -Force -ErrorAction SilentlyContinue
Set-Service wuauserv -StartupType Disabled

Stop-Service BITS -Force -ErrorAction SilentlyContinue
Set-Service BITS -StartupType Disabled

schtasks /Change /TN "\Microsoft\Windows\Windows Defender\Windows Defender Scheduled Scan" /Disable 2>$null

Stop-Service WSearch -Force -ErrorAction SilentlyContinue
Set-Service WSearch -StartupType Disabled
```

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
Set-MpPreference -DisableBehaviorMonitoring $true
Set-MpPreference -DisableIOAVProtection $true
Set-MpPreference -DisableScriptScanning $true
Set-MpPreference -DisableBlockAtFirstSeen $true
Set-MpPreference -EnableNetworkProtection Disabled
```

> Warning: These settings reduce interference/noise in a disposable sandbox. They are not suitable for normal workstations.

## 14. Install the validated Windows Python runtime

This exact deployment failed when analyzer.py was launched under Python 3.8.10 x86: CAPE Agent accepted /execpy, status changed from running to failed, exit code 1, and no ResultServer connection occurred. Replacing the guest runtime with Python 3.9.13 x86 resolved the dynamic-analysis failure.

```powershell
New-Item -ItemType Directory -Force C:\CAPE-Setup | Out-Null
Invoke-WebRequest -UseBasicParsing -Uri "https://www.python.org/ftp/python/3.9.13/python-3.9.13.exe" -OutFile "C:\CAPE-Setup\python-3.9.13-x86.exe"
Start-Process "C:\CAPE-Setup\python-3.9.13-x86.exe" -ArgumentList "/quiet InstallAllUsers=1 PrependPath=1 Include_test=0" -Wait
```

Validate the exact executable rather than trusting PATH:

```powershell
Test-Path "C:\Program Files (x86)\Python39-32\python.exe"
& "C:\Program Files (x86)\Python39-32\python.exe" --version
& "C:\Program Files (x86)\Python39-32\python.exe" -c "import struct; print(struct.calcsize('P') * 8)"
where.exe python
```

Expected validated output: Python 3.9.13 and pointer size 32.

> Validated: Current upstream CAPEv2 documentation says the Windows agent is tested with Python 3.7/3.8 x86. This runbook records the observed compatibility of this specific host/repository state: 3.9.13 x86 was the version that eliminated analyzer exit code 1.

## 15. Install CAPE Agent and make it persistent

### 15.1 Download the agent

```powershell
New-Item -ItemType Directory -Force C:\Agent | Out-Null
Invoke-WebRequest -UseBasicParsing -Uri "https://raw.githubusercontent.com/kevoreilly/CAPEv2/master/agent/agent.py" -OutFile "C:\Agent\agent.py"
Get-Item C:\Agent\agent.py
```

### 15.2 Start manually first

```powershell
& "C:\Program Files (x86)\Python39-32\python.exe" C:\Agent\agent.py
```

In a second PowerShell:

```powershell
curl.exe http://127.0.0.1:8000
```

From Ubuntu host:

```powershell
curl http://192.168.122.100:8000
```

### 15.3 Create Scheduled Task - one command at a time

```powershell
Unregister-ScheduledTask -TaskName "CAPE Agent" -Confirm:$false
```

```powershell
$Action = New-ScheduledTaskAction -Execute "C:\Program Files (x86)\Python39-32\python.exe" -Argument "C:\Agent\agent.py" -WorkingDirectory "C:\Agent"
```

```powershell
$Trigger = New-ScheduledTaskTrigger -AtLogOn
```

Get the exact local account string:

```powershell
whoami
```

Use the returned value literally. Example only:

```powershell
$Principal = New-ScheduledTaskPrincipal -UserId "DESKTOP-XXXX\Cape" -LogonType Interactive -RunLevel Highest
```

```powershell
Register-ScheduledTask -TaskName "CAPE Agent" -Action $Action -Trigger $Trigger -Principal $Principal -Description "CAPEv2 Agent"
```

```powershell
Start-ScheduledTask -TaskName "CAPE Agent"
```

### 15.4 Validate task runtime

```powershell
Get-ScheduledTask -TaskName "CAPE Agent" | Select TaskName,State
Get-ScheduledTaskInfo -TaskName "CAPE Agent" | Select LastRunTime,LastTaskResult
(Get-ScheduledTask -TaskName "CAPE Agent").Actions
netstat -ano | findstr :8000
curl.exe http://127.0.0.1:8000
```

> Important: Task Scheduler 'Ready' means waiting for a trigger. It is not proof that agent.py is listening. Validate TCP/8000.

> [!NOTE]
> Remember to install Pillow to enable the screenshot feature on the guest.

## 16. Validate CAPE control channels before snapshot

### 16.1 Agent channel

```bash
curl http://192.168.122.100:8000
```

### 16.2 ResultServer channel

On Ubuntu:

```powershell
ss -lntp | grep ':2042'
```

On Windows:

```powershell
Test-NetConnection 192.168.122.1 -Port 2042
```

Expected:

```powershell
TcpTestSucceeded : True
```

### 16.3 Reboot/logon persistence check

Reboot the Windows guest once, log in, then repeat:

```powershell
netstat -ano | findstr :8000
curl.exe http://127.0.0.1:8000
```

If this fails, fix Scheduled Task before snapshotting.

## 17. Create the known-good snapshot

Create the snapshot while Windows is running, logged in, network is ready, and CAPE Agent is already listening.

```bash
virsh -c qemu:///system snapshot-create-as cuckoo1 clean-realistic "Clean Windows baseline - Python 3.9"
virsh -c qemu:///system snapshot-list cuckoo1
virsh -c qemu:///system snapshot-info cuckoo1 clean-realistic
```

After snapshot creation, shut down the current VM instance:

```bash
virsh -c qemu:///system shutdown cuckoo1
watch -n 1 'virsh -c qemu:///system domstate cuckoo1'
```

Stop watch when the state becomes shut off.

### 17.1 Pin CAPE to the snapshot

```bash
sudo -u cape nano /opt/CAPEv2/conf/kvm.conf
```

Add under [cuckoo1]:

```bash
snapshot = clean-realistic
```

```bash
grep -nA15 '^\[cuckoo1\]' /opt/CAPEv2/conf/kvm.conf
sudo systemctl restart cape
journalctl -u cape -n 50 --no-pager
```

## 18. End-to-end validation

### 18.1 Create a benign submission

```bash
echo "hello cape" > /tmp/cape-test.txt
cd /opt/CAPEv2
sudo -u cape -H /etc/poetry/bin/poetry run python utils/submit.py /tmp/cape-test.txt
```

### 18.2 Follow runtime logs

```bash
journalctl -u cape -f
```

A healthy task should pass through:

```bash
Starting analysis on guest
Guest is running CAPE Agent
Uploading script files to guest
```

and continue into analysis rather than immediately reporting Analysis failed.

### 18.3 Submit an authorized malware sample

Keep samples outside normal user home directories. Example storage:

```bash
sudo mkdir -p /srv/malware-samples/{submit,archive}
sudo chown root:cape /srv/malware-samples /srv/malware-samples/submit
sudo chmod 750 /srv/malware-samples /srv/malware-samples/submit
```

Submit through CAPE only; never execute the sample on the Ubuntu host:

```bash
cd /opt/CAPEv2
SAMPLE="/srv/malware-samples/submit/sample.ps1"   # replace with the actual filename
sudo -u cape -H /etc/poetry/bin/poetry run python utils/submit.py --route internet "$SAMPLE"
```

## 19. Known failure modes and exact checks

| Symptom | What it means | Check / fix |
| --- | --- | --- |
| cape.service: Unable to bind ResultServer on 192.168.1.1:2042 | Stale installer default IP. | Set [resultserver] ip=192.168.122.1; verify virbr0; restart cape. |
| Error: adding task to database | Task insert failed before VM startup. | Test PostgreSQL credential, \dt, tasks table, CAPE DSN. |
| Agent 0.22 responds, then analyzer exit code 1 | Agent works but analyzer startup failed in guest. | On this build, Python 3.9.13 x86 fixed the failure. |
| No mapping between account names and security IDs | Invalid Scheduled Task principal. | Run whoami; recreate Principal with exact account string. |
| Scheduled Task shows Ready, no TCP/8000 | Task is waiting or failed immediately. | Start task manually; inspect LastTaskResult and Actions. |
| Invoke-WebRequest requires IE engine | Legacy Windows PowerShell parser. | Use -UseBasicParsing. |
| python --version shows 3.8 or WindowsApps | PATH/alias does not point to validated runtime. | Use C:\Program Files (x86)\Python39-32\python.exe explicitly. |
| CAPE migration fails because rooter unavailable | Django/CAPE startup path expects rooter. | Start cape-rooter first; verify /tmp/cuckoo-rooter; rerun migration. |

### 19.1 Packet-level capture for early analyzer failure

```bash
sudo tcpdump -ni virbr0 -s0 -w /root/cape-debug.pcap 'host 192.168.122.100 and (tcp port 8000 or tcp port 2042)'
```

Submit one task, stop capture after failure, then inspect:

```bash
sudo tcpdump -nn -A -r /root/cape-debug.pcap | grep -E 'execpy|status|2042'
```

The previously observed failing signature was /execpy accepted, status running, then status failed / exit code 1, with no TCP/2042 connection.

## 20. Host reboot acceptance and runtime/cron verification

A deployment is not complete until it survives a host reboot.

```bash
sudo reboot
```

After reconnecting:

```bash
systemctl is-active cron postgresql cape-rooter cape cape-processor cape-web
systemctl is-enabled cron postgresql cape-rooter cape cape-processor cape-web
systemctl status mongod --no-pager 2>/dev/null || systemctl status mongodb --no-pager 2>/dev/null || true
systemctl status libvirtd --no-pager 2>/dev/null || systemctl status virtqemud --no-pager 2>/dev/null || true

ip -br addr show virbr0
virsh -c qemu:///system net-info default
virsh -c qemu:///system list --all
virsh -c qemu:///system snapshot-list cuckoo1

ss -lntp | grep 2042
sudo crontab -l 2>/dev/null || true
sudo -u cape crontab -l 2>/dev/null || true
sudo grep -RniE 'cape|cuckoo|mongodb|mongod|suricata' /etc/cron.d /var/spool/cron/crontabs 2>/dev/null || true
tail -n 100 /var/log/cape-runtime-check.log 2>/dev/null || true
```

> Validated: At idle, cuckoo1 may be shut off. That is correct. CAPE restores/starts the guest for analysis; the important checks are that libvirt network, snapshot, CAPE services, ResultServer and cron runtime are healthy.

## 21. Save a known-good deployment bundle

```bash
sudo mkdir -p /root/cape-known-good
sudo cp -a /opt/CAPEv2/conf /root/cape-known-good/
sudo cp -a /lib/systemd/system/cape*.service /root/cape-known-good/ 2>/dev/null || true
sudo cp -a /etc/cron.d/cape-runtime-check /root/cape-known-good/ 2>/dev/null || true
sudo cp -a /usr/local/sbin/cape-runtime-check.sh /root/cape-known-good/ 2>/dev/null || true
sudo cp -a /root/kvm-qemu-install.log /root/cape-known-good/ 2>/dev/null || true
sudo cp -a /root/cape-base-install.log /root/cape-known-good/ 2>/dev/null || true
sudo tar -C /root -czf /root/cape-known-good-$(date +%F).tar.gz cape-known-good
```

Also record the repository revision:

```bash
cd /opt/CAPEv2
git rev-parse HEAD
git status --short
```

## 22. Final acceptance checklist

☐ Nested virtualization is available; KVM module is loaded.

☐ qemu:///system works for root and for the cape service account.

☐ virbr0 is UP at 192.168.122.1/24 and libvirt default network autostarts.

☐ cape user was created by cape2.sh before any sudo -u cape / chown cape:cape steps.

☐ Poetry environment installs successfully as cape.

☐ PostgreSQL role/database cape authenticate using the configured password.

☐ MongoDB is active on localhost:27017.

☐ cape-rooter, cape, cape-processor, cape-web are active and enabled.

☐ ResultServer listens on 192.168.122.1:2042.

☐ cron is active/enabled and installer-created cron entries have been reviewed.

☐ Optional cape-runtime-check cron job writes healthy state to /var/log/cape-runtime-check.log.

☐ cuckoo1 exists, uses the default libvirt network, and has IP 192.168.122.100.

☐ Windows guest uses Python 3.9.13 x86 for CAPE Agent.

☐ Scheduled Task starts C:\Agent\agent.py and TCP/8000 is reachable from host.

☐ Guest can connect to 192.168.122.1:2042.

☐ Snapshot clean-realistic exists and is selected in kvm.conf.

☐ Benign submission completes beyond analyzer startup.

☐ Authorized malware submission runs inside CAPE rather than on the host.

☐ Host reboot acceptance test passes.

## 23. Quick Commands

Health check:
```bash

systemctl --no-pager --full status cape-rooter cape cape-processor cape-web postgresql cron

systemctl is-active cape-rooter cape cape-processor cape-web postgresql cron

virsh -c qemu:///system net-list --all

systemctl status mongod --no-pager

systemctl status libvirtd --no-pager

ip -br addr show virbr0

ss -lntp | grep -E '2042|27017'

```

Quick CAPE log:
```bash

journalctl -u cape -f

journalctl -u cape-rooter -n 50 --no-pager

journalctl -u cape-processor -n 50 --no-pager

journalctl -u cape-web -n 50 --no-pager
```

Restart CAPE
```bash

systemctl restart cape cape-rooter cape-processor cape-web
```

Submit Malware:
```bash

sudo -u cape -H /etc/poetry/bin/poetry run python utils/submit.py --route internet /srv/malware-samples/submit/3c2d602ddfa3fb7fdc5aedc735a07cab7ee70a77e739070538d03dca17cd579d.ps1
```

Malware files will be located at /srv/malware-samples, which have 2 folders archive and submit

Check Task in DB
```bash

sudo -u postgres psql -d cape -c "SELECT id,status,target,added_on FROM tasks ORDER BY id DESC LIMIT 10;"
```

Check VM State
```bash

virsh -c qemu:///system domstate cuckoo1

virsh -c qemu:///system list --all

virsh -c qemu:///system snapshot-list cuckoo1

virsh -c qemu:///system snapshot-info cuckoo1 clean-realistic

tail -n 50 /var/log/cape-runtime-check.log
```

Start and Destroy VM
```bash

virsh -c qemu:///system start cuckoo1

virsh -c qemu:///system destroy cuckoo1
```

Check snapshot
```bash

virsh -c qemu:///system snapshot-list cuckoo1
```

Clean old analysis result
```bash

du -sh /opt/CAPEv2/storage/analyses

ls -lah /opt/CAPEv2/storage/analyses

rm -rf /opt/CAPEv2/storage/analyses/15
```

Cleaner.py:
```bash

systemctl stop cape cape-processor

cd /opt/CAPEv2

sudo -u cape -H /etc/poetry/bin/poetry run python utils/cleaners.py --clean (không có xóa trong MongoDB)

sudo -u cape -H /etc/poetry/bin/poetry run python utils/cleaners.py --clean --delete-mongo (sạch analysis report

systemctl start cape cape-processor
```

Forward port:
```bash

ssh -L 8000:127.0.0.1:8000 username@172.16.24.42 (run on host)
```

## 24. Source notes and deployment-specific decisions

Upstream references used to verify installer/service behavior:

https://github.com/kevoreilly/CAPEv2/blob/master/README.md

https://github.com/kevoreilly/CAPEv2/blob/master/docs/book/src/installation/host/installation.rst

https://github.com/kevoreilly/CAPEv2/blob/master/installer/cape2.sh

https://github.com/kevoreilly/CAPEv2/blob/master/docs/book/src/usage/web.rst

Deployment-specific decisions captured from the successful lab build:

Ubuntu 22.04.5 retained because it is the environment actually validated here, even though newer upstream guidance prefers Ubuntu 24.04 for fresh deployments.

The cape2.sh command syntax follows the installer help printed on the target host: <command> <iface_ip> [options].

Guest Python is pinned to 3.9.13 x86 because that version resolved the observed analyzer exit-code-1 failure on this repository/guest combination.

The runtime health cron job is an operational addition for this environment, not an upstream CAPEv2 requirement.

END OF RUNBOOK

---

## Current-lab notes added after validation

- `cuckoo1` is a **Windows 10 x64** guest. Keep `arch = x64` and tags such as `win10,x64` in `kvm.conf`. The CAPE Agent can still run under **Python 3.9.13 x86**; guest OS architecture and Python runtime architecture are separate concerns.
- The working snapshot for this lab is `clean-realistic`.
- Guest-side screenshots require Pillow in the validated Python runtime.
- For CAPE behavior processing on this deployment, `ram_mmap = no` avoided the observed `memoryview` parsing failure.
- If BinGraph is enabled, ensure its Python package is installed and keep CAPE binary copies long enough for on-demand graph generation.
- If MITRE ATT&CK reporting is enabled, install `pyattck` and configure `[mitre] enabled = yes` in `reporting.conf`.
- The current snapshot references the Windows/VirtIO ISO media. Keep those ISO files in place unless a new snapshot is deliberately created after ejecting the media.

## Upstream references

- <https://github.com/kevoreilly/CAPEv2>
- <https://github.com/kevoreilly/CAPEv2/blob/master/docs/book/src/installation/host/installation.rst>
- <https://github.com/kevoreilly/CAPEv2/blob/master/installer/cape2.sh>
- <https://github.com/kevoreilly/CAPEv2/blob/master/docs/book/src/usage/web.rst>
