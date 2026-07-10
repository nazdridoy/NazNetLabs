# Chapter 17: Running Windows 11 in a Virtual Machine or in the Cloud

Deploy and manage virtual environments for testing software, isolating applications, or running older operating systems.

---

## Lab 17.1: Enabling Hyper-V in Windows Features

Windows 11 includes Hyper-V, but you must enable it manually before you can create virtual machines.

### Procedure

1. **Opening Windows Features**:
   - Press **Win + R**, type `optionalfeatures`, Enter
   - Scroll down to find **Hyper-V**
2. **Installing the feature**:
   - Check the **Hyper-V** box to select both **Management Tools** and **Platform**
   - Click **OK** and wait for the installation to finish
   - Restart your computer when prompted
---

## Lab 17.2: Setting Up Hyper-V on Windows 11

After enabling the feature, you need to open and connect to the Hyper-V Manager console.

### Procedure

1. **Opening Hyper-V Manager**:
   - Right-click Start > **Terminal (Admin)**
   - Type `virtmgmt.msc`, Enter
   - Alternative: Search Start for "**Hyper-V Manager**"
2. **Connecting to the local server**:
   - Select your PC name in the left pane under **Hyper-V Manager**
   - You can now see virtual machines, checkpoints, and actions for this host
---

## Lab 17.3: Creating and Managing Virtual Machines

Hyper-V Manager provides the tools to create, start, stop, and configure virtual environments.

### Procedure

1. **Starting the new VM wizard**:
   - In Hyper-V Manager, select your host machine
   - Click **Action** > **New** > **Virtual Machine** from the top menu
---

## Lab 17.4: Understanding Machine Generations

When creating a VM, you must choose between Generation 1 (legacy BIOS) and Generation 2 (UEFI).

### Procedure

1. **Choosing the generation**:
   - During the New Virtual Machine wizard, reach the **Specify Generation** step
   - Select **Generation 2** to support modern OS features like Secure Boot
   - Select **Generation 1** only for legacy 32-bit operating systems
---

## Lab 17.5: Storage Controllers and Virtual Disks

Virtual machines need virtual storage devices to hold the operating system and user files.

### Procedure

1. **Allocating virtual disk space**:
   - In the wizard, reach the **Connect Virtual Hard Disk** step
   - Specify the name, location, and size (e.g., 64 GB for Windows 11)
   - Click **Next** to proceed
---

## Lab 17.6: Creating a Virtual Switch

Virtual machines require a virtual switch to communicate with your physical network hardware and the internet.

### Procedure

1. **Opening Virtual Switch Manager**:
   - In Hyper-V Manager, click **Virtual Switch Manager** in the right **Actions** pane
2. **Creating an external switch**:
   - Select **External**, then click **Create Virtual Switch**
   - Name the switch and select your physical network adapter
   - Click **Apply** and **Yes** to the network warning
---

## Lab 17.7: Using Quick Create

Hyper-V Quick Create automatically downloads and configures an operating system image with minimal input.

### Procedure

1. **Launching Quick Create**:
   - Open **Hyper-V Manager**
   - Click **Action** > **Quick Create** in the top menu
2. **Selecting an image**:
   - Choose **Windows 11 dev environment** or **Ubuntu** from the list
   - Click **Create Virtual Machine**
   - Wait for the download and configuration process
---

## Lab 17.8: Building a Custom VM Step-by-Step

A custom setup provides granular control over memory allocation and boot media.

### Procedure

1. **Configuring VM properties**:
   - Click **Action** > **New** > **Virtual Machine**
   - Assign startup memory (minimum 4096 MB for Windows 11)
   - Connect the VM to your external virtual switch
2. **Mounting the installation media**:
   - On the **Installation Options** step, select **Install an operating system from a bootable image file**
   - Browse to your Windows 11 `.iso` file and click **Finish**
---

## Lab 17.9: Using Checkpoints

Checkpoints capture the VM state at a specific moment in time.

### Procedure

1. **Creating a checkpoint**:
   - Right-click a running or stopped VM > **Checkpoint**
   - The checkpoint appears in the **Checkpoints** pane with a timestamp
2. **Applying a checkpoint**:
   - Right-click a checkpoint in the middle pane > **Apply**
   - Choose to create another checkpoint or just apply the old one
---

## Lab 17.10: Customizing Virtual Machine Settings

You can adjust virtual hardware parameters like the processor count or security settings at any time.

### Procedure

1. **Enabling the virtual TPM**:
   - Right-click the VM > **Settings**
   - Under **Security**, check **Enable Trusted Platform Module** (required for Windows 11)
2. **Allocating CPU cores**:
   - Select **Processor** in the left pane
   - Increase the **Number of virtual processors** to at least 2
   - Click **Apply**
---

## Lab 17.11: Adding a New Virtual Disk

Attach an additional `.vhdx` file to provide more storage to an existing VM.

### Procedure

1. **Adding the drive hardware**:
   - Open **VM Settings**
   - Select **SCSI Controller** on the left, select **Hard Drive**, click **Add**
2. **Creating the virtual file**:
   - Click **New** below the **Virtual hard disk** path box
   - Follow the wizard to create a new dynamically expanding `.vhdx` file
   - Click **OK** to save the VM settings
---

## Lab 17.12: Removing a Virtual Disk

Detach a virtual disk when no longer needed or if you want to mount it to a different VM.

### Procedure

1. **Removing the drive**:
   - Open **VM Settings**
   - Select the **Hard Drive** under the **SCSI Controller**
   - Click **Remove** on the right side, then **Apply**
> [!NOTE]
> Removing the disk from the VM settings does not delete the `.vhdx` file from your physical hard drive.

---

## Lab 17.13: Importing, Exporting, and Moving VMs

Export a VM to back it up or move it to another Hyper-V host without losing your data.

### Procedure

1. **Exporting the VM**:
   - Right-click the VM > **Export**
   - Choose a destination folder and click **Export**
2. **Importing a VM**:
   - In Hyper-V Manager, click **Action** > **Import Virtual Machine**
   - Point the wizard to the folder containing the exported VM
   - Choose whether to register it in-place or copy it
---

## Lab 17.14: Running Virtual Machines from the Cloud

Use Microsoft Azure or Windows 365 to run a Windows 11 desktop in the cloud, accessible from any device.

### Procedure

1. **Connecting via Windows App**:
   - Open the Microsoft Store, search for and install **Windows App**
   - Launch the app and sign in with your Microsoft Entra ID
2. **Launching the Cloud PC**:
   - Click your assigned Cloud PC or Azure Virtual Desktop from the **Home** tab
   - The remote session opens in full screen
