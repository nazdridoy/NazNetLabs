# Chapter 19: Managing Windows PCs in the Enterprise

When managing a fleet of computers, joining them to a centralized domain and enforcing policies ensures security, consistency, and compliance across your organization.

---

## Lab 19.1: Connecting a Windows 11 PC to a Domain

Joining an active directory domain allows users to log in with their corporate credentials and grants IT administrators control over the device.

> [!NOTE]
> Windows 11 Pro, Enterprise, or Education editions are required to join an Active Directory or Entra ID domain. Windows 11 Home does not support this feature.

### Procedure

1. **Opening Access work or school settings**:
   - Right-click Start > **Settings** > **Accounts**
   - Scroll down and click **Access work or school**
2. **Joining the domain**:
   - Click the **Connect** button
   - Click "**Join this device to a local Active Directory domain**" at the bottom of the prompt
   - Enter your domain name when prompted
3. **Authorizing the join**:
   - Enter the username and password of an account with permission to join devices to the domain
   - Choose the account type (**Standard user** or **Administrator**) for the person who will use this PC
   - Restart the computer to complete the process
---

## Lab 19.2: Managing Group Policy

The Local Group Policy Editor lets you enforce restrictions customize Windows behavior without editing the registry manually. 

### Procedure

1. **Opening the Group Policy Editor**:
   - Press **Win + R**, type `gpedit.msc`, Enter
   - Navigate the left pane to drill down into **Computer Configuration** or **User Configuration**
2. **Applying a specific policy**:
   - Navigate to **Computer Configuration** > **Administrative Templates** > **System** > **Logon**
   - Double-click "**Always use classic logon**" (or a similar policy you want to enforce)
   - Change the state from **Not Configured** to **Enabled**, then click **Apply**
3. **Forcing a policy update**:
   - For changes to take effect immediately, right-click Start > **Terminal (Admin)**
   - Run: `gpupdate /force`
   - Wait for the "Computer Policy update has completed successfully" message
