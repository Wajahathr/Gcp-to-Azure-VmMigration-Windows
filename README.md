GCP to Azure Windows VM Migration Guide
This guide outlines the step-by-step process for manually migrating a Windows Virtual Machine from Google Cloud Platform (GCP) to Microsoft Azure.

Since standard migration tools may occasionally fail due to OS agent conflicts or disk format incompatibilities, this procedure handles disk conversion (Dynamic to Fixed VHD) and offline OS preparation to ensure a successful boot in Azure.

Prerequisites
Active subscriptions for both GCP and Microsoft Azure.

Google Cloud SDK (gcloud) and Azure CLI (az) installed and authenticated.

Sufficient permissions to create storage buckets, storage accounts, VMs, and managed disks.

Step 1: Export the VM Image from GCP
First, we need to stop the source VM, create an image of its disk, and export it to a Google Cloud Storage (GCS) bucket as a .vhd file.

Bash
# 1. Stop the target VM in GCP
gcloud compute instances stop attendance-asani --zone=us-central1-a

# 2. Create an image from the instance's OS disk
gcloud compute images create attendance-img \
  --source-disk=instance-20241016-174718 \
  --source-disk-zone=us-central1-a

# 3. Restart the VM to minimize downtime (optional)
gcloud compute instances start attendance-asani --zone=us-central1-a

# 4. Create a GCS bucket for the export
gcloud storage buckets create gs://attendance-migration-2026 --location=us-central1

# 5. Export the image to the GCS bucket in VPC (VHD) format
gcloud compute images export \
  --destination-uri=gs://attendance-migration-2026/attendance.vhd \
  --image=attendance-img \
  --export-format=vpc

# 6. Make the VHD file temporarily readable for the Azure transfer
gsutil acl ch -u AllUsers:R gs://attendance-migration-2026/attendance.vhd
Step 2: Transfer VHD to Azure Blob Storage
Directly copy the VHD file from Google Cloud Storage to an Azure Storage Account container.

Bash
# 1. Start the blob copy operation
az storage blob copy start \
  --account-name migrationtoazure26part1 \
  --destination-container vhd-files \
  --destination-blob attendance.vhd \
  --source-uri "https://storage.googleapis.com/attendance-migration-2026/attendance.vhd" \
  --auth-mode login

# 2. Monitor the copy status (Wait until the status is "success")
az storage blob show \
  --account-name migrationtoazure26part1 \
  --container-name vhd-files \
  --name attendance.vhd \
  --query "properties.copy.status" \
  --auth-mode login \
  --output tsv 
Step 3: Convert Dynamic VHD to Fixed VHD
GCP exports VHDs in a "Dynamic" format, but Azure requires a "Fixed" format aligned to 1MB. We will deploy a temporary Linux VM in Azure to perform this conversion.

3.1 Setup Converter VM & Download VHD
Bash
# Deploy an Ubuntu VM for conversion
az vm create \
  --resource-group asani \
  --name converter-vm \
  --location uaenorth \
  --image Ubuntu2204 \
  --admin-username wajahat \
  --os-disk-size-gb 100 \
  --size Standard_D2s_v3 \
  --generate-ssh-keys

# Generate a SAS token to download/upload the VHD safely
ACC="migrationtoazure26part1"
RG="asani"
CONT="vhd-files"
KEY=$(az storage account keys list -g $RG -n $ACC --query '[0].value' -o tsv)
EXP=$(date -u -d "1 day" '+%Y-%m-%dT%H:%MZ')
SAS=$(az storage container generate-sas --account-name $ACC --account-key $KEY --name $CONT --permissions acdrw --expiry $EXP -o tsv)
echo "?$SAS"

# Get the public IP of the Ubuntu VM and SSH into it
az vm show -d -g asani -n converter-vm --query publicIps -o tsv
ssh wajahat@<UBUNTU_VM_IP>

# Inside the Ubuntu VM: Download the VHD using AzCopy
azcopy copy "https://migrationtoazure26part1.blob.core.windows.net/vhd-files/attendance.vhd?<SAS_TOKEN>" ./attendance.vhd
3.2 Perform Disk Conversion
Run these commands inside the Ubuntu VM:

Bash
# Convert dynamic VHD to RAW format
nohup qemu-img convert -p -f vpc -O raw ./attendance.vhd ./attendance.raw > conversion.log 2>&1 &
tail -f conversion.log

# Resize the RAW image to meet Azure's sizing requirements (e.g., 52GB)
qemu-img resize -f raw ./attendance.raw 53248M

# Convert RAW back to Azure-compatible Fixed VHD
qemu-img convert -p -f raw -O vpc -o subformat=fixed,force_size=on ./attendance.raw ./attendance-final.vhd

# Upload the final Fixed VHD back to Azure Blob Storage as a PageBlob
azcopy copy "./attendance-final.vhd" "https://migrationtoazure26part1.blob.core.windows.net/vhd-files/attendance-final.vhd?<SAS_TOKEN>" --blob-type PageBlob

exit
Step 4: Prepare the OS using a Windows Helper VM
GCP's native agents will cause boot and networking issues in Azure. We must attach the OS disk to a temporary Windows VM to disable GCP agents and enable Remote Desktop.

4.1 Create Disk and Helper VM
Bash
# Create an Azure Managed Disk from the Fixed VHD
az disk create \
  --resource-group asani \
  --name attendance-disk \
  --source "https://migrationtoazure26part1.blob.core.windows.net/vhd-files/attendance-final.vhd" \
  --location uaenorth \
  --os-type windows \
  --hyper-v-generation V2

# Create a temporary Windows Server Helper VM
az vm create \
  --resource-group asani \
  --name helper-win-vm \
  --image Win2022Datacenter \
  --size Standard_B2s \
  --admin-username wajahat \
  --admin-password "TempAsani2026@" \
  --location uaenorth

# Attach the migrated disk to the helper VM
az vm disk attach \
  --resource-group asani \
  --vm-name helper-win-vm \
  --name attendance-disk
4.2 Offline Registry Edits (Inside Helper VM)
Log into the helper-win-vm via RDP.

Open Disk Management and bring the attached disk Online. Note the drive letter (e.g., F:).

Open Command Prompt (Run as Administrator) and execute the following to disable GCP agents, turn off the firewall, and allow RDP:

DOS
:: Load the offline system registry
reg load HKLM\RESCUE_SYS F:\Windows\System32\config\SYSTEM

:: Disable GCP Agents
reg add "HKLM\RESCUE_SYS\ControlSet001\Services\GCEAgent" /v Start /t REG_DWORD /d 4 /f
reg add "HKLM\RESCUE_SYS\ControlSet001\Services\google_osconfig_agent" /v Start /t REG_DWORD /d 4 /f
reg add "HKLM\RESCUE_SYS\ControlSet001\Services\google_compute_engine" /v Start /t REG_DWORD /d 4 /f

:: Allow RDP and Disable Firewalls
reg add "HKLM\RESCUE_SYS\ControlSet001\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
reg add "HKLM\RESCUE_SYS\ControlSet001\Services\SharedAccess\Parameters\FirewallPolicy\DomainProfile" /v EnableFirewall /t REG_DWORD /d 0 /f
reg add "HKLM\RESCUE_SYS\ControlSet001\Services\SharedAccess\Parameters\FirewallPolicy\PublicProfile" /v EnableFirewall /t REG_DWORD /d 0 /f
reg add "HKLM\RESCUE_SYS\ControlSet001\Services\SharedAccess\Parameters\FirewallPolicy\StandardProfile" /v EnableFirewall /t REG_DWORD /d 0 /f

reg unload HKLM\RESCUE_SYS
4.3 Configure Auto-Admin Script (Optional Backdoor)
To ensure accessibility, inject a batch script that creates a local admin on first boot.

DOS
:: Replace Utilman with CMD for accessibility screen bypass
takeown /f F:\Windows\System32\Utilman.exe
icacls F:\Windows\System32\Utilman.exe /grant administrators:F
ren F:\Windows\System32\Utilman.exe Utilman.exe.bak
copy F:\Windows\System32\cmd.exe F:\Windows\System32\Utilman.exe

:: Create user creation batch file
echo net user ************ "@!" /add > F:\asani_user.bat
echo net localgroup administrators ******** /add >> F:\asani_user.bat
echo net localgroup "Remote Desktop Users" ******** /add >> F:\asani_user.bat
echo reg add "HKLM\SYSTEM\Setup" /v SetupType /t REG_DWORD /d 0 /f >> F:\asani_user.bat
echo reg add "HKLM\SYSTEM\Setup" /v CmdLine /t REG_SZ /d "" /f >> F:\asani_user.bat

:: Trigger batch file on boot
reg load HKLM\RESCUE_SYS F:\Windows\System32\config\SYSTEM
reg add "HKLM\RESCUE_SYS\Setup" /v SetupType /t REG_DWORD /d 2 /f
reg add "HKLM\RESCUE_SYS\Setup" /v CmdLine /t REG_SZ /d "cmd.exe /c C:\asani_user.bat" /f
reg unload HKLM\RESCUE_SYS
Once done, set the disk to Offline in Disk Management before detaching.

Step 5: Deploy Final Azure VM
Detach the modified disk from the Helper VM and use it to create the final production VM.

Bash
# Detach the disk from the helper VM
az vm disk detach --resource-group asani --vm-name helper-win-vm --name attendance-disk

# Create the final VM using the prepared disk
az vm create \
  --resource-group asani \
  --name asani-attendance-vm \
  --attach-os-disk attendance-disk \
  --os-type windows \
  --location uaenorth \
  --size Standard_B2ms

# Open RDP Port
az vm open-port --resource-group asani --name asani-attendance-vm --port 3389 --priority 100


# Google/GCP services ko dhoond kar stop aur disable karna
$gcpServices = Get-Service | Where-Object {$_.Name -like "*google*" -or $_.Name -like "*gce*"}
foreach ($service in $gcpServices) {
    Stop-Service $service.Name -Force -ErrorAction SilentlyContinue
    Set-Service $service.Name -StartupType Disabled
    sc.exe delete $service.Name
}

# GCP ke folders ko uda dena (Agar permission ho)
Remove-Item -Path "C:\Program Files\Google\Compute Engine" -Recurse -Force -ErrorAction SilentlyContinue

# Boot configuration mein Serial Console on karna
bcdedit /ems {current} on
bcdedit /emssettings EMSPORT:1 EMSBAUDRATE:115200

# Windows Boot Manager mein console enable karna
bcdedit /set {bootmgr} displaybootmenu yes
bcdedit /set {bootmgr} timeout 5

# Agent download karna
Invoke-WebRequest -Uri "https://go.microsoft.com/fwlink/?LinkID=394789" -OutFile "$env:TEMP\AzureAgent.msi"

# Silent installation (Bina kisi pop-up ke install hoga)
Start-Process msiexec.exe -ArgumentList "/i `"$env:TEMP\AzureAgent.msi`" /quiet /qn /norestart" -Wait

# Confirm karna ke agent chal raha hai
Get-Service -Name "RdAgent", "WindowsAzureGuestAgent"

$size = (Get-PartitionSupportedSize -DriveLetter C).SizeMax
Resize-Partition -DriveLetter C -Size $size

# Hack file delete karein
Remove-Item -Path "C:\asani_user.bat" -Force

# VM ko restart karein
Restart-Computer -Force
