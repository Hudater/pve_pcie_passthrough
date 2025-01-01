# Link: https://forum.proxmox.com/threads/simple-working-gpu-passthrough-on-uptodate-pve-and-amd-hardware.145462/



Apr 20, 2024
#1
Hello Guys,
I wanted to share my steps to my perfectly working GPU Passthrough setup after a lot of debugging, testing and searching for the best, working method. Maybe it helps someone. First my system specs:
CPU: AMD Ryzen 7800X3D (Should Probably Work with all Ryzen 7000 APUs)
GPU: Sapphire AMD 7900 XTX Pulse
RAM: 2x 16GB G.SKILL DDR5 6000 CL30 (Not that Important for this Passthrough, just for information)
Mainboard: Gigabyte B650M K
SSD: 2x Western Digital WD Black SN850X 2TB (Not that Important for this Passthrough, just for Information)
Current tested PVE version: 8.2.2

BIOS Settings
- CSM (Boot) has to be enabled. Otherwise you will suffer the reset bug.
- IOMMU enabled.
- SR-IOV enabled.

(I also enabled Above 4G Decoding AND Resizable Bar Support, which works perfectly in my case. This is not neccessary for the passthrough, can get you more performance though.)

Enable Modules
You need to enable the required Modules in Proxmox. Therefore edit the configuration with:
Code:
nano /etc/modules-load.d/modules.conf

add these three lines:

Code:
vfio
vfio_iommu_type1
vfio_pci

Save and execute following command:

Code:
update-initramfs -u -k all

Download GPU ROM
You need the fitting ROM file for you're GPU. Without it I experienced the reset bug. In my case (With my GPU) I executed following commands to get the fitting ROM file to the correct folder. Search at techpowerup, if you're lucky they got a ROM file for you're GPU and you can just edit the following lines to mach your's:

Code:
cd /usr/share/kvm
wget https://www.techpowerup.com/vgabios/254079/Sapphire.RX7900XTX.24576.221129.rom
mv Sapphire.RX7900XTX.24576.221129.rom 7900xtx.rom

Restart you're PVE node!

That's it for preparations. Now you can create a vm (I testet it with Fedora 40 Beta and Windows 11). Works perfectly with following Settings:

- CPU Type: Host
- Display: none (After Installation of the System and Display Driver, for Installation I used VirtIO)
- Machine: q35
- BIOS: UEFI
- Controller: VirtIO SCSI Single
- Hard Disk: Discard, IO Thread, SSD Emulation
- Network: VirtIO
- USB Device Passthrough (Mouse/Keyboard or other devices you need to use which are plugged in via USB)
- PCI Passthrough (Raw Device, 0000:XX:XX.X, all functions, ROM Bar, PCI Express, primary GPU)

If you have Problems, add the gpu passthrough only after full windows Installation. Don't forget to install the Qemu-Guest-Tools/Drivers and enable the Guest Agent Option unter VM Options.

VERY IMPORTANT:
After you have createt you're VM AND enabled the PCI Passthrough, you need to add the ROM file to the settings. Sadly there isn't any GUI feature for it, you need to do it manually. Inside the shell edit you're vm config file (NODE is you're machine, default pve, xxx stands for you're VM ID):

Code:
nano /etc/pve/nodes/NODE/qemu-server/XXX.conf

Find the line with you're GPU passthrough in it, and add you're romfile to it. In my case it looks like this:

Code:
hostpci0: 0000:03:00,pcie=1,x-vga=1,romfile=7900xtx.rom

That's it. Just save it and you can start you're VM. Works absolutly Perfectly in my case. I testet it with the integrated Benchmark of Horizon Zero Dawn, on 4k and Max Settings, between Native Windows and the VM I got a deviation of under 1%, so practicly no performance loss. My AMD drivers work without problems, they even recognize Rezisable Bar Support / AMD SmartAccess Memory. I also passed-through one of my SSDs for my games, so that my Windows 11 installation also can use DirectStorage.

OPTIONAL: PVE GRUB Config
Works perfectly without it, but it may get you more performance or resolve some issues in some usecases:
1. In the PVE Shell type: nano /etc/default/grub
2. change the GRUB_CMDLINE_LINUX_DEFAULT="quiet" line to GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt"
3. Close with CTRL + X, and save with y
4. execute: update-grub