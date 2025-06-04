# Autoinstall Testing Setup

- [Autoinstall Testing Setup](#autoinstall-testing-setup)
  - [Preface](#preface)
  - [Process](#process)
    - [Clean NVME Drives](#clean-nvme-drives)
    - [Reburn Carrera Firmware to Golden Versions](#reburn-carrera-firmware-to-golden-versions)
      - [Version mismatch](#version-mismatch)
      - [Non default credentials](#non-default-credentials)
  - [Next Steps](#next-steps)

## Preface

The autoinstall process is a utility we will provide Celestica to fully install all software in a default state on a ztC Endurance system in the factory. The autoinstall process will:
1. Install Standby OS on both Compute Modules
2. Transfer base RHEL ISO to Compute Module B
3. Runs zen_prep
4. Triggers **generic** RHEL install onto Compute Module A

However, when Celestica runs this script they will be doing so on an *entirely* fresh ztC Endurance system. This means:
1. Fresh golden firmware installed
2. Default credentials for:
   1. BMC GUI
   2. BMC Sysadmin
   3. BIOS Redfish

This is important for several reasons. First being that zen_prep is part of the flow of the autoinstall. The "factory_install.sh" script that drives the autoinstall will not proceed if it detects **ANY** zen_prep failures. This includes: incorrect firmware versions (Non Golden), unsigned drivers (Developer Builds), Non default BMC GUI, BMC Sysadmin, or redfish credentials. Thus, we need to ensure our systems are completely configured for all tests to pass [^1]. This entails:
1. **FRESH** Golden Firmware
   1. Satisfies correct firmware versions + default credentials
2. Using official CI / CD builds with signed drivers
3. Running autoinstall process from Ubuntu System with access to required autoinstall tools
   1. bolti.cdx.eng.stratus.com is the best equivalent in our lab 

[^1]: It is worth noting that this process *can* be tested with non default credentials via extra flags in the factory_install.sh process. However, this is not a good replication of the conditions Celestica will be using this tool under. Therefor, it is not advised to do this.

---
## Process

### Clean NVME Drives

**If RHEL is installed:** Having an existing RHEL instance installed on the system nvme drives can be problematic autoinstall processes. RHEL is aggressive with the boot order after it is installed. This is fine and expected behavior when running a system in a customer context, but troublesome in a development context. Thus, we should wipe RHEL from our nvme drives before we proceed any further.

**If RHEL is not installed:** Move on to [Reburn Firmware](#reburn-carrera-firmware-to-golden-versions) 

**Steps:**
1. Log onto Compute Module currently running RHEL
2. Reboot to "Stratus Maintenance" boot entry
```bash
sudo efibootmgr             # Read which boot entry number is is "Stratus Maintenance", ex: 0004
sudo efibootmgr -n 0004     # Setting the next boot to "Stratus Maintenance"
sudo shutdown -r now        # Reboot the system to the next boot entry selected
```
3. Once booted to "Stratus Maintenance", Run:
```bash
sudo nvme format -s1 -f /dev/nvme1n1
sudo nvme format -s1 -f /dev/nvme2n1
```

### Reburn Carrera Firmware to Golden Versions

The steps taken here largely depend on the state of the firmware on the system being tested. If the firmware is all the current "Golden" versions [^2]

[^2]: If you are generally unfamiliar with firmware burns, it is advised to receive assistance on the burning process after you have compiled the list of which firmwares need to be reburned

```
FW                  Version     
------------------- ----------- 
BIOS                4.00.00     
BMC                 4.24.00     
SES_IO              3.1.0.24    
SES_STG             4.0.0.24    
MBCPLD              01.04.00    
IOCPLD              01.03.00    
STGCPLD             01.03.00    
```

Match what your system currently has, and **ALL** your credentials are default (BMC, Redfish) then you don't need to burn any firmware. The sections below go into more detail on what conditions require which firmware reburn.

#### Version mismatch
If you notice a discrepancy between the Golden FW versions listed above, and what is currently installed on your system, note which mismatches there are and append them to a list

#### Non default credentials
- BMC GUI credentials: If your BMC GUI credentials are **not** admin/admin = Reburn BMC
- Redfish credentials: If your BIOS redfish credentials are **not** Administrator/superuser = Reburn BIOS **OR** manually reset with curl command (don't bother with this if you plan to reburn BIOS anyways due to a version mismatch)
```bash
old_pwd=""; # insert current redfish password
bmc_ip=""; # insert BMC IP address
curl -s -k -u Administrator:${old_pwd} --data "{\"Password\": \"superuser\"}" -H "Content-Type: application/json" -X PATCH https://$bmc_ip/redfish/v1/AccountService/Accounts/1 --header 'If-Match: *'
# Repeat this process for both BMCs
```

Once you have assembled a list of firmware that needs to be reburned, follow the steps outlined in [Carrera Firmware](https://stratustech.atlassian.net/wiki/spaces/CAR/pages/3170238510/Carrera+Firmware#Firmware-Burn-Procedures) for the respective category

---
## Next Steps

With necessary firmware reburned, and defaults restored you are now ready to test the autoinstall procedure with clean factory like defaults. These documents outline the general procedure:
1. [Factory Install Powerpoint](https://smartm-my.sharepoint.com/:p:/g/personal/collin_hay_stratus_com/EXVmWPCNBRpGs1VqOTKZu88BGXN7Rx-QXKcaX5POZUC32w?e=tYirp8)
2. [Autoinstall tarball README](https://github.com/stratustech/Carrera-Tools/blob/main/utilities/autoinstall/README.md) [^3]

[^3]: The Autoinstall tarball is not currently in the CI / CD pipeline, one has already been supplied to QA but can be acquired on a request basis for now. There is currently a copy on the Ubuntu-Devel server at /h/chay/autoinstall.tar.gz

You will also need a system to conduct the autoinstall from. As it stands, we don't know the specifics of the Celestica Malaysia site factory. So an in lab system with all the necassary tools running Ubuntu will suffice for now. As it stands, **bolti.cdx.eng.stratus.com** has all the necessary tools and connections to conduct a full autoinstall.