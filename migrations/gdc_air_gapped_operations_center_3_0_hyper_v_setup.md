# GDC Air-Gapped Operations Center 3.0: Hyper-V Setup (Simple Guide)

- **Estimated time:** ~2 hours
- **Owner:** OIC team
- **Source:** [Google Cloud Distributed Cloud Hosted Docs](https://docs.cloud.google.com/distributed-cloud/hosted/docs/latest/gdcag/infrastructure/operations-center-setup-30/setup-hyperv)

---

## Before You Start

Check all of the following first:

1. **Windows is installed** on **every** bare metal (BM) host. Do not start until it is.
2. **Host count:** Verify how many hosts you have (usually `BM01` to `BM07` per site).
3. **IP addresses:** Ensure you have an IP address for each host in CIDR format (e.g., `192.168.100.11/24`).
4. **Hardware version:** Identify whether you are on version `2.0` or `3.0` (default is `3.0`).
5. **Server generation:** Identify whether your servers are HPE Gen 10 or Gen 11.

---

## Phase 1: Rename Every Server (All BM Hosts)

1. Open **Server Manager**.
2. Click the **Computer Name** field.
3. Rename the server to `<SITE>-OC-BM0<N>` (e.g., `LON-OC-BM01`).
4. > [!IMPORTANT]
   > Do **NOT** change the Domain/Workgroup ("Member Of") setting. Leave it as it is for now.
5. Click **OK**.
6. Restart the server when prompted.

---

## Phase 2: Update Firmware with HPE SUM (All BM Hosts)

This phase installs all firmware, BIOS, and hardware driver updates.

1. Navigate to `C:\operations_center\bin` and mount the HPE Smart Update Manager (SUM) ISO (e.g., `hpe_sum.iso`).
   > [!NOTE]
   > Gen 10 and Gen 11 servers use different ISOs. Use the one that matches your hardware.
2. In the root of the mounted ISO, right-click `launch_sum.bat` and select **Run as Administrator**.
3. A web browser will open:
   - If a certificate warning appears, click **Advanced** $\rightarrow$ **Continue**.
   - If the page gets stuck loading, refresh the browser.
4. Select **Localhost Guided Update**.
5. Select **Interactive** deployment mode.
6. Ensure **"Install Prerequisite components if not already installed"** is checked.
7. Ensure **"Baseline of Install Set"** is selected, then click **OK**.
8. Wait while SUM takes inventory (it can be slow to start). When complete, click **Next**.
9. Verify that updates are selected, then click **Deploy**.
   > [!NOTE]
   > Review any warnings at the top and bottom of the page. If a warning does not need fixing, select **Ignore** and continue. Warnings about drivers pending firmware updates (or vice versa) are normal.
10. Once deployment finishes, click **Reboot**.
11. After the reboot, **repeat steps 1 through 10**. SUM must run multiple times until all updates are fully applied.
12. This phase is complete when the SUM discovery finds no remaining actions.

---

## Phase 3: Set Up the Hyper-V Hosts

### Step 1: Install PowerShell Modules (All Hosts: BM01–BM07)

Open PowerShell as Administrator and run:

```powershell
cd C:\operations_center\dsc
.\Install-PowerShellModules.ps1
```

### Step 2: Check Network Adapters (All Hosts: BM01–BM07)

Open PowerShell as Administrator and run:

```powershell
Get-NetAdapter | Sort-Object Name
```

Verify that **both** of the following show a **Broadcom P225p** interface:
- `PCIe Slot 2 Port 1`
- `PCIe Slot 2 Port 2`

> [!CAUTION]
> If they are missing, **STOP**. The network cards are in the wrong slots and must be physically reseated.

### Step 3: Initialise the Hosts

#### BM01 Only
*(In a multi-site setup, execute this only on **BM01 of Site 1**. This command also creates the `CONFIG1` virtual machine.)*

```powershell
cd C:\operations_center\dsc
.\Initialize-BareMetalHost.ps1 -IPv4Cidr <BM01_cidr> -HardwareVersion <nn.nn> -ConfigVMName <config_host>
```

**Parameters:**
- `<BM01_cidr>`: IP of BM01 in CIDR notation (e.g., `192.168.100.11/24`)
- `<nn.nn>`: Hardware version (`2.0` or `3.0`; defaults to `3.0` if omitted)
- `<config_host>`: Typically `<SITE>-CONFIG1`

---

#### BM02 to BM07
*(In a multi-site setup: Site 1 BM02–BM07, and Site 2 BM01–BM07.)*

```powershell
cd C:\operations_center\dsc
.\Initialize-BareMetalHost.ps1 -IPv4Cidr <BM0N_cidr> -HardwareVersion <nn.nn>
```

**Parameters:**
- `<BM0N_cidr>`: IP of that host in CIDR notation (e.g., `192.168.100.12/24`)
- `<nn.nn>`: Hardware version (`2.0` or `3.0`; defaults to `3.0` if omitted)

---

### Step 4: Reboot and Rerun Command (All Hosts)

After the host finishes rebooting, rerun its specific command from **Step 3**.  
*(Note: BM01 still uses its designated `-ConfigVMName` command).*

### Step 5: Answer the File Cleanup Prompt

The script copies `C:\operations_center` to `H:\operations_center`, then prompts to delete the original files on `C:`.

- **No errors encountered:** Type `A` (Yes to All) and press **Enter**.
- **Errors encountered:** Type `L` (No to All), resolve the errors, and retry.

Success output indicator:

```text
Bootstrap of Bare Metal Hyper-V host is complete!
PS H:\operations_center\dsc>
```

> [!NOTE]
> Your active working folder is now located on `H:\`, not `C:\`.

### Step 6: Set VLAN ID (Known Bug Workaround)

Run the following command on the hosts:

```powershell
Get-VMNetworkAdapter -SwitchName OC-Servers -ManagementOS | Set-VMNetworkAdapterVlan -Access -VlanId 201
```

> The bare metal hosts are now fully configured.

---

## Common Pitfalls

| # | Pitfall | Solution |
|---|---|---|
| 1 | **Running the CONFIG1 command on the wrong host** | Only **BM01 of Site 1** uses `-ConfigVMName`. |
| 2 | **Running SUM only once** | Run it repeatedly until it reports no further actions. |
| 3 | **Using the wrong SUM ISO** | Ensure the ISO matches the server generation (Gen 10 vs. Gen 11). |
| 4 | **Changing the domain while renaming** | Leave Domain/Workgroup settings untouched during initial rename. |
| 5 | **Typing `A` at the delete prompt despite errors** | Enter `L` to preserve files and troubleshoot errors first. |
| 6 | **Skipping the VLAN 201 command** | Always run Step 6, or network connectivity will fail. |