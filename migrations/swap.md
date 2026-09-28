## Step-by-Step Procedure to Swap the Drive

Follow this procedure to pull the donor drive from **BM07** and prepare it on **BM01**:

### Step 1: Remove the Drive from BM07
1. If **BM07** is powered on and configured in Windows/Hyper-V, shut down **BM07** gracefully or take the drive offline.
2. Physically pull the donor drive from its bay on **BM07**.

---

### Step 2: Insert the Drive into BM01
1. Insert the donor drive into the empty/failed drive bay on **BM01**.
2. Boot or reboot **BM01** into **UEFI System Utilities** (press **F9** during POST).

---

### Step 3: Handle the "Foreign Configuration" on BM01
> **Note:** Because the drive from **BM07** still contains RAID metadata from **BM07's** array, **BM01's** MR controller will detect it as a **Foreign Drive**. Since **BM01** and **BM07** share the same security key, **BM01** unlocks it automatically.

1. Navigate to: **System Utilities** > **System Configuration** > **HPE MRXXX Gen11** > **Main Menu** > **Configuration Management**
2. Select **Manage Foreign Configuration**.
3. Select **Clear Foreign Configuration** (*DO NOT import it*).
4. Confirm with **Yes**.

*This wipes **BM07's** volume metadata from the drive and marks the drive as **Unconfigured Good (UGood)**, making it available for **BM01's** array.*

---

### Step 4: Perform Option B / Volume Creation on BM01
Now that all drives on **BM01** (including the donor from **BM07**) are recognized and unconfigured:
* Drive Security remains active and enabled with your rack security key.

1. Go to: **Main Menu** > **Configuration Management** > **Create Logical Drive**
2. Proceed to build the three standard **RAID 6** volumes:
   * **System:** 150 GiB
   * **Hyper-V:** 4 TiB
   * **Data:** Remaining capacity
