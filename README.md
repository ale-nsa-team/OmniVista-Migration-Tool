# OVE/OVC4 to OV10 Migration Tool

A tool used for migrating data from OVE/OVC4 to OVC10.

**Version:** 1.0.1

Migrate network configuration data from OVE/OVC4 to OVC10 systems with automatic dependency resolution.

Supported migration paths :

- **OVE4 → OVC10** (on-prem OmniVista 2500 NMS 4.x - use `--device-source ove`)
- **OVC4 → OVC10** (cloud OVC 4.x - use the default `--device-source ovc4`)

## Download

The binaries are not stored in this repository. Download them from the [latest release](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest):

| File | Where | Needed for |
|---|---|---|
| `migration_tool.exe` | [Release assets](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest) | Everyone |
| `ovshared-4.9R3.jar` | [Release assets](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest) | OVE 4.9R3 sources only (see [Prerequisites](#for-ove-version-49r3)) |
| `sha256sum.txt` | [Release assets](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest) | Verifying the downloads |
| `config.toml` | This repository | Everyone |
| `ov4APGroupToOV10SiteMapping.csv` | This repository | Optional, only to spread AP Groups across several OV10 Sites |

Put `migration_tool.exe`, `config.toml` and `ov4APGroupToOV10SiteMapping.csv` in the same folder.

## Contents

- [Officially Validated Migration Matrix](#officially-validated-migration-matrix)
- [Prerequisites](#prerequisites)
  - [For OVE (Version 4.9R3)](#for-ove-version-49r3)
  - [For OVC4](#for-ovc4)
  - [For OVC10](#for-ovc10)
  - [Required Credentials](#required-credentials)
- [Migration Overview](#migration-overview)
- [Phase 1: Preparation (No Maintenance Window Required)](#phase-1-preparation-no-maintenance-window-required)
  - [Step 1: Save AP Configurations](#step-1-save-ap-configurations)
  - [Step 2: Configure Credentials](#step-2-configure-credentials)
  - [Step 3: Collect Data from OVE/OVC4](#step-3-collect-data-from-oveovc4)
  - [Step 4: Verify Collection Results](#step-4-verify-collection-results)
  - [Step 5: Migrate Configurations (Without Devices)](#step-5-migrate-configurations-without-devices)
    - [Optional: Distribute AP Groups Across Multiple OV10 Sites](#optional-distribute-ap-groups-across-multiple-ov10-sites)
  - [Step 6: Review Configurations on OVC10](#step-6-review-configurations-on-ovc10)
    - [Legacy OmniSwitch (OS6350 / OS6450)](#legacy-omniswitch-os6350--os6450)
    - [Switch Management Credentials](#switch-management-credentials)
- [Phase 2: Device Migration (Maintenance Window Required)](#phase-2-device-migration-maintenance-window-required)
  - [Step 7: Test Run - Migrate One Device](#step-7-test-run---migrate-one-device)
  - [Step 8: Migrate Remaining Devices](#step-8-migrate-remaining-devices)
  - [Step 9: Migrate AP Mesh Configuration](#step-9-migrate-ap-mesh-configuration)
  - [Precautions](#precautions)
- [Get Help](#get-help)
- [What Gets Migrated](#what-gets-migrated)
- [What's Not Migrated](#whats-not-migrated)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)

## Officially Validated Migration Matrix

| Environment | Source | Target | Validation Status |
|---|---|---|---|
| Production | OVC 4.9.3 GA | OVC 10.5.2 GA | Officially supported and validated |
| Production | OVE 4.9R3 GA + official patch | OV Terra 10.5.2 GA / MR | Officially supported and validated |
| QA setups | OVC 4.9.3 GA | OVC 10.6.1 EA | Validated in QA |
| QA setups | OVE 4.9R3 GA + official patch | OV Terra 10.6.1 EA | Validated in QA |

No other migration paths are currently validated by the test team.

## Prerequisites

### For OVE (Version 4.9R3)

There is a known bug in 4.9R3 that prevents passwords from being exposed via the migration API (`/api/migrationtool`).

Use the official OVE 4.9R3 patch package from the patch repository before running migration.

### For OVC4

- Version 4.9.3 GA
- For production use, contact OmniVista PLM

### For OVC10

Ensure the required firewall ports are open between your network devices and the OVC10 cloud. Refer to the **OVC10 Firewall Requirements** for the full list of ports and protocols needed before starting any migration.

For AP device migration, ensure **Auto-assign license for new devices** is enabled at the OVC10 organization level.

- If this is disabled, migrated APs can appear as registered but do not receive a license automatically and may fail to connect.
- The tool performs a non-blocking pre-flight advisory check and warns if this setting is disabled.
- Recommended action: enable Auto-assign before device migration, or manually assign licenses to migrated APs after migration.

### Required Credentials

**OVE/OVC4 Token:** Security > External Apps in OVE/OVC4 UI

**OVC10 Credentials:** Create application at `[OVC10_BASE_URL]/user/applications`:

- `org_id`, `site_id` (e.g., `6911b88a6b47ba847ef69f34`), `email`, `password`, `app_id`, `app_secret`

## Migration Overview

The migration is split into two phases to minimize risk and network downtime:

| Phase | Maintenance Window | Description |
|---|---|---|
| Phase 1: Preparation | Not required | Collect configs, migrate configurations only, verify in OVC10 |
| Phase 2: Device Migration | **Required** | Test with one device, then migrate remaining devices in batches |

## Phase 1: Preparation (No Maintenance Window Required)

All steps in this phase are non-disruptive and can be completed during normal business hours.

### Step 1: Save AP Configurations

> **CRITICAL:** Before deleting APs from OVE/OVC4, save their running configurations to prevent configuration loss.

1. Log in to your OVE/OVC4 web interface
2. Navigate to **Network > Managed Devices**
3. Select the APs you want to migrate (or select all APs)
4. Click the **Actions** dropdown menu
5. Select **Save to Running** (saves configuration to persistent storage)
6. Wait for confirmation that configurations have been saved

**Why this is critical:** Saving configurations protects against these scenarios:

- **AP Reboot During Migration:** If an AP reboots before onboarding to OVC10, it boots from saved configuration and continues advertising SSIDs
- **Onboarding Issues:** If problems occur during OVC10 onboarding, the AP maintains network service using local saved configuration
- **Call Home After Deletion:** When you delete an AP from OVE/OVC4 and it calls home to the Activation Server, the server may respond "I don't know you," potentially causing configuration loss. Saved local configuration prevents service disruption.

### Step 2: Configure Credentials

Navigate to the folder where you downloaded the tool and open PowerShell.

Copy the template and edit with your values:

```powershell
Copy-Item config.toml config.local.toml
```

Edit `config.local.toml` with your preferred text editor.

The example below follows the same order as the shipped `config.toml`. **Keep that order:** in TOML every setting belongs to the section above it, so the three global settings must stay at the top, before the first `[section]` header. Moved below one, they are read as part of that section and silently ignored.

```toml
# --- Global settings (must stay above every [section] header) ---
secret_key_pass = "your-secret-key-password"
data_dir = "data"           # Optional: directory for collected data
log_level = "INFO"          # Optional: DEBUG, INFO, WARNING, ERROR, CRITICAL

# Optional: trust certificate fingerprints for self-signed certs
[trusted_certs]
ovc4 = "fingerprint-here"
ovc10 = "fingerprint-here"

[ovc4]
base_url = "https://your-ovc4-server.com"
token = "your-ovc4-api-token"
max_workers = 5             # Optional: parallel collection workers (1-15, default: 5)
device_source = "ovc4"      # Optional: "ovc4" (default) or "ove" for OVE systems

[ovc10]
base_url = "https://your-ovc10-server.com"
org_id = "your-organization-id"
site_id = "your-site-id"
email = "your-email@domain.com"
password = "your-password"
app_id = "your-app-id"
app_secret = "your-app-secret"
skip_devices = true         # Optional: skip devices with --data-dir
mgmt_template = "OVC10 Default User Mgmt Template"  # Optional: override the switch management template
mapAPGroupsToSites = "ov4APGroupToOV10SiteMapping.csv"  # Optional: AP Groups per Site (see Step 5)

# Optional: tuning for the migrate phase - every key has a working default
[migration]
migration_request_timeout_ms = 2000
migration_request_max_retries = 5
migration_max_workers = 8
```

Only `secret_key_pass`, `[ovc4]` and `[ovc10]` are required; every commented-out key in `config.toml` is optional and documented in the file itself.

### Step 3: Collect Data from OVE/OVC4

**From OVC4:**

```powershell
.\migration_tool.exe collect --config-file config.local.toml --data-dir data
```

**From OVE:**

```powershell
.\migration_tool.exe collect `
  --config-file config.local.toml `
  --data-dir data `
  --device-source ove
```

### Step 4: Verify Collection Results

> **CRITICAL:** Before proceeding to migration, check the collection phase results.

The collection phase reports success/failure status at the end:

```
Data collection completed:
  [OK] Successful: 18
  [FAIL] Failed: 2
```

If any collectors failed:

1. Review the logs (`migration_tool.log`) to identify which objects failed and why
2. Choose one of these options:
   - **Option A - Contact Tech Support:** If failures involve complex objects or multiple dependencies, involve the Tech Support team to investigate and resolve the issues before proceeding to migration
   - **Option B - Manual Creation:** If only a few objects failed (e.g., 1-3 simple objects), you may manually create them in OVC10 through the UI before running the migration phase
3. **Do NOT proceed** to migration until all failed objects are either:
   - Successfully collected (re-run collection after fixes)
   - Manually created in OVC10
   - Confirmed as not needed for your deployment

**Why this is important:** Failed collection means incomplete data. Migrating incomplete data can result in:

- Missing dependencies causing migration failures
- Incomplete network configurations
- Manual rework to fix missing objects after migration

### Step 5: Migrate Configurations (Without Devices)

Migrate all configuration objects first, leaving devices for the next phase:

```powershell
.\migration_tool.exe migrate `
  --config-file config.local.toml `
  --data-dir .\data `
  --skip-devices
```

#### Optional: Distribute AP Groups Across Multiple OV10 Sites

By default every AP Group is created in the single Site named by `site_id` in your config file. If your OV10 organization has several Sites and you want your OV4 AP Groups spread across them, fill in `ov4APGroupToOV10SiteMapping.csv`, shipped in the same folder as `config.toml` and `migration_tool.exe`.

> **Doing nothing is safe.** The file ships with every sample row commented out, which means "put every AP Group in the default Site" - behaviour identical to previous releases. You only need this section if you actually want AP Groups in more than one Site.

##### Step A: Collect first, then list your AP Group names

The mapping file is checked against the data you collected in Step 3, so run `collect` before filling it in. The AP Group names live in `ap_group_data.json` under the `AP_GROUP` block. Open the file and read the `name` field of each entry, or list them from PowerShell:

```powershell
$data = Get-Content .\data\ap_group_data.json -Raw | ConvertFrom-Json
($data | Where-Object { $_.type -eq 'AP_GROUP' }).data.name
```

Copy the names exactly as they appear, including spaces, trailing characters and symbols such as `*`.

##### Step B: Find each OV10 Site Id

1. Log in to OVC10 and select the Site from the Site selector.
2. Read the address bar. The Site Id is the 24-character lowercase hex string in the URL, for example `6a01822e47e327655a5b4979`.
3. Repeat for every Site you plan to use.

A Site Id is always 24 characters of `0-9` and `a-f`. A Site **name** will not work.

##### Step C: Fill in the mapping file

Edit `ov4APGroupToOV10SiteMapping.csv`. Keep the header row exactly as shipped, then add one row per AP Group you want to place:

```csv
OV4 AP Group Name,OV10 Site Id
Corp-Wireless,6a01822e47e327655a5b4979
Guest-Wireless,6a01822e47e327655a5b4980
"Warehouse-APs, Legacy",6a01822e47e327655a5b4980
"#Staging-APs",6a01822e47e327655a5b4979
```

Rules for the file:

- **One Site per AP Group.** Listing the same AP Group twice aborts the migration. The file *places* a group, it does not clone it into several Sites.
- **You do not have to list every group.** Any AP Group not listed is created in the `site_id` from your config file.
- **Names are matched exactly** - spaces, case and characters such as `*` all count.
- Wrap a name containing a comma in double quotes: `"Warehouse-APs, Legacy",<SiteId>`
- A name starting with `#` must also be quoted: `"#Staging-APs",<SiteId>`. Unquoted, the line is read as a comment and silently ignored.
- A name containing a double quote uses CSV escaping: write `""` for each `"`.
- Lines starting with `#` are comments; blank lines are ignored. Excel, LibreOffice and Notepad output are all accepted (a UTF-8 byte-order mark and Windows line endings are handled).

##### What follows an AP Group into its Site - and what does not

| Object | Site it is created in |
|---|---|
| AP Group | The mapped Site |
| Provisioning Template of that AP Group | The same Site as its AP Group |
| AP devices whose group is mapped | The same Site as their AP Group |
| Mesh, AP IP Mode, AP IoT Radio, AP Radio / QoE settings | The Site where that AP was actually onboarded |
| SSID device-group assignments | The Site the referenced group lives in |
| Scheduled backups and scheduled upgrades | Always the `site_id` from the config file, by design |
| RF Profiles, certificates, AAA, UPAM, policies and other org-level objects | Not Site-scoped - unaffected by this file |

If an AP Group already exists in OVC10 in a different Site than the one you mapped it to, the tool follows the Site OVC10 reports for that group (so the AP's group assignment stays valid) and logs a warning naming both Sites.

##### Step D: Run the migration

No extra flag is needed. The file is picked up automatically when it sits next to the executable:

```powershell
.\migration_tool.exe migrate --config-file config.local.toml --data-dir .\data --skip-devices
```

To keep several mapping files around (for example one per customer site survey), point at one explicitly. The tool looks for the file in this order, first match wins:

1. `--map-ap-groups-to-sites <path>` on the command line
2. `mapAPGroupsToSites = "<path>"` under `[ovc10]` in the config file
3. `ov4APGroupToOV10SiteMapping.csv` in the current directory, if it exists

```powershell
# Command line
.\migration_tool.exe migrate --config-file config.local.toml --data-dir .\data `
  --map-ap-groups-to-sites .\my-site-mapping.csv
```

```toml
# Or in config.local.toml
[ovc10]
mapAPGroupsToSites = "my-site-mapping.csv"
```

Note the difference between options 1-2 and option 3: a file you name explicitly **must exist**. A typo in the path aborts the migration rather than silently falling back to the default Site.

##### Step E: Confirm the placement

The mapping is validated **before anything is migrated**. Every problem in the file is reported in a single run, each with its line number, and the entire migration is aborted. No objects of any kind are created, so you can fix the file and simply re-run:

| Log message | What to fix |
|---|---|
| `invalid header ... Expected 'OV4 AP Group Name,OV10 Site Id'` | The header row was edited or removed - restore it exactly |
| `expected 2 columns ... got N` | A row has a missing or extra comma - quote names containing commas |
| `'<value>' is not a valid OV10 Site Id` | Not a 24-character lowercase hex string - re-copy it from the OV10 URL |
| `AP Group '<name>' is already mapped on line N` | The same AP Group appears twice - keep one row |
| `AP Group '<name>' is not present in the collected OVC4 data` | Name misspelled, or collected before the group existed - fix the spelling or re-run `collect` |
| `AP Group to Site mapping references Site(s) that do not exist` | The Site Id is not in this organization - the log lists the Sites that are |
| `User does not have access to site '<id>'` | Your OVC10 account needs Site admin rights on every mapped Site |
| `AP Group to Site mapping file not found` | The path given via the flag or config file is wrong |

After a successful run, the placement is recorded in two places:

- The migration log ends with an **AP Group -> Site placement** block listing every mapped group and the default Site used for the rest.
- The HTML migration report contains an **AP Group → Site Placement** table naming the file the mapping came from.

Check one AP Group in the OVC10 UI as well, to confirm it landed where you expected before moving on to device migration.

### Step 6: Review Configurations on OVC10

Before migrating any devices, log in to OVC10 and manually verify the migrated configurations:

- AP Groups, RF Profiles, Provisioning Templates
- SSIDs and WLAN Services
- AAA Servers, AAA Profiles
- Access Role Profiles, Policies
- Any other objects relevant to your deployment

#### Legacy OmniSwitch (OS6350 / OS6450)

OVC10 10.6.1 supports OS6350 and OS6450 switches (running AOS 6.7.2.R08) as **Legacy OmniSwitch** devices. These switches have no ovng-agent and are managed exclusively via SNMP in OVC10.

Management User Templates are collected from OVC4 and migrated to OVC10 as part of switch configuration. If OVC4 already has a Management Template configured with SNMP credentials (`USE_SNMP`), that template will be collected and available in OVC10 after migration.

Before migrating Legacy OmniSwitch devices, you must:

1. **Collect the device SNMP credentials** - during collection, the tool reads each device SNMP profile and creates matching `MGMT_USERS_TEMPLATE` entries in `device_data.json` when needed.

2. **Override the generated template only when needed** - use `--mgmt-template` (or `[ovc10].mgmt_template` in `config.local.toml`) to force a specific template for the run:

   ```powershell
   .\migration_tool.exe migrate --config-file config.local.toml `
     --snapshot-file data\device_data.json `
     --mgmt-template "My-SNMP-Template"
   ```

> **Note:** SNMPv3 profiles with either `authNoPriv` (username + auth password) or `authPriv` are considered valid for template generation.

#### Switch Management Credentials

Review the Management User Templates under the switch configuration section in OVC10.

The migration tool supports overriding the switch management template during device migration:

- CLI flag: `--mgmt-template`
- Config key: `[ovc10].mgmt_template` in `config.local.toml`

Important behavior:

1. One migration run uses one management template value for all migrated switches in that run.
2. To use different templates for different switch groups, run migration multiple times with different `--serial` and `--mgmt-template` values.
3. If `--mgmt-template` is not provided, migration uses the `mgmtUsersTemplate` values already present in the collected snapshot data. Migration no longer generates new management templates.

Resolve any discrepancies before proceeding to device migration. Manual corrections are much easier to make before devices are onboarded.

## Phase 2: Device Migration (Maintenance Window Required)

Perform the following steps during a scheduled maintenance window.

### Step 7: Test Run - Migrate One Device

Start with a single device (one AP or one switch) to validate the end-to-end process before migrating the rest.

#### 7.1 Prepare a single-device snapshot

You can either use `--serial` to filter by serial number directly from `device_data.json`, or create a separate snapshot file.

**Option A - Use `--serial` (recommended, no file editing needed):**

Skip this step and go directly to 7.2. The `--serial` flag handles the filtering.

**Option B - Create a separate snapshot file:**

Open `data\device_data.json` and locate the entry for the device you want to migrate. Copy that object into a new individual snapshot file, keeping the same wrapper format:

For AP - save as `data\SSZ194603166.json`:

```json
[
    {
        "type": "AP_DEVICE",
        "data": [
            {
                "name": "SSZ194603166",
                "serialNumber": "SSZ194603166",
                "groupName": "default group",
                "deviceFamily": "AP",
                "deviceLocationLldp": true
            }
        ]
    }
]
```

For Switch - save as `data\P39Q0178.json`:

```json
[
    {
        "type": "SWITCH_DEVICE",
        "data": [
            {
                "name": "P39Q0178",
                "serialNumber": "P39Q0178",
                "deviceFamily": "AOS",
                "mgmtUsersTemplate": "Default Management User Template"
            }
        ]
    }
]
```

#### 7.2 Delete device from OVE/OVC4

1. Collect Support Info on the test device in OVE/OVC4 (**Administration > Audit > Collect Support Info**) - save the output for later comparison
2. Log in to OVE/OVC4 and navigate to the **Device Catalog** page
3. Locate the device by serial number and select **Delete** (or **Remove**)
4. Confirm the removal

#### 7.3 Migrate the device

**Using `--serial` (no separate file needed):**

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\device_data.json `
                             --serial SSZ194603166
```

**Switch example with template override (`--mgmt-template`):**

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\device_data.json `
                             --serial P39Q0178 `
                             --mgmt-template "Template-SW-Core"
```

**Using a separate snapshot file (Option B from 7.1):**

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\SSZ194603166.json
```

#### 7.4 Verify the migration

1. **Update software version** - edit the device in OVC10 and update AWOS to the latest available version (optional but recommended)
2. **Collect Support Info** on the device in OVC10 and compare against the output saved before deletion to confirm configurations are correctly applied
3. **Verify client connections** - confirm wireless clients (APs) or network connectivity (switches) are working as expected

If issues occur, follow the [Troubleshooting](#troubleshooting) steps. Use the Collect Support Info outputs from OVE/OVC4 and OVC10 side-by-side to identify configuration differences.

If your network includes AOS switches, repeat with one switch before proceeding.

### Step 8: Migrate Remaining Devices

Once the test run is successful, migrate all remaining devices using `device_data.json`.

1. **Delete all remaining devices from OVE/OVC4** before migrating (same as step 7.2 - via **Device Catalog > Delete**). Do this in batches and verify the network after each batch before continuing.

2. **Migrate using the full device snapshot:**

   ```powershell
   .\migration_tool.exe migrate --config-file config.local.toml `
                                --snapshot-file data\device_data.json
   ```

   This migrates all devices in `data\device_data.json` in a single run.

**Migrating in batches with `--serial`** (recommended for large deployments - no file splitting needed):

```powershell
# Migrate a specific set of devices by serial number (comma-separated format)
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\device_data.json `
                             --serial SN000001,SN000002,SN000003
```

```powershell
# Switch batch with a specific management template
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\device_data.json `
                             --serial SW000101,SW000102,SW000103 `
                             --mgmt-template "Template-SW-Branch"
```

Verify the network after each batch before continuing with the next.

### Step 9: Migrate AP Mesh Configuration

Mesh configuration is collected separately in `data\mesh_data.json`. Migrate it **after** the corresponding AP devices have been migrated to OVC10, because the tool resolves each AP by serial number before applying its Mesh settings:

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\mesh_data.json
```

Verify Mesh APs come up as expected after migration. If an AP was not migrated yet, migrate that device first and rerun the Mesh migration.

### Precautions

- **Do not use `--skip-devices` when using `--snapshot-file`.** That flag only applies with `--data-dir` and is silently ignored otherwise. It will not protect you from accidentally migrating devices.
- **Do not run collect and migrate simultaneously** against the same output directory. Collect overwrites snapshot files in place.
- **Verify the target group exists in OVC10** before migrating. Migration fails at the group lookup step; there is no automatic group creation.
- Migrating a device that is still actively managed by OVE/OVC4 may cause inconsistent state on the AP. Ensure the device is ready to be cut over before migration.

## Get Help

```powershell
.\migration_tool.exe --help
.\migration_tool.exe collect --help
.\migration_tool.exe migrate --help
```

## What Gets Migrated

- **Network Infrastructure:** AP Groups, Provisioning Templates, RF Profiles
- **RF Profiles:** Only profiles actively referenced by at least one AP Group are collected and migrated. Profiles that exist in OVE/OVC4 but are not assigned to any AP Group (e.g. the AP Group is using the "default" RF profile instead) will not be included. To migrate such a profile, either assign it to an AP Group in OVE/OVC4 before running collection, or recreate it manually in OVC10.
- **Switch Configuration:** Switch Templates, Value Mappings, Management User Templates
  - **Management User Templates:** These templates are collected and migrated to OVC10.
  - **Switch device migration:** The tool can set `mgmtUsersTemplate` on switch payloads during migration.
  - **Template override support:** Use `--mgmt-template` (or `[ovc10].mgmt_template`) to apply one template value per migration run.
- **Wireless Networks:** SSIDs and WLAN Services
- **AP Mesh:** Mesh configuration for Access Points, collected in `mesh_data.json` and migrated after the AP devices exist in OVC10
- **AP Configuration:** `AP_IP_MODE`, `AP_IOT_RADIO`, `AP_RADIO`, and `AP_QOE_RTLS_STATUS`. These provide per-AP IP mode and static IP settings, IoT radio settings, radio settings, and QoE/RTLS status. They are collected in `ap_config_data.json` and migrated after AP devices exist in OVC10.
- **Authentication:** AAA Servers (RADIUS, LDAP), AAA Profiles
- **Access Control:** Access Role Profiles, Access Auth Profiles
- **Policies:** Location Policies, Period Policies, WCF Profiles
- **External Services:** IoT/Location Servers (External Engines)
- **Security:** VPN Settings, Certificates (various types)
- **User Management:** Employee Accounts, Guest Accounts, Company Properties
- **Devices:** Switches, Legacy OmniSwitches, and Access Points
  - Supported switch types:
    - **AOS switches** (`deviceFamily: AOS`) - standard AOS 8 switches
    - **Legacy OmniSwitch** (`deviceFamily: LEGACY_OMNISWITCH`) - OS6350 and OS6450 running AOS 6.7.2.R08, managed via SNMP in OVC10
  - **Automatically skipped** (unsupported in OVC10):
    - OS2260, OS2360 (AOS 5x)
    - OS2220 (Websmart AOS 8.3.1)
    - OS6250 (AOS 6.7.1), OS6850/E, OS6855 (AOS 6.4.6), OS9700/E, OS9800/E (AOS 6.6.4)
  - Devices without serial numbers are skipped (treated as third-party)
  - **Device filtering:** Use `--serial` (or `--device-serial`) to migrate selected devices only
- **Scheduling:** Upgrade Schedulers, Event Responders

## What's Not Migrated

The following configurations are **not** collected or migrated by this tool and must be handled manually.

### Switch Configurations That Become Stale After Migration

> **Why this matters:** OVE/OVC4 (OmniVista) pushes its own management/VPN IP to switches as the destination for monitoring and analytics traffic. After migration, these entries on the switch still point to the old OVE/OVC4 IP, which is no longer valid on OVC10. This leaves stale configurations that may cause misdirected traffic, failed trap delivery, or unnecessary load on the old system.

**Recommended approach:**

1. **Before deleting devices from OVE/OVC4** - remove the following configurations from the switches while OVE/OVC4 is still managing them
2. **After completing migration** - re-apply these configurations in OVC10 using OVC10's correct management/VPN IP

**Configurations to clean up:**

- **SNMP Trap** - OVE/OVC4 configures itself as an SNMP trap receiver on managed switches. After migration, the old OVE/OVC4 trap station VPN IP on the switch becomes stale. Remove the OVE/OVC4 trap destination from switches before cutting them over, then add the correct OVC10 trap destination afterwards.

- **sFlow** - OVE/OVC4 configures sFlow collector IP and sampling settings on switches. The collector IP will reference OVE/OVC4's VPN IP and becomes invalid post-migration. Remove sFlow collector configuration before deletion; re-configure in OVC10 after migration.

- **Application Visibility** - OVE/OVC4 configures the analytics/DPI collector endpoint on switches. This IP reference becomes stale after migration. Remove the Application Visibility configuration before deleting devices from OVE/OVC4, then reconfigure under OVC10.

- **Policy Server** - OVE/OVC4 acts as the OmniVista policy server for switches (used for Unified Policy enforcement). The policy server IP pushed to switches points to OVE/OVC4's VPN IP. After migration to OVC10, this reference is stale. Remove the policy server configuration on OVE/OVC4 before device deletion, then assign the correct OVC10 policy server IP after migration.

- **Authentication Server / UPAM IP (UNP/UPAM switches)** - Switches configured for UNP/UPAM authentication point to the OVC4 UPAM/authentication-server IP. The migration tool moves configuration data but does **not** automatically re-point switches to the new OVC10 UPAM IP, so UNP/UPAM authentication keeps targeting the old, now-invalid address after migration (OVNG-24862 Item 1).

**Post-Migration Step: Reconfigure UPAM/Authentication-Server IP on Affected Profiles**

After the migration is complete, you must update the UPAM/authentication-server IP on all affected Profiles to point to the OVC10 UPAM instance. Perform the following steps:

1. **Locate the OVC10 UPAM IP and port:** In OVC10, navigate to **Configure > Network Access > Authen Servers** and record the UPAM server IP address and port number.
2. **Update each UNP profile:** Edit every relevant profile to use the OVC10 UPAM IP address and port as the RADIUS/authentication server.

### Objects With No Equivalent Setting in OVC10

These objects exist in OVE/OVC4 but have no corresponding configuration in OVC10 and are therefore not migrated:

| OVE/OVC4 Location | What is NOT migrated |
|---|---|
| UPAM UI Page (OVE/C) | Switch User Account (employee/guest accounts are already migrated) |
| Settings Page (OVE) | Email Server |
| WLAN > Floor Plan (OVE/C) | Floor Plan (SSIDs are already migrated) |
| Security > Authentication Servers (OVE) | ACE Server (RADIUS and LDAP servers are already migrated) |
| Security > Users and Groups (OVE/C) | Roles, Users, Groups |
| Unified Access (OVE/C) | AAA Global Setting (AAA profiles and access role profiles are already migrated) |
| Configuration > Report (OVE/C) | Report |
| Configuration > CLI Scripting (OVE/C) | CLI Scripting |
| Configuration > Resource Manager > Upgrade Image (OVE) | Upgrade Image (upgrade schedulers are already migrated) |
| Network > Discovery Profile (OVE) | Discovery Profile |
| Network > Topology (OVE/C) | Topology Map |
| Network > Analytics (OVE/C) | Analytic Profiles, Chart View Profiles |
| Network > IOT (OVE/C) | Custom VRF (IOT categories are already migrated) |
| Network > Trap Configuration (OVE/C) | Trap Configuration |

These must be recreated manually in OVC10 if equivalent functionality exists, or are simply not applicable in the new system.

### Objects Supported in OVC10 But Not Migrated by This Tool

The following objects have a configuration equivalent in OVC10 but are not currently handled by the migration tool and must be recreated manually:

| OVC10 Location | What is NOT migrated |
|---|---|
| UPAM (OVE/C) | Guest Operator (guest accounts are already migrated) |
| UPAM > Settings > LDAP/AD Configuration (OVE/C) | The LDAP / Active Directory directory that UPAM authenticates users against. (UPAM's External RADIUS servers and the Security > Authentication Servers LDAP servers used by switches are already migrated - this is a different, UPAM-specific setting.) |
| AP Registration > Access Points (OVE/C) | Remaining per-AP override settings such as AP Configuration Update, RF Profile Override, and similar AP-level overrides. IP mode is migrated through `ap_config_data.json`. |

**Post-Migration Step: Recreate the UPAM LDAP/AD Directory and Re-point Its Access Policies**

The tool collects UPAM Access Policies, Guest/BYOD strategies, Captive Portal templates and External RADIUS servers, but **not** the LDAP/AD directory definition under **UPAM > Settings > LDAP/AD Configuration**. Any Access Policy whose authentication source is LDAP or Active Directory therefore arrives in OVC10 without the directory it points at, and must be finished by hand:

1. **Recreate the directory in OVC10** - navigate to **Network Access > UPAM Settings** and add the LDAP or Active Directory server using the same host, port, base DN, bind account and SSL settings you recorded from OVE/OVC4. Do this before reviewing the migrated Access Policies.
2. **Fix the authentication source on each affected Access Policy.** OVE/OVC4 offers a single combined `External LDAP/AD` choice; OVC10 splits it into two distinct values, `External LDAP` and `External AD`. OVE/OVC4 stores nothing that says which one a policy meant, so the tool migrates every `External LDAP/AD` policy as `External LDAP` and writes a warning to the log:

   ```
   Access policy authenticationAgainst 'External LDAP/AD' is not an OVC10 value; migrating as
   'External LDAP'. Change it in OVC10 if this policy authenticates against Active Directory.
   ```

   Search the migration log for `authenticationAgainst` to list every policy that needs review, then switch the ones that authenticate against Active Directory to `External AD` in OVC10.
3. **Re-select the directory on the policy** - after step 1 the newly created LDAP/AD server is selectable; assign it to each policy, then save.

> Access policies using **Local Database**, **External Radius** or **IMSI/IMEI Database** need no rework: their authentication source migrates unchanged, and External RADIUS servers are migrated with the rest of the UPAM data.

**For AP migrations, this is critical:**

- Remaining per-AP overrides from OVE/OVC4 are not reapplied by this tool and may need to be recreated manually after AP onboarding.
- Mesh is supported and is collected in `mesh_data.json`; migrate it after the corresponding AP devices are present in OVC10.

### Other Configurations Requiring Manual Action

- **Historical data** - Logs, alarms, event history, and performance statistics from OVC4 are not transferred to OVC10.
- **Network topology / maps** - Any custom network maps or floor plans configured in OVC4 must be recreated in OVC10.
- **Captive Portal pages** - Custom-branded captive portal page templates are not migrated and must be recreated in OVC10.

## Troubleshooting

### Check Logs

View detailed information in `migration_tool.log` in your working directory.

**Enable debug logging:**

```toml
# In config file
log_level = "DEBUG"
```

Or via command line:

```powershell
.\migration_tool.exe collect `
  --log-level DEBUG `
  --config-file config.local.toml
```

### Common Issues

#### OVE-Specific Issues

**Patch Files Not Applied:**

- Contact your OVE administrator to verify patches are installed
- Ensure OVE server is running version 4.9R3 with the required official patch

**Token Authentication Failure:**

- Regenerate token in OVE UI: **Security > External Apps**
- Verify token hasn't expired
- Test connectivity using PowerShell:

  ```powershell
  Invoke-WebRequest -Uri "https://your-ove-server/api/about" `
    -Headers @{Authorization="Bearer YOUR_TOKEN"} `
    -SkipCertificateCheck
  ```

**Data Collection Failures:**

- Use correct device source: `--device-source ove` for OVE
- Contact OVE administrator to check service status and logs

#### OVC4-Specific Issues

**API Authentication Failures:**

- Regenerate token in OVC4 UI: **Security > External Apps**
- Ensure token has appropriate permissions
- Verify token format (no extra quotes/spaces)

**Device Collection Issues:**

- Verify devices are visible in OVC4 UI: **Network > Device Catalog**
- Check device connectivity status
- Use `device_source = "ovc4"` (default)

**Performance Issues:**

- Adjust `max_workers` (increase for faster, decrease for stability)
- Default: 5, Range: 1-15
- Collect during off-peak hours

#### Common to Both

**Scheduler Backup/Upgrade Depends on Device Migration:**

- Backup scheduler and upgrade scheduler profiles that reference device serial numbers require those devices to already exist in OVC10.
- If referenced devices are not migrated yet, scheduler migration fails by design (no partial migration).
- Typical error message:
  - `... cannot be migrated because device(s) [...] have not been migrated to OVC10 yet ...`
- Recommended recovery:
  1. Migrate the missing devices first.
  2. Re-run the current migration step to remigrate the scheduler profile.

**Guest Account Phone Number Uniqueness:**

- OVC10 requires guest account phone numbers to be unique (after normalization with the `+` prefix).
- OVE/OVC4 can store multiple guest accounts with the same phone number.
- Before migration, review guest accounts and remove or consolidate duplicate phone numbers to avoid migration failures.

**Data VPN Settings (DATA_VPN_SETTING):**

If you have VPN settings configured in OVE/OVC4, verify them before migration:

- **Server VPN IP Range Error:** "The Server VPN IP must be in the IP Range"
  - **Cause:** OVC10 requires the Server VPN IP to fall within the defined IP range
  - **Solution:** In OVE/OVC4, verify your VPN configuration before migration:
    1. Navigate to **Network > AP Registration > Data VPN Servers** in OVE/OVC4
    2. Ensure Server VPN IP is within the configured IP range
    3. Update the configuration if needed before collecting data
    4. Re-collect data after fixing VPN settings

**Network Connectivity:**

- Verify connectivity:

  ```powershell
  Test-NetConnection -ComputerName your-server -Port 443
  ```

- Check firewall rules allow HTTPS (port 443)
- Consider VPN if accessing remotely

**Device Not Behaving as Expected After Migration:**

- Collect Support Info from the device in OVC10: **Devices > select device > Collect Support Info**
- Compare it against the Collect Support Info captured from OVE/OVC4 before deletion (see Step 7.2)
- Differences in the output will point to which configurations were not correctly applied
- Check that the AP Group, SSID, and RF Profile assignments match between OVC4 and OVC10

**Configuration Not Saved:**

- Always save AP configurations: **Network > Managed Devices > Actions > Save to Running**
- Verify "Configuration Saved" status before deletion

**Device Deletion Issues:**

- Ensure devices not assigned to critical services
- Use UI to properly delete: **Network > Device Catalog > Delete**
- Verify deletion completed before migration

#### Migration Tool Field-Proven Issues

These are recurring field/validation issues observed during OVE/OVC4 to OVC10/OV Terra migrations.

**OVE 4.9R3 password API gap:**

- Symptom: password-backed fields (for example AP Group/UPAM components or Mgmt Users SNMP password) are missing in collected data.
- Action: ensure OVE 4.9R3 MR1 (or later) patch level is installed before running `collect`.

**Guest/employee/account data truncation at large scale:**

- Symptom: collect appears successful but guest/company/employee snapshots are incomplete when source record counts are large.
- Action: use an OVE build that includes batch retrieval support for guest/employee/account data; if counts look suspicious, compare OVE UI totals with collected JSON totals before `migrate`.

**Guest phone normalization and uniqueness conflicts:**

- Symptom: migrate fails with phone validation errors or duplicate-phone conflicts.
- Action: normalize phone numbers to E.164-style format (`+<country><number>`) and remove duplicates before collect/migrate.

**RF profile bounds validation:**

- Symptom: migration rejected for `externalAntennasGain` (out of allowed range).
- Action: clamp/adjust values to OVC10 valid range (`1.0..16.0`) before submission.

**SNMP management template requirement for legacy/third-party switches:**

- Symptom: migration finishes but selected switches are not fully managed, or device migration fails because required SNMP settings are missing/mismatched.
- Action: ensure a valid Management Users Template with matching SNMP settings exists in OVC10/OV Terra, and pass it during migration for affected devices.

**Post-migration onboarding for certain switch/AP models:**

- Symptom: migration reports success, but some devices are not immediately in managed state.
- Action: perform required post-migration onboarding (for example call-home URL update and callhome trigger for certain switch types, or AP DHCP/AS update + reboot where applicable).

**Backup scheduler object depends on migrated devices:**

- Symptom: scheduler migration fails with dependency errors because referenced device serials do not yet exist in OVC10.
- Action: migrate devices first, then re-run scheduler migration.

### Getting More Help

For issues not covered above:

1. Enable DEBUG logging: `log_level = "DEBUG"`
2. Review `migration_tool.log` with full details
3. Contact support with log file and error messages

## Security Best Practices

1. **Protect Config Files:** Store in a secure location with restricted access. In PowerShell, you can set file permissions:

   ```powershell
   icacls config.local.toml `
     /inheritance:r `
     /grant:r "$env:USERNAME:(R)"
   ```

2. **Handle Snapshot Files Securely:** Contain sensitive data - delete after migration or store securely
3. **Verify Certificate Fingerprints:** Only trust certificates from known servers
4. **Use HTTPS:** All API communications use encrypted connections
