# Chapter 18: Using Android and Linux on Windows 11

Windows 11 integrates with other ecosystems, letting you access your mobile device directly from your desktop and run native Linux environments without a dual-boot setup.

---

## Lab 18.1: Linking an Android Phone to Your PC

Use the Phone Link app to view text messages, see mobile notifications, and access recent photos without picking up your phone.

### Procedure

1. **Setting up Phone Link**:
   - Open the Start menu, type `Phone Link`, and press Enter
   - Select **Android** and sign in with your Microsoft account
2. **Pairing the device**:
   - Open the **Link to Windows** app on your Android phone
   - Scan the QR code displayed on your PC screen
   - Grant the necessary permissions on your phone when prompted
---

## Lab 18.2: Using Windows Subsystem for Linux (WSL)

WSL lets developers run a GNU/Linux environment, including command-line tools, utilities, and applications, directly on Windows.

> [!NOTE]
> You no longer need to manually enable the Virtual Machine Platform or WSL features through the control panel. The modern installer handles prerequisites automatically.

### Procedure

1. **Installing WSL**:
   - Right-click Start > **Terminal (Admin)**
   - Type `wsl --install` and press Enter
   - Wait for the installation of the default Ubuntu distribution to complete
   - Restart your computer when prompted
2. **Setting up your Linux user**:
   - Open the Start menu and launch **Ubuntu**
   - The first launch opens a terminal to finalize the installation
   - Enter a new UNIX username and password when prompted
