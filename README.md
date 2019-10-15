# Guide to Slipstream VirtIO Drivers into Windows ISO's
## Introduction
This guide will help manually add VirtIO drivers to Windows ISO's so that the drivers are automatically installed and available to a Windows VM on Proxmox. This way the VirtIO ISO will not need to be attached to a virtual machine on Proxmox and there will be no need to manually load the drivers 1-by-1 in Windows Setup at install time. While these ISO's should work with any KVM based virtualization, I am only testing them using a Proxmox host.

By slipstreaming these drivers direct into an ISO, it drastically reduces the work required to setup new Windows based VM's by removing the need to manually install these drivers for each machine.

### Tested Windows Versions
I have only tried and tested the following methods with the following Windows Versions/ISO's:

- Windows 10 version 1903 (acquired direct from Media Creation Tool or the Microsoft website [here](https://www.microsoft.com/en-us/software-download/windows10))
- Windows Server 2019 (acquired from the [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019?filetype=ISO))

Just because I have not tested a version of Windows does not mean the methods outlined here will not work. They may work as is, with modification or not at all. Feel free to try other versions as needed.

### Disclaimers
These methods are NOT intended to circumvent licensing or legitimate purchase/use of the mentioned software. Users are responsible for properly licensing and activating their copies/installation of Windows. The details provided in this guide are intended to help users test Windows in virtual environments using either evaluation licenses or licenses the user owns already. All users are responsible for legally licensing, activating and using their installation(s) of Windows on their own systems.

I also am not covering how to inject updates into Windows, though it is possible using similar methods. Should you want to also inject Windows updates I suggest you research the topic and incorporate the additional setup and commands into your workflow.

### What are VirtIO Drivers?
In brief, VirtIO drivers are drivers designed to allow Windows VM's to perform better when used with KVM based virtualization, like Proxmox currently uses. This includes but is not limited to: display drivers, storage drivers, networking drivers, etc. The VirtIO ISO also contains the qemu guest agent which allows the KVM host (Proxmox) to have better control over a guest OS, in this case Windows. This allows functions like properly shutting down a Windows VM from Proxmox's controls. The ISO also contains the Balloon service files as well as the driver, which when installed allow a Windows guest to work with the memory ballooning feature of KVM/Proxmox. Generally speaking, the drivers make the VM run more smoothly, with more control and less issues.

### Resources
The following are good resources to help with the process.

- Official VirtIO ISO and drivers: https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/ (I generally use the "Latest virtio-win iso" entry under "Direct downloads")
- Windows 10 Media Creation Tool or ISO download: https://www.microsoft.com/en-us/software-download/windows10
- Windows Server Evaluation Center: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019?filetype=ISO
- Windows Assessment and Deployment Toolkit (ADK): https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install (generally grab the latest version)
- Recommended Settings for Windows Proxmox VM's: https://davejansen.com/recommended-settings-windows-10-2016-2018-2019-vm-proxmox/ (Great article on the manual process and settings to use for the VM)
- Installing Windows Server GUI-less as a Proxmox VM: https://medium.com/@carll/installing-server-2016-2019-core-gui-less-with-proxmox-649ba8d634db (Another great article from the perspective of doing the process without a desktop GUI)

## Tips
- Keep original copies of your ISO's in a safe place, especially if they are not simply downloads from Microsoft's site (like a retail ISO).
- Prior to doing any of these steps, ensure your original ISO works for installation.
- You may consider keeping copies of the modified install.wim and boot.wim in case you need to inject additional drivers or updates to them at another time.
- Read over the entire guide before attempting the steps. Understand what each step does and why you are doing it.

## Preparation
The following are the steps to prepare for slipstreaming the VirtIO drivers into a Windows ISO.

1. Setup a Windows based machine to prepare the ISO's on. This can be another VM or a physical machine. I recommend using either Windows 10 Pro or Windows Server 2019 to do this on. Install Windows, update it, etc. Make sure to have enough space to handle your ISO's + spare space. I would generally recommend ~25GB free space per ISO plus at least 32GB for Windows and programs to install.
2. Install Windows ADK. The only required selection from the checklist of install options is "Development Tools". If PXE booting Windows images you may also choose "Windows Preinstall Environment (Windows PE)". This will give us access to commands like `dism` and commands in PowerShell needed for working with the ISO's.
3. Download the VirtIO ISO and have whatever Windows ISO(s) you plan to work with on this machine.
4. In file explorer, change the settings to show hidden files and folders and to display file extensions. Generally on the View Tab, 2 check boxes.

Next, make the following directories. These are not hard requirements. You can name them whatever you like and place them any place on the system you like. I am going to use and reference the directories below throughout the guide however and will be placing them all in the `Downloads` folder of my current user.

1. `iso-win#` - This directory will contain all the files from the ISO of a specific version of Windows
2. `iso-virtio` - This directory will contain all the files from the VirtIO ISO
3. `drv-win#` - This directory will contain only the drivers needed for a specific version of Windows
4. `wim-win#` - This directory will contain the current `install.esd`, `install.wim` and/or `boot.wim` to be modified
5. `mount` - The directory we will mount the .wim files to modify them.

For entries with `win#` replace "#" with whatever version/abbreviation you like. Ex: `iso-win10` or `drv-win2k19`.

After making the above directories, we need to copy files from the ISO's to the proper locations.

1. Mount the Windows ISO.
2. Select all the files and folders on the Windows ISO and copy them. Paste them to the `iso-win#` directory.
3. Unmount the Windows ISO and Mount the VirtIO ISO.
4. Select all the files and folders on the VirtIO ISO and copy them. Paste them to the `iso-virtio` directory.
5. Unmount the VirtIO ISO.

The VirtIO contents is explained [here](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/#virtio-win-iso-contents). The gist is there is a folder for each type of driver (display, network, etc.). Within each folder are multiple folders, one for each version of Windows. Within each of those folders is a folder for the CPU architecture like amd64, ARM64 or x86. We only need to inject the drivers that match the same version of Windows being used with the same architecture planned (generally speaking only "amd64").

I am sure there are ways to automate the following separation of drivers, but I just did it manually.

I copied all the files/folders in `iso-virtio` to `drv-win#`. I then went into each folder and deleted any folder representing a version of windows other than the one matching the `drv-win#` folder, the version of Windows we are currently working on injecting drivers in. Ex: if working with a Windows 10 ISO, in the `NetKVM` directory select all folder, ctrl+click the `w10` folder to de-select it and then delete. This will leave only the `w10` folder in the `NetKVM` folder. Repeat this for every folder under `drv-win#`.

While neither file in `guest-agent` is a driver, I would keep at least the version matching the architecture (typically the `x86_64.msi` file). It will prove helpful post install.

I then went back to the `drv-win#` folder.

1. Click in the search bar at the top right of the file explorer window.
2. From the "Search" tab, select "Kind" and select "folder". This inserts "kind:=folder" in the search box.
3. Put a single space after that entry and then type in "ARM" and hit enter or wait.
4. Once complete, select all the results and delete them. This should delete all of the "ARM" subfolders in `drv-win#` as we don't need those drivers.
5. Repeat steps 2-3 but instead of "ARM", type in "x86". Only do this if you do not plan to install Windows as 32bit/x86 (in some cases it's not supported anyway, like in Windows Server). Then repeat step 4 and delete all the resulting folders. This deletes all the "x86" subfolders from the `drv-win#` folder.

This should leave just the folders/drivers needed for the version of Windows we are injecting the drivers into. This is important to help reduce the size of an image and to avoid having incompatible drivers loaded/installed.

**Optional but Recommended**
I recommend taking the `drv-win#` folder and copying it to the `iso-win#` folder. Doing so will help make post install steps easier.

Now we need to copy/move the `install.esd` or `install.wim` (depends on Windows version) file from the Windows ISO to the `wim-win#` directory. We also need to copy the `boot.wim` file.

1. Copy `install.esd` or `install.wim` from `C:\Users\myuser\Downloads\iso-win#\sources` to `C:\Users\myuser\Downloads\wim-win#`
2. Copy `boot.wim` from `C:\Users\myuser\Downloads\iso-win#\sources` to `C:\Users\myuser\Downloads\wim-win#`

This should leave us with:

- Contents of the Windows ISO in `iso-win#`
- Contents of the VirtIO ISO in `iso-virtio`
- Only the folders/files of the drivers needed for the specific version of Windows being worked on in `drv-win#`
- 2 files in `wim-win#`, `install.wim` or `install.esd` and `boot.wim`
- An empty `mount` folder.
- Windows ADK installed

## Slipstream VirtIO Drivers into Windows ISO
This process is repetitious. `.wim` files (and `.esd`) often have multiple indexes. So when you boot from a Windows 10 ISO for example, and you have options to install "Windows 10 Home", "Windows 10 Pro", etc. each is a different index in the same `install.wim`/`install.esd` on the same ISO. This means we need to inject the driver into each option we intend to keep/use. We also need to inject the driver into the `boot.wim`, which also has indexes (generally 1 for Win PE and 2 for the Windows Setup). Unless PXE booting and using the Win PE portion, we only need to inject into the Windows Setup portion for the `boot.wim`.

To cut down on this, I recommend removing any index (option) that you do not intend to use ever. For example if you will never be using "Windows 10 Education" from a Windows 10 ISO, better to remove it then to spend time injecting drivers to it. We will cover how to do this soon.

The following directions will typically provide 2 methods for each step. One using the "Deployment and Imaging Tools Environment" (I will refer to this as ADK, though not technically true), the other using Windows Powershell (PS) and commands provided to it by the ADK module. Both should have the same end result, it's just a matter of preference. However, do not mix and match them as I have not tested it in this fashion. To open "Deployment and Imaging Tools Environment" go to "Start | Windows Kits | Deployment and Imaging Tools Environment". To instead open PowerShell do "Start | Windows PowerShell | Windows PowerShell". It may be required to open either as administrator. This can be done by right clicking its icon in the Start menu and selecting "More | Run as Administrator".

For all commands below replace paths with your actual paths, IE: Do **NOT** just copy and paste these commands. For reference I will use `C:\Users\myuser\Downloads\` as the base path, with subfolders as laid out above.

### 1. Getting Wim Info
This command will display information about a given image file (`.wim` or `.esd`). Primarily we want to know how many indexes there are, and which index(s) we will use and which we won't.

ADK
```dism /Get-WimInfo /WimFile:C:\Users\myuser\Downloads\wim-win#\install.wim```

PS
```Get-WindowsImage -ImagePath C:\Users\myuser\Downloads\wim-win#\install.wim```

### 2. (Situational) Remove Unused Indexes
This command will remove the designated index from the image. If you are unsure, do **NOT** delete an index. Only use this command to delete options from an ISO you are sure you will not use. I also highly recommend to re-run the command from step 1 above after removing each index. This is because options are re-indexed after each removal. IE: if index 3 was "Windows 10 Home" and index 4 was "Windows 10 Pro", after removing index 3, "Windows 10 Pro" now becomes index 3 instead of 4. AKA: Dont assume the indexes are the same after removing another.

ADK (replace "1" with the desired index number found using the command from step 1)
```dism /Delete-Image /ImageFile:C:\Users\myuser\Downloads\wim-win#\install.wim /Index:1 /CheckIntegrity```

PS
```Remove-WindowsImage -ImagePath "C:\Users\myuser\Downloads\wim-win#\install.wim" -Index 1 -CheckIntegrity```

### 3. (Situational) Convert .esd to .wim
This step is only needed if the Windows ISO had an `install.esd` instead of `install.wim`, if you already have an `install.wim` skip this step. I am providing this step for educational purposes only, I suggest you research the difference between `.esd` and `.wim`. Users are responsible for doing this step and understanding the implications. It should be noted that using `install.esd` and not converting it is unlikely to work for the rest of this guide.

This step can also take a relatively long time (like 5-15 mins on a modern machine). Just let it run to completion.

ADK
```dism /Export-Image /SourceImageFile:C:\Users\myuser\Downloads\wim-win#\install.esd /SourceIndex:1 /DestinationImageFile:C:\Users\myuser\Downloads\wim-win#\install.wim /Compress:Max /CheckIntegrity```

PS
```Export-WindowsImage -SourceImagePath "C:\Users\myuser\Downloads\wim-win#\install.esd" -SourceIndex 1 -DestinationImagePath "C:\Users\myuser\Downloads\wim-win#\install.wim" -CompressionType Max -CheckIntegrity```

## Per Index Steps
The following steps will need to be repeated per index that needs to be modified. Generally, this is a minimum of 2 times, once for at least one index of the `install.wim` and again for the `boot.wim` setup index. It needs to be done for each additional index in the `install.wim` that you want the drivers available to.

Perform the steps 4-7 in order for a single image/index before repeating them for the next.

Replace `install.wim` with `boot.wim` when injecting drivers into the setup image.

### 4. Mount an Image
In this step we mount an image/index to inject the drivers into. This is similar in concept to mounting an ISO except that we are mounting a specific index within the `install.wim` or `boot.wim` image.

ADK
```dism /Mount-Image /ImageFile:C:\Users\myuser\Downloads\wim-win#\install.wim /Index:1 /MountDir:C:\Users\myuser\Downloads\mount```

PS
```Mount-WindowsImage -Path "C:\Users\myuser\Downloads\mount" -ImagePath "C:\Users\myuser\Downloads\wim-win#\install.wim" -Index 1```

### 5. Inject/Slipstream Drivers
Now we are going to slipstream or inject all of the drivers we have in `C:\Users\myuser\Downloads\drv-win#` into the currently mounted Windows image. This way they will be pre-installed and we will not need to manually install them one by one during setup or after installation.

ADK
```dism /Add-Driver /image:C:\Users\myuser\Downloads\mount /driver:C:\Users\myuser\Downloads\drv-win# /Recurse /ForceUnsigned```

PS
```Add-WindowsDriver -Path "C:\Users\myuser\Downloads\mount" -Driver "C:\Users\myuser\Downloads\drv-win#"" -Recruse -ForceUnsigned```

In the above commands, `-Recurse` has the command look through all the folders and subfolders of our `drv-win#` directory for drivers. The `-ForceUnsigned` switch removes the requirement for drivers to be digitally signed to be installed which may be required for some drivers in the VirtIO collection.

### 6. Verify Drivers Have Been Added
It's a good idea to ensure that the drivers have actually been added to the image. While not essential to the process, IE: skipping this step will still result in the same outcome, it's a good practice to help avoid mistakes. The following command will list out the 3rd part drivers in the image, which should list the VirtIO drivers we previously injected.

ADK
```dism /Image:C:\Users\myuser\Downloads\mount /Get-Drivers```

PS
```Get-WindowsDriver -Path C:\Users\myuser\Downloads\mount```

### 7. Save Changes and Un-mount the Image
Once the drivers have been injected, we need to save the changes to the image while we un-mount it.

ADK
```dism /Unmount-Image /MountDir:C:\Users\myuser\Downloads\mount /Commit /CheckIntegrity```

PS
```Dismount-WindowsImage -Path "C:\Users\myuser\Downloads\mount" -Save -CheckIntegrity```

## Final One Time Steps (per ISO)
The proceeding step is meant to be done after steps 4-7 have been repeated for each file/index required. Be sure that you have done steps 4-7 for the setup index of `boot.wim` and at least one index of the `install.wim`.

### 8. Copy Updated `.wim` Files Back to ISO Folder
We need to take the modified `install.wim` and `boot.wim` and place them back in the ISO file/folder structure. These steps can be done in File Explorer.

1. Copy `install.wim` from the `C:\Users\myuser\Downloads\wim-win#` directory.
2. Paste `install.wim` to `C:\Users\myuser\Downloads\iso-win#\sources\` directory.
1. Copy `boot.wim` from the `C:\Users\myuser\Downloads\wim-win#` directory.
2. Paste `boot.wim` to `C:\Users\myuser\Downloads\iso-win#\sources\` directory.

### 9. Rebuild Windows ISO
Our last step takes everything we have done and rebuilds a new ISO using the updated images with the drivers contained within them. The resulting ISO will be what we want to install Windows VM's from going forward.

This particular step doesn't have ADK/PS variations, it's just a single command. As far as I am aware, the same command can be run from either ADK (cmd) or PS.

```oscdimg -u2 -m -bC:\Users\myuser\Downloads\iso-win#\efi\microsoft\boot\efisys.bin C:\Users\myuser\Downloads\iso-win#\ C:\Users\myuser\Downloads\iso-win#\new-iso-name.iso```

Replace `new-iso-name.iso` with the desired name of the new Windows ISO. I recommend this not be the same as the original ISO. The above command also assumes UEFI bootable media, I am personally not concerned with BIOS booting Windows.

This will allow for ISO images larger than 4.7GB single layer DVD's and beyond the approx. 4GB `install.wim` limit for an ISO.

## Post Install Using the New ISO
After installing Windows in a Proxmox VM using your new ISO, there are still a few steps that you need to take to get the most out of it. Earlier in the guide, I recommended copying our `drv-win#` folder to the `iso-win#` folder. I am going to assume in the steps below this was done. If not, you will need to mount the VirtIO ISO/disc for the current Windows VM to finish the post install steps.

### 1. Install QEMU Guest Agent
The Windows ISO (and the VirtIO ISO) which should be still mounted post Windows Install as a CD/DVD drive/disc contains a folder `drv-win#\guest-agent` which should contain at least a single `.msi` file (if not 2 files) named `qemu-ga-i386.msi` or `qemu-ga-x86_64.msi` (for 32bit/64bit Windows OS's respectively). This is similar to the VMWare/ESXi guest agent allowing the host (in this case Proxmox) to have more accurate information of and better control over its guest VM's.

To install this on the Windows guest VM, in File Explorer:

1. Navigate to the Windows CD/DVD (the ISO mounted within the VM that we installed from), let's assume `D:\` and go to the guest-agent folder `D:\drv-win#\guest-agent`.
2. Double click on the appropriate `.msi` for your architecture of Windows (`i386` = 32bit and `x86_64` = 64bit).

A command prompt window will open briefly then vanish, that's it. It should be installed.

### 2. Enable the Balloon Service
The Balloon driver and service allow KVM guests like our Windows VM to work with the memory ballooning feature of KVM. While our previous steps have added the driver to the ISO and thus our install, we still need to setup and activate the service for ballooning in our Windows VM.

To install/activate the ballooning service:

1. Create a new folder `C:\Program Files\Balloon`.
2. Navigate to the Windows CD/DVD (the ISO mounted within the VM that we installed from), let's assume `D:\` and go to the proper balloon directory `D:\drv-win#\Balloon\w10\amd64`. Replace `w10` with the folder for your version of Windows and `amd64` for your architecture of Windows.
3. Copy all the files in the proper sub directory of `Balloon\w10\amd64\*`. There are roughly 6 files in this folder, one of which should be `blnsvr.exe`.
4. Paste these files into the created directory from step 1 at `C:\Program Files\Balloon`.
5. Open `cmd` (command prompt) as an Admin from `C:\Program Files\Balloon` or `cd` to `C:\Program Files\Balloon`.
6. Run the command `blnsvr.exe -i`. This should give a few lines of output in cmd about starting the service.

### 3. Reboot the VM
To finalize the post install configuration, reboot the Windows VM. To verify that the guest agent is working, you can look at the Summary of the Windows VM in Proxmox. Specifically under "Bootdisk size" should be the "IPs" entry listing the IP addresses of the VM with a button that says "More". The "More" button displays further guest agent network info.

You can confirm that the Balloon service is running from the Windows guest using the PS command: `get-service -name ballo*`. It should return a result stating the Balloon Service is running.

## Conclusion
This guide should provide the means to:

- Create a Windows ISO (or ISO's) with VirtIO drivers baked in.
- Implement post install steps for Windows VM's in Proxmox to take advantage of qemu guest agent and the balloon driver/service.
- Reproduce the process of injecting the VirtIO drivers into Windows media using ADK (cmd) or PowerShell.
- Eliminate the need to manually install VirtIO drivers during installation or post install for Windows VM's.

Hopefully you find this guide helpful. Please open an issue should you have any corrections, updates, questions or input on this guide. Thank you.
