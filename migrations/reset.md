## Procedure: Option B (Full Reset / Set a New Security Key)

If you must apply a new key or ensure the controller matches a newly generated secret, perform the following steps:

### Step 1: Delete Existing Logical Drives
> **Note:** On HPE MR / MegaRAID controllers, Drive Security cannot be disabled while secured logical drives or drive groups exist.

1. Go to: **Main Menu** > **Configuration Management** > **Clear Configuration**
2. Check **Confirm** and select **Yes**.

---

### Step 2: Disable Drive Security
1. Go to: **Main Menu** > **Controller Management** > **Advanced Controller Management**
2. Select **Disable Drive Security**.
3. Check **Confirm** and select **Yes**.

---

### Step 3: Re-enable Drive Security
1. Once disabled, return to **Advanced Controller Management**. (*Enable Drive Security will now be available.*)
2. Select **Local Key Management (LKM)**.
3. Enter your **Security Key** (and optional Identifier).
```
# Generates a 28-character compliant secret string
Add-Type -AssemblyName System.Web
[System.Web.Security.Membership]::GeneratePassword(28, 4)
```
5. Uncheck **Pause for password at boot time**.
6. Check **I Recorded the Security Settings for Future Reference**.
7. Click **Enable Drive Security**.

---

### Step 4: Create Logical Volumes
1. Proceed to **Configuration Management** > **Create Logical Drive** to create the three required RAID 6 volumes.
