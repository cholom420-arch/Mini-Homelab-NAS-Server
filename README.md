# Mini-Homelab-NAS-Server
Mini Homelab NAS/Server

NAS/Server

Hardware:

CASE: Jonsbo N1

CPU: Intel Core i7 7700T

CPU COOLER: Noctua NH-L9i chromax.black

MOTHERBOARD: Asrock H270M-ITX/ac

NETWORK CARD: TP-Link TX201 2.5Gbps LAN Card PCI-E

MEMORY: CORSAIR DDR4-3200MHz VENGEANCE LPX Series 32GB [16GB×2]

0S STORGE Proxmox and Open Media Vault: Samsung 256GB Nvme

STORAGE Proxmox Backup: 2tb Seagate

STORAGE OMV DATA (MergerFS):2x Western Digital Red WD60EFPX 6tb

STORAGE (Snapraid Parity Drive): Western Digital Red WD80EFAX 8tb

PSU: Silverstone SST-ST45SF-G 450w SFX



Phase 1: Hardware Assembly & BIOS Configuration
1. Assembly Notes (Jonsbo N1)

Airflow: Ensure the Noctua NH-L9i fan is pushing air towards the CPU. The Jonsbo N1 has limited airflow; ensure the rear 140mm fan is set to exhaust.

SATA Cabling: Connect the 5 SATA cables from the Jonsbo backplane to your ASRock H270M-ITX/ac. Connect your 2.5Gbps LAN card to the single PCIe slot.

Drive Placement:

Slots 1-2: WD Red 6TB (Data)

Slot 3: WD Red 8TB (Parity - Must be the largest drive)

Slot 4: Seagate 2TB (Backups)

M.2 Slot (Motherboard): Samsung 256GB NVMe (OS)

2. BIOS/UEFI Settings Before installing software, boot into BIOS (F2 or Del):

Advanced > CPU Configuration: Enable Intel Virtualization Technology (VT-x) and VT-d.

Boot: Set the NVMe drive as the primary boot option.

Power: Set "Restore on AC/Power Loss" to Power On (crucial for a headless server).

Phase 2: Proxmox VE 8 Installation
1. Install Proxmox

Download the Proxmox VE 8.x ISO.

Flash to USB (using Rufus or Etcher).

Boot from USB and follow the wizard.

Target Harddisk: Select Samsung 256GB NVMe.

Network: The installer should detect your TP-Link TX201 (RTL8125). If unsure, plug the cable into the motherboard's 1GbE port for the install, then switch later.

2. Post-Install Configuration (CLI) Access your Proxmox Shell (either directly or via the Web UI > Node > Shell).

Update Repositories (Remove Enterprise, Add No-Subscription)

Bash
# Disable Enterprise Repo
sed -i 's/^deb/#deb/g' /etc/apt/sources.list.d/pve-enterprise.list

# Add No-Subscription Repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Update System
apt update && apt dist-upgrade -y
Initialize the Backup Drive (2TB Seagate) Assuming you want this drive accessible by Proxmox to store VM backups.

Go to Datacenter > Storage > Add > Directory.

ID: local-backups.

Select the partition on your Seagate drive (You may need to Wipe Disk in the Disks menu first).

Content: VZDump backup file, ISO image.

Phase 3: OMV Virtual Machine Setup & Disk Passthrough
We will not use a virtual disk for your data. We will pass the physical WD drives directly to OMV so it can manage SMART data and sleep states.

1. Create the VM

General: Name: OMV-NAS, VM ID: 100.

OS: Do not use media yet (select "Do not use any media").

System: Machine: q35, BIOS: OVMF (UEFI).

Disks: Delete the default scsi0 disk (we will add one later).

CPU: 4 Cores (Type: host).

Memory: 8192 MB (8GB is plenty).

Network: VirtIO (paravirtualized).

2. Pass Physical Disks to VM (CLI) Go to the Proxmox Shell. We need to identify your disks by their unique ID to prevent issues if cable orders change.

Bash
# List all disks by ID
ls -l /dev/disk/by-id/
Look for strings matching your WD drives (e.g., ata-WDC_WD60EFPX...).

Run the following commands to attach them to VM 100:

Bash
# Example commands (Replace [DISK_ID] with your actual IDs found above)

# Data Drive 1 (6TB)
qm set 100 -sata1 /dev/disk/by-id/ata-WDC_WD60EFPX_[SERIAL1]

# Data Drive 2 (6TB)
qm set 100 -sata2 /dev/disk/by-id/ata-WDC_WD60EFPX_[SERIAL2]

# Parity Drive (8TB)
qm set 100 -sata3 /dev/disk/by-id/ata-WDC_WD80EFAX_[SERIAL3]
3. Add Boot Drive & Install Media

Back in Proxmox UI > VM 100 > Hardware.

Add > Hard Disk: Storage: local-lvm, Size: 32G. (This is for the OMV OS).

Add > CD/DVD Drive: Select the OMV ISO you uploaded to local storage.

Options > Boot Order: Ensure the CD drive is enabled and first.

Phase 4: OpenMediaVault Installation
Start the VM and open the Console.

Follow the OMV installation steps (Language, Hostname, Root Password).

Partitioning: Select the 32GB Virtual Disk (do not install OS on the 6TB/8TB drives!).

Once installed, remove the ISO and reboot.

Find the IP address on the console screen and log in to the Web UI (admin / openmediavault).

Phase 5: MergerFS & SnapRAID Configuration
You need the OMV-Extras plugin to install MergerFS and SnapRAID easily.

1. Install OMV-Extras (SSH/CLI into OMV) SSH into your OMV VM (User: root):

Bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | bash
Refresh the OMV Web UI. Go to System > Plugins, search for and install:

openmediavault-snapraid

openmediavault-mergerfs

2. Mount Filesystems

Go to Storage > File Systems.

Mount your two 6TB drives and the 8TB drive. Format them as XFS or EXT4 (XFS is recommended for large Linux ISOs/media).

Tip: Label them clearly, e.g., DISK1, DISK2, PARITY1.

3. Configure MergerFS (The Data Pool) MergerFS pools your drives into one logical folder.

Go to Storage > mergerfs.

Click Create.

Name: pool.

Branches: Select only your Data Drives (DISK1, DISK2). DO NOT select the Parity drive.

Policy: Most free space (Fills the drive with the most space first).

Save and Apply.

4. Configure SnapRAID (The Protection) SnapRAID calculates parity so you can recover if a drive fails.

Go to Services > SnapRAID > Arrays.

Parity: Select your 8TB Drive (PARITY1).

Content: Select your Data Drives (DISK1, DISK2).

Note: Also recommended to put a "Content file" on the Boot Disk for redundancy.

Data: Select your Data Drives (DISK1, DISK2).

Save and Apply.

Click Sync to run the initial parity calculation.

Phase 6: Sharing Your Data
Create a Shared Folder:

Storage > Shared Folders.

Device: Select pool (The MergerFS mount).

Name: Media or Backups.

Enable SMB/CIFS:

Services > SMB/CIFS > Settings: Enable.

Services > SMB/CIFS > Shares: Add the Shared Folder you created above.
