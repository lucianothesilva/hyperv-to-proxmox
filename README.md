# hyperv-to-proxmox
How to migrate Hyper-v vhdx VM to Proxmox qcow2

Sourced from [broadband09](https://broadband9.co.uk/how-to-migrate-hyper-v-vhdx-vm-to-proxmox-qcow2/) with some changes for my environment.

## Requisites

| Name | Version |
|------|---------|
| <a name="Virtio Drivers"></a> [Virtio Drivers](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.215-2/virtio-win.iso)) | 0.1.215-2 |
| <a name="Qemu "></a> [Qemu ](https://opentofu.org/docs/intro/install/](https://cloudbase.it/qemu-img-windows/)) |x |
| <a name="FileZilla "></a> [FileZilla ](https://opentofu.org/docs/intro/install/](https://cloudbase.it/qemu-img-windows/](https://filezilla-project.org/download.php))) |x |

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

Exporting VHDx to Qcow2 format using qemu-img.exe
Now time for the export to qcow2

Browse to the Qemu-img application that has been downloaded via powershell admin then run the command, change your settings as desired. For us the command was:

`.\qemu-img.exe convert ‘D:\Hyperv\Virtual Hard Disks\SRVTeste.vhdx’ -O qcow2 D:\Migrar\SRVTeste.qcow2`

When you start running this command, there is no sort of feedback in the powershell console window so be patient.

You can check the progress by going to the output directory and looking at the current file properties of the file to see how much it has done so far.

 **Proxmox Preparations:**

Create a Virtual Machine on the Proxmox server without a Virtual disk and without a Network Device (for now)

Well, the proxmox wizard when creating a VM forces you to create a Virtual machine disk, just set the size to 1GB, then after it’s created, go into it and “Detach” and “Remove” that disk.

Go into the Proxmox VM settings and change the OS Type to Windows (if you didn’t do it at the creation wizard) and enable the “QEMU Guest Agent” setting:

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/17f6dc4e-89cf-4fca-9118-a3b4d328ff15)

As you can see the Hardware settings of the Virtual Machine is without both network adapter and without the disk:

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/15f42128-8b24-4b0c-b116-e7b65a33977c)

The new Virtual Machine has been assigned a new VM id which in our case is 103.
So we want to place this vDisk in the directory relating to this VM folder.
In our case the location will be `/var/lib/vz/images/`
So we need to create the 103 folder and then give it the correct permissions:

`mkdir /var/lib/vz/images/103`

`chmod 740 /var/lib/vz/images/103`

After this is done transfer the .qcow2 file to this folder via Filezilla, using ssh credentials over port 22.

(In my environment this was note the correct path, but I was unable to copy to the proper one due to a bug, so I did everything in here then moved the file via the web interface)

Once the transfer is finished go back into the Proxmox shell and run the following command:

`qm set 101 -scsi0 local:101/SRVOmegaTeste.qcow2`

Switch the BIOS type in Hardware Settings of the VM from Default to OVMF to enable UEFI 

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/6be2cf3b-db82-46e6-b255-8593fb2db94b)


**IF BSOD**

Attach virtio ISO in the machine CD/DVD drive.

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/74f0c53e-02bf-4073-a15c-aedd554c275d)


Boot to revcovery mode, open cmd and run `diskpart` > `list disk`

If no disk is available:

Exit diskpart and run drvload `D:\vioscsi\2k12r2\amd64\vioscsi.inf` (change the OS to yours)

When I now do a “list volume” I am able to see that the letter “C” has been assigned to the operating system installation

Run `dism /image:E:\ /add-driver /Driver:D:\ /recurse`

C: is my letter for Windows

D: is my letter for the virtio drivers image disc

Boot up the VM, should be running smoothly.

(After this I had to move the machine to /dev/iscsi-v wich was my iscsi storage)

![image](https://github.com/lucianothesilva/hyperv-to-proxmox/assets/20344783/2d474926-52ad-4d71-92ea-1dfd2dd15e3a)

