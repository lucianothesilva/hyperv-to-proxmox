# hyperv-to-proxmox
How to migrate Hyper-v vhdx VM to Proxmox, for local storage or scsi storage.

Sourced from [broadband09](https://broadband9.co.uk/how-to-migrate-hyper-v-vhdx-vm-to-proxmox-qcow2/) with some changes for my environment.

## Requisites

| Name | Version |
|------|---------|
| <a name="Virtio Drivers"></a> [Virtio Drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.215-2/virtio-win.iso) | 0.1.215-2 |
| <a name="Qemu "></a> [Qemu ](https://cloudbase.it/qemu-img-windows/) |2.3.0 |
| <a name="FileZilla "></a> [FileZilla ](https://cloudbase.it/qemu-img-windows/](https://filezilla-project.org/download.php)) |3.67.1|

Steps:
1. Install Virtio Drivers

2. Export the machine from Hyperv

3. Convert the .vhdx to .raw

4. Move to local storage 

    1. Create a VM in Proxmox
  
    2. Create Folder for machine
  
    3. Copy the .raw to the vm folder
  
5. Move to iscsi storage

    1. Create a VM in Proxmox
  
    2. Copy the .raw to the vm disk
  


**Prepare the Hyper-v VM with Virtio Drivers**


This step is crucial otherwise when the disk is on Proxmox then it will not load due to the drivers for the scsi or ide controller not being there.

You will get blue screens on the VM Console and maybe even a no bootable device found error.

Attach the Hyper-v VM with the virtio drivers iso file that has been downloaded on the hyper-v server:

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/fab537b9-5a82-454b-a93c-1c48eb9aaee7)

Browse inside the Virtual Machine console to the attached Virtio DVD Drive:

Double click / Install “Virtio-win-gt-x64.msi” to Install Guest Tools, follow the steps and ensure all the options are selected for the drivers installation:

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/9f5140c2-c8ba-4880-a934-29c0b566b58e)

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/0aa97cd8-bdf1-40da-91a4-3d07f40e3218)

Press Next and allow the installation to finish.

Then install the correct msi inside the “guest-agent” folder:

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/29b242bc-7a93-4b2a-b3ad-5bf5c932aac4)


After this step you can shutdown the Virtual machine ready to convert and export the new Disk to use inside of Proxmox.

Exporting VHDx to .raw format using qemu-img.exe
Now time for the export to .raw

Browse to the Qemu-img application that has been downloaded via powershell admin then run the command, change your settings as desired. For us the command was:

`.\qemu-img.exe convert ‘D:\Hyperv\Virtual Hard Disks\mymachine.vhdx’ -O raw D:\Migrate\mymachine.raw -p`


Go into the Proxmox VM settings and change the OS Type to Windows (if you didn’t do it at the creation wizard) and enable the “QEMU Guest Agent” setting:

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/17f6dc4e-89cf-4fca-9118-a3b4d328ff15)


# 2 types of storage migration

Choose the one for your scenario.

## **Migrate to pve local storage:**

Create a Virtual Machine on Proxmox, if the machine was set with UEFI make sure to set the BIOS to OVMF(UEFI) and add a EFI storage, also check the Qemu Agent checkbox.

![image](https://github.com/user-attachments/assets/d013942b-45dd-4138-81fd-d2e2d5d31c3f)

In the proxmox wizard when creating the VM it forces you to create a virtual machine disk, just set the size to 1GB, then after it’s created, go into it and “Detach” and “Remove” that disk.

Go into the Proxmox VM settings and change the OS Type to Windows (if you didn’t do it at the creation wizard) and enable the “QEMU Guest Agent” setting:

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/17f6dc4e-89cf-4fca-9118-a3b4d328ff15)

The new Virtual Machine has been assigned a new VM ID which in our case is 103.
So we want to place this disk in the directory relating to this VM folder.
The location will be `/var/lib/vz/images/`
To do this we need to create the folder for the machine with id 103 and then give it the correct permissions:

`mkdir /var/lib/vz/images/103`

`chmod 740 /var/lib/vz/images/103`

After this is done transfer the .raw file to this folder via Filezilla, using the Proxmox credentials.

Once the transfer is finished go back into the pve shell and run the following command:

`qm set 103 -scsi0 local:103/mymachine.raw`


## **Migrate to iscsi storage:**

Create a new VM on Proxmox, if the machine you're migrating was set with UEFI, make sure to set the BIOS to OVMF(UEFI) and add a EFI storage, also check the Qemu Agent checkbox.

![image](https://github.com/user-attachments/assets/d013942b-45dd-4138-81fd-d2e2d5d31c3f)

Go into the Proxmox VM settings and change the OS Type to Windows (if you didn’t do it at the creation wizard) and enable the “QEMU Guest Agent” setting:

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/17f6dc4e-89cf-4fca-9118-a3b4d328ff15)


In the Proxmox wizard when creating the virtual machine disk, just set the size to the same as the .raw file.

The new Virtual Machine has been assigned a new VM id which in our case is 103.

So we must copy the .raw file to the volume of the machine, to do it on the Windows Server run:

`scp .\mymachine.raw root@192.168.0.xxx:/dev/iscsi-vg/vm-103-disk-1`

Start the machine.

**IF BSOD**

Attach virtio ISO in the machine CD/DVD drive.

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/74f0c53e-02bf-4073-a15c-aedd554c275d)


Boot to revcovery mode (and if it doesnt attach a Windows ISO and boot from it), open cmd and run 

`diskpart` > `list disk`

If no disk is available:

Exit diskpart and run 

`drvload D:\vioscsi\2k12r2\amd64\vioscsi.inf` (change the OS to yours)

When I now open **diskpart** again and run `list volume` I am able to see that the letter “C” has been assigned to the operating system installation, if not run on diskpart:

`select volume 1` (or the one of your drive)

`assign letter=C`

`list volume`

To inall the drivers run:

Run `dism /image:C:\ /add-driver /Driver:D:\ /recurse`

C: is my letter for the Windows drive.

D: is my letter for the virtio drivers image disc.

Boot up the VM, should be running smoothly.
