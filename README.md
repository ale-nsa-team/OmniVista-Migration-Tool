# OV Migration Helper Tool

Migrates the configuration and the devices of OV4 (OVE or OVC4) to OV10 (OVC10 or OV Terra).

**Version:** 1.0.3

## Download

Download the files of the [latest release](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest), and `config.toml` and `ov4APGroupToOV10SiteMapping.csv` from this repository, into one folder. See [Files You Need](#files-you-need).

## Upgrading From 1.0.1

1.0.3 is the first public release since 1.0.1. If you used 1.0.1, note these changes (details in [Release Notes](#release-notes)):

- **AOS 6x switches (OS6350, OS6450) and third-party devices are not supported in 1.0.3.** See [Legacy OmniSwitch](#legacy-omniswitch-os6350--os6450).
- **The validated target is OV10 10.6.1.** See [Officially Validated Migration Matrix](#officially-validated-migration-matrix).
- **The config file is simpler.** The sections are now `[ov4]` and `[ov10]`; a `config.local.toml` that still uses `[ovc4]` and `[ovc10]` works as before. `migrate` authenticates with the OV10 application only: remove `email` and `password` from `[ovc10]`. See [Files Written for an Earlier Version](#files-written-for-an-earlier-version).
- **`device_source` and `--device-source` are no longer used.** The tool tells OVE from OVC4 by the host name of `[ov4] base_url`: an OVE must not have `ovcirrus.com` in its host name. See [Step 2](#step-2-collect-data-from-ov4).
- **Self-signed certificates are trusted by the tool itself**, which saves their fingerprint in `config.local.toml`. See [Certificates](#certificates).
- **Collect again** with 1.0.3 if you migrate the OVE UPAM External LDAP server: snapshots collected with 1.0.1 hold it under its OVE field names.

## Revision History

| Version | Date | Description |
|---|---|---|
| 0.2.0 | June 2026 | Initial version. |
| 1.0.1 | September 2026 | Migrates to OV Terra. Places OV4 AP Groups in several OV10 Sites. See [Release Notes](#release-notes). |
| 1.0.3 | November 2026 | Validated from OVE and OVC4 4.9R3 to OVC10 and OV Terra 10.6.1. The guide is reorganized, with a Quick Start. Migrates the OVE UPAM External LDAP server as a non-secure server. Leaves out the guest accounts that OV10 rejects, widens Data VPN IP ranges and retries slow collection requests. Simpler config file: `[ov4]` and `[ov10]`, OV10 application only, self-signed certificates trusted by the tool. Reports the objects that OV10 does not support, and the settings that wait for their devices, instead of failing them. Does not support AOS 6x switches or third-party devices: OV4 4.9R3 does not expose their SNMP credentials. You can add the AOS 6x switches to OV10 by hand. See [Release Notes](#release-notes). |

## Contents

- [Introduction](#introduction)
  - [How the Tool Works](#how-the-tool-works)
  - [Terms Used in This Guide](#terms-used-in-this-guide)
  - [Officially Validated Migration Matrix](#officially-validated-migration-matrix)
  - [Files You Need](#files-you-need)
- [Plan Your Migration](#plan-your-migration)
  - [The Two Phases](#the-two-phases)
  - [What Is Migrated, in Short](#what-is-migrated-in-short)
  - [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Phase 1: Configuration Migration (No Maintenance Window)](#phase-1-configuration-migration-no-maintenance-window)
  - [Step 1: Fill In config.local.toml](#step-1-fill-in-configlocaltoml)
  - [Step 2: Collect Data From OV4](#step-2-collect-data-from-ov4)
  - [Step 3: Check the Collection](#step-3-check-the-collection)
  - [Step 4 (Optional): Place AP Groups in Several OV10 Sites](#step-4-optional-place-ap-groups-in-several-ov10-sites)
  - [Step 5: Migrate the Configuration](#step-5-migrate-the-configuration)
  - [Step 6: Read the Migration Report](#step-6-read-the-migration-report)
  - [Step 7: Review the Configuration on OV10](#step-7-review-the-configuration-on-ov10)
- [Phase 2: Device Migration (Maintenance Window Required)](#phase-2-device-migration-maintenance-window-required)
  - [Before You Migrate Devices](#before-you-migrate-devices)
  - [Step 8: Test With One Device](#step-8-test-with-one-device)
  - [Step 9: Migrate the Remaining Devices in Batches](#step-9-migrate-the-remaining-devices-in-batches)
  - [Step 10: Migrate the AP Mesh Configuration](#step-10-migrate-the-ap-mesh-configuration)
  - [Step 11: Migrate the AP Settings and Schedulers That Waited for Devices](#step-11-migrate-the-ap-settings-and-schedulers-that-waited-for-devices)
  - [Step 12: Point the Switches to OV10](#step-12-point-the-switches-to-ov10)
- [Troubleshooting](#troubleshooting)
  - [Read the Log](#read-the-log)
  - [Connection and Certificates](#connection-and-certificates)
  - [Collection Problems](#collection-problems)
  - [Migration Problems](#migration-problems)
  - [Device Problems](#device-problems)
  - [Getting More Help](#getting-more-help)
- [Reference](#reference)
  - [Command-Line Options](#command-line-options)
  - [Configuration File](#configuration-file)
  - [Snapshot Files](#snapshot-files)
  - [What Gets Migrated](#what-gets-migrated)
  - [What's Not Migrated](#whats-not-migrated)
  - [Security Best Practices](#security-best-practices)
- [Release Notes](#release-notes)

## Introduction

### How the Tool Works

The tool is a Windows program, `migration_tool.exe`, that you run in PowerShell. It moves the configuration of an OV4 server to an OV10 organization with two commands:

1. `collect` reads the configuration of OV4 and writes it into snapshot files: one JSON file per group of objects, in a folder on your computer. The passwords and secrets in them are encrypted with a passphrase you choose, `secret_key_pass`.
2. `migrate` reads the snapshot files and creates the objects on OV10. Each run writes a log and an HTML report of what was migrated, what failed and why.

The configuration moves first, while OV4 still manages the devices. The devices move last, in a maintenance window: you remove each device from OV4, then `migrate` adds it to OV10. See [The Two Phases](#the-two-phases).

### Terms Used in This Guide

| Term | Meaning |
|---|---|
| OV4 | The source server: OVE (OmniVista 2500 NMS 4.x, on premises) or OVC4 (OmniVista Cirrus 4.x cloud). |
| OV10 | The target: OVC10 (OmniVista Cirrus 10 cloud) or OV Terra. |
| Snapshot file | A JSON file written by `collect`, such as `ap_group_data.json`. See [Snapshot Files](#snapshot-files). |
| Migration report | The HTML file that each `migrate` run writes. See [Step 6](#step-6-read-the-migration-report). |

A statement that names only OVE, OVC4, OVC10 or OV Terra applies to that product only.

### Officially Validated Migration Matrix

| Source | Target |
|---|---|
| OVE | OV Terra |
| OVE | OVC10 |
| OVC4 | OV Terra |
| OVC4 | OVC10 |

- OV4 (OVE or OVC4): 4.9R3 with the official OV4 4.9R3 patch applied ([Prerequisites](#prerequisites)).
- OV10 (OVC10 or OV Terra): 10.6.1.

> [!NOTE]
> The migration tool supports OVE Standalone as well as OVE HA installations.

> [!IMPORTANT]
> **Not supported in 1.0.3:** AOS 6x switches (OS6350, OS6450) and third-party devices. OV10 manages them over SNMP, and OV4 4.9R3 does not expose their SNMP credentials, so the tool cannot migrate them. This is an OV4 limitation, not a migration tool error. Support is planned for a later version. Until then, you can add the AOS 6x switches to OV10 by hand, with their SNMP credentials, or keep them managed by OV4: see [Add AOS 6x Switches to OV10 by Hand](#add-aos-6x-switches-to-ov10-by-hand).

### Files You Need

Put these files in one folder on the computer that runs the tool:

| File | Where | What it is |
|---|---|---|
| `migration_tool.exe` | [Release assets](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest) | The tool. Run it in PowerShell, from this folder. |
| `config.toml` | This repository | The configuration template: copy it to `config.local.toml` ([Step 1](#step-1-fill-in-configlocaltoml)). |
| `ov4APGroupToOV10SiteMapping.csv` | This repository | The optional AP Group to Site mapping ([Step 4](#step-4-optional-place-ap-groups-in-several-ov10-sites)). |
| `ovshared-4.9R3.jar` | [Release assets](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest) | The official OVE 4.9R3 patch, for OVE sources only ([Prerequisites](#prerequisites)). It is installed on the OVE server, not run from this folder. |
| `README.pdf` | [Release assets](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest) | This guide, as a PDF. |
| `sha256sum.txt` | [Release assets](https://github.com/ale-nsa-team/OmniVista-Migration-Tool/releases/latest) | The SHA-256 checksum of each release asset. |

To check that the release assets arrived intact, run this in PowerShell, in that folder. Each file should show `OK`. A file that shows `FAILED` is damaged: download it again.

```powershell
Get-Content sha256sum.txt | ForEach-Object {
    $hash, $name = $_ -split "\s+", 2
    if ((Get-FileHash $name -Algorithm SHA256).Hash -eq $hash) { "${name}: OK" } else { "${name}: FAILED" }
}
```

## Plan Your Migration

### The Two Phases

The migration is split into two phases to minimize risk and network downtime:

| Phase | Maintenance window | What you do | Steps |
|---|---|---|---|
| Phase 1: Configuration Migration | Not required | Collect from OV4, migrate the configuration without the devices, and review it on OV10. The devices stay on OV4 and the network is not affected. | 1 to 7 |
| Phase 2: Device Migration | **Required** | Save the AP configurations, migrate one device as a test, then the others in batches, then the settings that wait for the devices. | 8 to 12 |

### What Is Migrated, in Short

| Area | Migrated | Not migrated, or migrated with limits |
|---|---|---|
| Wireless | AP Groups, provisioning templates, RF profiles used by an AP Group, SSIDs, WLAN services, mesh, and the IP mode, IoT radio, radio and QoE/RTLS settings of each AP | Other per-AP overrides, floor plans |
| Switches | Switch templates, value mappings, Management User Templates | The Management User Templates that use SNMP credentials, which OV4 does not expose. The OV4 addresses configured on the switches, such as the SNMP trap receiver and the sFlow collector: set them again for OV10 |
| Authentication and access | AAA servers and profiles, UPAM (access policies, guest and BYOD access strategies, External RADIUS servers, captive portal templates, guest service levels, Guest Access global configuration), the UPAM External LDAP server (OVE), Access Role Profiles, Access Auth Profiles, tunnel profiles, policies | The TLS of the External LDAP server, the content of captive portal pages, Web Content Filtering, the ACE server, the AAA global setting |
| Users | Employee accounts, guest accounts, company properties | Expired guest accounts, guest accounts that share a phone number or email, guest operators, switch user accounts, OV4 users, roles and groups |
| Devices | APs, AOS 8 switches | In 1.0.3, AOS 6x switches (OS6350, OS6450) and third-party devices: OV4 does not expose their SNMP credentials ([Legacy OmniSwitch](#legacy-omniswitch-os6350--os6450)). The switch models that OV10 does not support, such as OS2260 and OS6850 |
| Other | IoT/location servers, VPN settings, certificates, IoT categories, inactivity timeout, DHCP Option 82, WIPS policy, upgrade and backup schedulers, event responders | History, reports, CLI scripts, discovery profiles, topology, analytics, trap configuration |

The details are in [What Gets Migrated](#what-gets-migrated) and [What's Not Migrated](#whats-not-migrated).

### Prerequisites

#### On OV4

- **OVE:** install the official OVE 4.9R3 patch on the OVE server before you collect. It is attached to the release as `ovshared-4.9R3.jar`; it is also in the patch repository.
- **OVC4:** version 4.9.3 GA.
- **An API token:** Security > External Apps in the OV4 UI. It goes into `config.local.toml` ([Step 1](#step-1-fill-in-configlocaltoml)).

Before you start the migration, also do these, if they apply to you:

| If you migrate | Do this on OV4 first | Details |
|---|---|---|
| AOS 6x switches (OS6350, OS6450) or third-party devices | Not supported in 1.0.3, because OV4 4.9R3 does not expose their SNMP credentials: leave them out of the device migration. Add the AOS 6x switches to OV10 by hand, or keep them managed by OV4. Have their SNMP credentials ready. | [Add AOS 6x Switches to OV10 by Hand](#add-aos-6x-switches-to-ov10-by-hand) |
| An OVE UPAM External LDAP server that uses a certificate | Record its TLS setting and port, and keep a copy of its LDAPS certificate files and of its Trust CA. The tool migrates it without TLS. | [UPAM External LDAP Servers That Use a Certificate](#upam-external-ldap-servers-that-use-a-certificate-ove) |
| An RF profile that no AP Group uses | Assign it to an AP Group, or recreate it on OV10 later: only the RF profiles used by an AP Group are collected. | [What Gets Migrated](#what-gets-migrated) |

#### On OV10

The tool creates the RF profiles, provisioning templates, AP Groups, Management User Templates and the other configuration objects itself: do not create them on OV10 beforehand. What must exist or be set on OV10 before you migrate:

| Item | Where | Needed for |
|---|---|---|
| The organization and the Site(s) the objects go to | OV10 | Every run: `org_id`, `site_id`, and the Sites of [Step 4](#step-4-optional-place-ap-groups-in-several-ov10-sites) |
| An application, with its `app_id` and `app_secret` | User menu > Applications (see below) | Every run |
| The application created by a user with write access to the organization and the Sites | OV10 | Every run. The pre-flight check stops on a missing permission. |
| The firewall ports between the devices and OV10 | Your network (see below) | Device migration |
| Auto-assign license for new devices enabled | Organization settings (see below) | AP migration |
| A Management User Template with the switch credentials | OV10 switch configuration | Only when you pass `--mgmt-template` (or set `mgmt_template`): the tool refers to it by name and does not create it. |
| No External LDAP server | OV10 | Only when you migrate the UPAM External LDAP server (OVE): OV10 holds only one External LDAP server per organization, so the tool cannot create the OVE one while OV10 has another ([details](#external-ldap-server-fails-with-the-maximum-number-of-external-ldap-server-is-reached)). |

**The OV10 application.** Create it at `[OV10_BASE_URL]/user/applications`. In the OV10 UI it is the Applications entry of the user menu, at the top right (the menu shows your user name, for example admin). Create it with a user who has write access to the organization and the Sites. The application gives `app_id` and `app_secret`; you also need the `org_id` and the `site_id` (24 hexadecimal characters, for example `6911b88a6b47ba847ef69f34`). `migrate` authenticates with the application only: `config.local.toml` holds no OV10 email or password.

**Firewall ports.** Open the required ports between your network devices and OV10 (the OVC10 cloud, or your OV Terra server) before you start. The full list of ports and protocols is in the [OVC10 Firewall Requirements](https://docs.ovcirrus.com/ov/network-prerequisites).

**Auto-assign license for new devices.** For AP migration, enable it at the OV10 organization level, or assign licenses to the migrated APs by hand afterwards. When it is disabled, migrated APs can appear as registered but receive no license automatically and may fail to connect. The tool checks this setting before it migrates and warns when it is disabled, without stopping.

#### On the Computer That Runs the Tool

- Windows, with PowerShell.
- HTTPS access to OV4 and to OV10, on port 443 or on the port of their `base_url`. Use a VPN if you work remotely.
- The [files you need](#files-you-need) in one folder. You run every command from that folder.

## Quick Start

The short path. Each item names its step and links to it: read the step the first time, because a value copied wrongly, such as a URL, an id or a token, stops the tool.

**Phase 1, without a maintenance window:**

1. [Prerequisites](#prerequisites): on OVE, install the official 4.9R3 patch. On OV4, create an API token under Security > External Apps. On OV10, create the application and enable Auto-assign license for new devices.
2. [Step 1](#step-1-fill-in-configlocaltoml): in the tool folder, copy `config.toml` to `config.local.toml` and fill it in: your own `secret_key_pass`, `[ov4]` with the OV4 URL and token, and `[ov10]` with the OV10 values. The host name of an OVE must not contain `ovcirrus.com`.
3. [Step 2](#step-2-collect-data-from-ov4): collect. With a self-signed certificate, such as the one of an OVE, the first run trusts it and saves its fingerprint in `config.local.toml`: there is nothing to copy.

   ```powershell
   .\migration_tool.exe collect --config-file config.local.toml --data-dir data
   ```

4. [Step 3](#step-3-check-the-collection): check the result of the collection.
5. [Step 4](#step-4-optional-place-ap-groups-in-several-ov10-sites), optional: place AP Groups in several OV10 Sites.
6. [Step 5](#step-5-migrate-the-configuration): migrate the configuration without the devices.

   ```powershell
   .\migration_tool.exe migrate --config-file config.local.toml --data-dir .\data --skip-devices
   ```

7. [Step 6](#step-6-read-the-migration-report) and [Step 7](#step-7-review-the-configuration-on-ov10): read the migration report, then review the configuration on OV10.

**Phase 2, in a maintenance window:**

1. [Before You Migrate Devices](#before-you-migrate-devices): save the running configuration of the APs, and remove the OV4 settings from the switches.
2. [Step 8](#step-8-test-with-one-device) and [Step 9](#step-9-migrate-the-remaining-devices-in-batches): remove one device from OV4 and migrate it as a test, then do the same for the others, in batches.

   ```powershell
   .\migration_tool.exe migrate --config-file config.local.toml --snapshot-file data\device_data.json --serial <serial number>
   ```

3. [Step 10](#step-10-migrate-the-ap-mesh-configuration) and [Step 11](#step-11-migrate-the-ap-settings-and-schedulers-that-waited-for-devices): migrate the mesh configuration, then the AP settings and schedulers that waited for the devices.
4. [Step 12](#step-12-point-the-switches-to-ov10): point the switches to OV10.

## Phase 1: Configuration Migration (No Maintenance Window)

Every step of this phase is non-disruptive: the devices stay managed by OV4, and you can do it during normal business hours.

### Step 1: Fill In config.local.toml

1. Open PowerShell in the folder that holds the tool.
2. Copy the template:

   ```powershell
   Copy-Item config.toml config.local.toml
   ```

3. Open `config.local.toml` in a text editor and fill in your values.

The block below is the shipped `config.toml`, line for line: your `config.local.toml` starts as this copy. Every optional key is commented out and has a working default: see [Configuration File](#configuration-file) for the full list.

```toml
# Migration Tool::Configuration
# Copy this file to config.local.toml and update with your actual configurations

# Lines that start with # are ignored: to set an optional key, remove the leading # and change the value.

# OV4 is the source server, OVE or OVC4; OV10 is the target server, OVC10 or OV Terra.
# Before 1.0.3 the sections [ov4] and [ov10] were named [ovc4] and [ovc10]; those names still work.

# Secret key pass: a passphrase you choose - not an OVE/OVC4/OV10 password. collect encrypts the
# passwords in the collected files with it, and migrate needs the same value to decrypt them.
secret_key_pass = "your-secret-key-password"
# The directory where collected data will be saved (default: the current folder)
# data_dir = "."
# The one snapshot file migrate sends when neither --data-dir nor --snapshot-file is given
# snapshot_file = "data/ap_group_data.json"
# Minimal log level (CRITICAL, ERROR, WARNING, INFO, DEBUG; default: INFO)
# log_level = "INFO"

# Certificate fingerprints of servers whose certificate no trusted CA signed, such as an OVE,
# which uses a self-signed certificate by default. You do not need to fill them in: the tool reads
# the certificate of the server, trusts it, and writes its line here (ov4 = "..." or ov10 = "...").
# When the certificate of a server changes, the tool replaces the line and logs a warning.
# Format: Compact hex (64 chars) or colon-separated
[trusted_certs]
# Source: OVE or OVC4
# ov4 = "a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456"
# Target: OVC10 or OV Terra
# ov10 = "f6e5d4c3b2a1098765432109876543210fedcba9876543210fedcba987654321"

# Source server: OVE or OVC4. The tool tells them apart by the host name of base_url: a host name
# under ovcirrus.com is an OVC4, any other is an OVE - so an OVE must not have ovcirrus.com in its
# host name.
[ov4]
base_url = "https://your-ove-or-ovc4-server.com"
token = "your-ove-or-ovc4-api-token"
# max_workers = 5 # Number of parallel workers for data collection (default: 5)
# account_processing_max_workers = 16 # Worker cap for guest/employee account processing (default: 16)
# collect_request_timeout_ms = 3000 # Socket timeout for OVE/OVC4 API during collection (default: 3000)
#                                   # A request that times out is sent twice more, with 2x then 4x this timeout

# Target server: OVC10 or OV Terra. migrate authenticates with the application (app_id and
# app_secret) only: no OV10 user email or password is needed.
[ov10]
base_url = "https://your-ov10-server.com"
org_id = "your-organization-id"
site_id = "your-site-id"
app_id = "your-app-id"
app_secret = "your-app-secret"
# skip_devices = true # Skip device migration when using --data-dir
# Optional: the Management User Template of every switch of the run, as --mgmt-template. It must
# already exist on OV10: the tool does not create it.
# mgmt_template = "My-SNMP-Template"
# Optional: place AP Groups (and their APs) in specific OV10 Sites.
# Defaults to ov4APGroupToOV10SiteMapping.csv in the current directory when present.
# mapAPGroupsToSites = "ov4APGroupToOV10SiteMapping.csv"

[migration]
# migration_request_timeout_ms = 30000 # Socket timeout for OV10 API (default 30000: OV10 sometimes takes more than 3 s)
# Guest Access global configuration and AAA profiles always wait at least 30000
# migration_request_max_retries = 3 # Attempts per OV10 request, the first one included (default: 3)
# migration_request_retry_backoff_ms = 500 # Base backoff for migration retries, exponential (default: 500)
# migration_request_interval_ms = 0 # Min delay between OVC10 migration requests (default: 0; for example 100 to reduce OV10 load)
# migration_parallel_standalone = true # Parallelize standalone object types only
# migration_max_workers = 8 # Worker threads for guest/employee account migration (default: 8, lower to reduce OVC10 load)
```

The required settings:

| Setting | What to put |
|---|---|
| `secret_key_pass` | A passphrase you choose. See [secret_key_pass](#secret_key_pass) below. |
| `[ov4] base_url` | The address of OV4, starting with `https://`. The tool tells OVE from OVC4 by its host name ([Step 2](#step-2-collect-data-from-ov4)). |
| `[ov4] token` | The API token from Security > External Apps in the OV4 UI. |
| `[ov10] base_url` | The address of OV10, starting with `https://`. |
| `[ov10] org_id` and `site_id` | The organization and the default Site, 24 lowercase hexadecimal characters each. Every AP Group goes to this Site unless you map it to another one ([Step 4](#step-4-optional-place-ap-groups-in-several-ov10-sites)). |
| `[ov10] app_id` and `app_secret` | The values of the OV10 application ([On OV10](#on-ov10)). |

Copy the token and the OV10 values exactly as the servers give them, with no extra space or quote.

Three rules of the file:

- A line that starts with `#` is a comment, and the tool ignores it. To set an optional key, remove the leading `#` and change the value. Editing a value on a line that still starts with `#` has no effect.
- Keep the global settings at the top, before the first `[section]` header. In TOML every setting belongs to the section above it: moved below a header, `secret_key_pass`, `data_dir` and `log_level` are read as part of that section and silently ignored.
- `[ov4]` always holds your source server, whether it is OVE or OVC4, and `[ov10]` your target server, whether it is OVC10 or OV Terra. The same goes for the `ov4` and `ov10` keys under `[trusted_certs]`.

A `config.local.toml` written for an earlier version works as it is: see [Files Written for an Earlier Version](#files-written-for-an-earlier-version).

#### secret_key_pass

`secret_key_pass` is a passphrase you choose. It is not an OVE, OVC4 or OV10 password, and it is not sent to any server.

- `collect` uses it to encrypt the passwords and secrets it writes into the snapshot files.
- `migrate` needs the same value to decrypt them. If it is lost or changed, collect the data again.
- The tool warns when it is still the example value of `config.toml`: anyone with the shipped `config.toml` could then decrypt the passwords in your snapshot files.

#### Certificates

You do not fill in `[trusted_certs]`. Before the first request, the tool reads the certificate of the server. When no CA that this computer trusts signed it, the tool trusts the certificate by its SHA-256 fingerprint, logs the fingerprint, writes it under `[trusted_certs]` in `config.local.toml` and goes on. An OVE (OV2500) uses such a self-signed certificate unless you installed another one:

```text
WARNING | The certificate of https://192.168.70.67 is not signed by a CA this computer trusts (OVE uses a self-signed certificate by default). It is trusted by its SHA-256 fingerprint:
WARNING |        Compact: A1B2C3...64 hex characters
INFO    | Saved ov4 = "A1B2C3...64 hex characters" under [trusted_certs] in config.local.toml.
```

- The next runs trust the saved fingerprint. The OV10 server is handled the same way on the first `migrate`, with an `ov10 = "..."` line.
- To make sure the certificate is the one of your server, compare the fingerprint in the log once with the SHA-256 fingerprint your browser shows for it.
- When the certificate of a server changes, for example after it is renewed, the tool logs a warning with the saved and the new fingerprint, trusts the new certificate and replaces the line.
- When the tool cannot write `config.local.toml`, for example because it is read-only, it trusts the certificate for that run only and logs the line to add.
- `--ov4-trusted-cert` and `--ov10-trusted-cert` still pass a fingerprint on the command line.

#### What the Tool Checks Before It Starts

Before `collect` and `migrate` send any request, the tool checks `config.local.toml`:

| Problem | Result |
|---|---|
| `[ov4] base_url`, `[ov4] token`, `[ov10] base_url`, `app_id` or `app_secret` still holds the example value from `config.toml` | The tool stops, with the name of the setting. |
| A `base_url` does not start with `https://`, or `org_id` or `site_id` is not 24 lowercase hexadecimal characters | The tool stops, with the name of the setting. |
| `secret_key_pass` is still the example value | The tool warns, and goes on. |
| A setting that is no longer used: `device_source`, or `email` and `password` under `[ov10]` | The tool warns, ignores it, and goes on. Remove it. |

Then it connects to the server, and says what is wrong when it cannot reach it: see [Connection and Certificates](#connection-and-certificates).

### Step 2: Collect Data From OV4

The same command collects from OVE and from OVC4:

```powershell
.\migration_tool.exe collect --config-file config.local.toml --data-dir data
```

**OVE or OVC4.** The tool tells them apart by the host name in `[ov4] base_url`. A host name that contains `ovcirrus.com`, such as `eu.manage.ovcirrus.com`, is an OVC4; any other host name or IP address is an OVE. The log shows which one it detected, in a line starting with `Source server: OVE` or `Source server: OVC4`. OVE and OVC4 return their device inventory from different APIs, and the tool uses the API of the detected one.

> [!WARNING]
> Do not give an OVE a host name that contains `ovcirrus.com`. The tool would take it for an OVC4 and ask it for its devices through the OVC4 API: collect succeeds, but `device_data.json` holds no devices. Use the IP address of the OVE, or another host name.

**What collect writes.** One snapshot file per group of objects, in the `--data-dir` folder, here `data`. The passwords and secrets in them are encrypted with your `secret_key_pass`. The list of files is in [Snapshot Files](#snapshot-files).

- Collect again whenever you need to, for example after you fix something on OV4: it overwrites the snapshot files in place.
- Do not run `collect` and `migrate` at the same time on the same folder.
- A request that OV4 is slow to answer is sent again by itself. If collection still fails with `The read operation timed out`, see [Collection Fails With "The read operation timed out"](#collection-fails-with-the-read-operation-timed-out).

### Step 3: Check the Collection

Do not migrate before you have checked the result of the collection. It ends with a summary:

```text
Data collection completed:
  [OK] Successful: 18
  [FAIL] Failed: 2
```

If a collector failed:

1. Find in `migration_tool.log` which objects failed and why ([Read the Log](#read-the-log)).
2. Choose one of these options:
   - **Contact Tech Support** when the failures involve complex objects or several dependencies. Have them resolve the issues before you migrate.
   - **Create the objects on OV10 by hand** when only a few simple objects failed, for example 1 to 3, before you migrate.
3. Do not migrate until every failed object is collected (collect again after the fix), created on OV10 by hand, or confirmed as not needed.

Failed collection means incomplete data. Migrating incomplete data can make other objects fail for a missing dependency, leave the network configuration incomplete, and leave work to redo by hand.

**Expected differences with OV4.** These are not failures:

| What you see | Why | Details |
|---|---|---|
| Fewer guest accounts in `guest_account_data.json` than on OV4. The log says `<count> of <total> Guest Accounts are not collected because they have expired on OVE`. | OV10 does not accept an expired guest account, so collect leaves it out. The guest account collector still counts as successful. | [Expired Guest Accounts](#expired-guest-accounts) |
| `Certificate '<name>' (<type>) is not collected` | OV4 served an empty or unreadable file for that wireless certificate, often because it has expired. | [Certificate Not Collected](#certificate-not-collected) |
| Fewer RF profiles than on OV4 | Only the RF profiles used by an AP Group are collected. | [What Gets Migrated](#what-gets-migrated) |

### Step 4 (Optional): Place AP Groups in Several OV10 Sites

Skip this step if all your AP Groups belong in one OV10 Site, and go to [Step 5](#step-5-migrate-the-configuration). Skipping it is safe: the mapping file ships with every sample row commented out, which means "put every AP Group in the default Site", as in earlier releases.

By default, every AP Group is created in the Site of `site_id` in your config file. To spread your OV4 AP Groups across several OV10 Sites, fill in `ov4APGroupToOV10SiteMapping.csv`, in the same folder as `config.toml` and `migration_tool.exe`. Step 5 then places each AP Group in the Site you mapped it to.

#### 4.1 List Your AP Group Names

The mapping file is checked against the data you collected in [Step 2](#step-2-collect-data-from-ov4). The AP Group names are in `ap_group_data.json`, in the `name` field of each entry of the `AP_GROUP` block. To list them from PowerShell:

```powershell
$data = Get-Content .\data\ap_group_data.json -Raw | ConvertFrom-Json
($data | Where-Object { $_.type -eq 'AP_GROUP' }).data.name
```

Copy the names exactly as they appear, including spaces, trailing characters and symbols such as `*`.

#### 4.2 Find Each OV10 Site Id

1. Log in to OV10 and select the Site in the Site selector.
2. Read the address bar: the Site Id is the 24-character lowercase hexadecimal string in the URL, for example `6a01822e47e327655a5b4979`.
3. Repeat for every Site you plan to use.

A Site Id is always 24 characters of `0-9` and `a-f`. A Site name does not work.

#### 4.3 Fill In the Mapping File

Edit `ov4APGroupToOV10SiteMapping.csv`. Keep the header row exactly as shipped, then add one row per AP Group you want to place:

```csv
OV4 AP Group Name,OV10 Site Id
Corp-Wireless,6a01822e47e327655a5b4979
Guest-Wireless,6a01822e47e327655a5b4980
"Warehouse-APs, Legacy",6a01822e47e327655a5b4980
"#Staging-APs",6a01822e47e327655a5b4979
```

Rules for the file:

- **One Site per AP Group.** Listing the same AP Group twice aborts the migration: the file places a group, it does not copy it into several Sites.
- **You do not have to list every group.** An AP Group that is not listed is created in the `site_id` of your config file.
- **Names are matched exactly:** spaces, case and characters such as `*` all count.
- Quote a name that contains a comma: `"Warehouse-APs, Legacy",<SiteId>`.
- Quote a name that starts with `#`: `"#Staging-APs",<SiteId>`. Unquoted, the line is read as a comment and silently ignored.
- Double each double quote in a name: write `""` for each `"`, as CSV requires.
- Lines starting with `#` are comments, and blank lines are ignored. Excel, LibreOffice and Notepad files are all accepted: a UTF-8 byte-order mark and Windows line endings are handled.

#### 4.4 Use Another File Name or Folder

To keep several mapping files, for example one per customer site survey, name the file to use. The tool looks for the mapping file in this order, and the first one found wins:

1. `--map-ap-groups-to-sites <path>` on the command line.
2. `mapAPGroupsToSites = "<path>"` under `[ov10]` in the config file.
3. `ov4APGroupToOV10SiteMapping.csv` in the folder you run the command from, if it exists.

```powershell
# Command line
.\migration_tool.exe migrate --config-file config.local.toml --data-dir .\data --skip-devices `
  --map-ap-groups-to-sites .\my-site-mapping.csv
```

```toml
# Or in config.local.toml
[ov10]
mapAPGroupsToSites = "my-site-mapping.csv"
```

A file you name in 1 or 2 must exist: a typo in the path aborts the migration, instead of silently putting every AP Group in the default Site.

#### 4.5 What Follows an AP Group Into Its Site

| Object | Site it is created in |
|---|---|
| AP Group | The mapped Site |
| Provisioning Template of that AP Group | The Site of its AP Group |
| AP devices whose group is mapped | The Site of their AP Group |
| Mesh, AP IP Mode, AP IoT Radio, AP Radio / QoE settings | The Site where that AP was actually onboarded |
| SSID device-group assignments | The Site of the group they refer to |
| Scheduled backups and scheduled upgrades | Always the `site_id` of the config file, by design |
| RF Profiles, certificates, AAA, UPAM, policies and the other organization-level objects | Not Site-scoped: the file does not affect them |

If an AP Group already exists on OV10 in a different Site than the one you mapped it to, the tool follows the Site that OV10 reports for that group, so that the group of the AP stays valid, and logs a warning naming both Sites.

#### 4.6 Errors in the Mapping File

`migrate` checks the mapping file before it migrates anything. It reports every problem of the file in one run, each with its line number, and then aborts the whole migration: no object of any kind is created, so you can fix the file and run again.

| Log message | What to fix |
|---|---|
| `invalid header ... Expected 'OV4 AP Group Name,OV10 Site Id'` | The header row was edited or removed: restore it exactly. |
| `expected 2 columns ... got N` | A row has a missing or extra comma: quote the names that contain a comma. |
| `'<value>' is not a valid OV10 Site Id` | It is not 24 lowercase hexadecimal characters: copy it again from the OV10 URL. |
| `AP Group '<name>' is already mapped on line N` | The same AP Group appears twice: keep one row. |
| `AP Group '<name>' is not present in the collected OVC4 data` | The name is misspelled, or the data was collected before the group existed: fix the name, or collect again. The message says OVC4 for an OVE source too. |
| `AP Group to Site mapping references Site(s) that do not exist` | The Site Id is not in this organization: the log lists the Sites that are. |
| `User does not have access to site '<id>'` | The OV10 user who created the application of `app_id` needs Site admin rights on every mapped Site. |
| `AP Group to Site mapping file not found` | The path given on the command line or in the config file is wrong. |

### Step 5: Migrate the Configuration

Migrate every configuration object now, and leave the devices for Phase 2:

```powershell
.\migration_tool.exe migrate `
  --config-file config.local.toml `
  --data-dir .\data `
  --skip-devices
```

- `--data-dir` sends every snapshot file of the folder, in the order of [Snapshot Files](#snapshot-files). `--skip-devices` leaves out the devices.
- If you filled in the mapping file in [Step 4](#step-4-optional-place-ap-groups-in-several-ov10-sites), no extra option is needed: the tool picks it up when it is in the folder you run the command from, the tool folder if you follow this guide, and creates each listed AP Group in its Site. Run from another folder, the tool does not find it, and creates every AP Group in the `site_id`. For a file with another name or in another folder, see [4.4](#44-use-another-file-name-or-folder).
- Always give `--data-dir` or `--snapshot-file`. With neither, `migrate` takes `snapshot_file` or `data_dir` from the config file, and a `data_dir` there makes it migrate every snapshot file, devices included.
- A request that OV10 is slow to answer is sent again by itself, up to three attempts in all. An object that OV10 never answers holds the run for about 90 s: the tool has not stopped. See [Migration Fails With "Request timeout: The read operation timed out"](#migration-fails-with-request-timeout-the-read-operation-timed-out).

### Step 6: Read the Migration Report

Every run ends with:

- a Migration Summary table at the end of `migration_tool.log`;
- an HTML report in a `reports` folder next to the snapshot files, here `data\reports\migration_report_<timestamp>.html`.

Both list, per object type, what was created, what failed with its error, and what was not sent. Resolve every failure before you continue: look up its message in [Troubleshooting](#troubleshooting).

#### Outcomes

Each object ends in one of four outcomes:

| Outcome | What it means | Counted as failed |
|---|---|---|
| Migrated | Created on OV10, or already there. | No |
| Failed | OV10 rejected it, or an error stopped it. Its error says why. | Yes |
| Not Supported by OV10 | OV10 does not support it as it is on OV4, or it requires an object that OV10 does not support, so the tool did not send it. This is an OV10 limitation, or an OV4 bug where the reason says so, not a migration tool error. The list is in [Not Supported by OV10](#not-supported-by-ov10). | No |
| Waiting for Devices on OV10 | A setting of a device that OV10 does not manage yet: the IP mode, IoT radio, radio or QoE/RTLS settings of an AP, a mesh setting, or a backup or upgrade scheduler. It migrates once its device does: migrate the devices (Phase 2), then the same snapshot file again ([Step 11](#step-11-migrate-the-ap-settings-and-schedulers-that-waited-for-devices)). | No |

The success rate counts only the objects that were sent. The HTML report has a **Not Supported by OV10** section and a **Waiting for Devices on OV10** section, each grouping the objects by reason, with what to do; the log ends with a warning line per reason.

#### Needs Attention

An object can be migrated and still need a step by hand, because the tool changed it so that OV10 accepts it: for example a UPAM External LDAP server created without TLS, a Data VPN Server whose IP Range was widened, or a tunnel profile that ended at a Data VPN Server on OV4. It counts as Migrated, and the HTML report also lists it under **Needs Attention**, with what to finish by hand ([Step 7](#step-7-review-the-configuration-on-ov10)).

#### Knock-On

An object that requires another object which was not migrated cannot be migrated either, for example an SSID whose access policy or Access Role Profile was not migrated. Its message starts with `Knock-on:`, names the object it requires and, along a chain, the first object of the chain, which is the one to fix. For example:

```text
Knock-on: requires ACCESS_POLICY 'Upgrade-Test', which was not migrated because GUEST_ACCESS_STRATEGY 'Upgrade-Test' is not supported by OV10 (see its own entry). Once ACCESS_POLICY 'Upgrade-Test' is migrated, migrate this object again.
```

- When the first object of the chain is not supported by OV10, the knock-on is listed under Not Supported by OV10 too, and is not counted as failed.
- When it failed, the knock-on is failed too. The report shows it in amber under Failed Items, and the log line `Failed: <n> (<k> as a knock-on of an object they require)` counts it.
- An object of another snapshot file that failed, such as an Access Role Profile, may already exist on OV10 from an earlier run, so the SSID that uses it is still sent; only if the SSID fails too is it reported as a knock-on.

Fix the first object of the chain, then migrate the knock-ons again.

#### Running migrate Again

You can migrate the same snapshot file again, for example after you fix a failure: the tool sends every object of the file again.

- When OV10 answers that an object already exists, the tool counts it as Migrated and leaves it as it is on OV10: it does not update it. To send a changed object again, delete it on OV10 first, or make the change on OV10 by hand.
- The UPAM External LDAP server is the exception: OV10 holds only one, and answers a second one with `The maximum number of External LDAP Server is reached` instead of saying that it exists. Migrating `upam_ldap_data.json` again therefore reports the server as Failed, although the earlier run created it ([details](#external-ldap-server-fails-with-the-maximum-number-of-external-ldap-server-is-reached)).
- The settings that OV10 holds once are written again on every run: the inactivity timeout, DHCP Option 82, the WIPS policy, the Guest Access global configuration and the per-AP settings.

#### AP Group Placement

If you filled in the mapping file in [Step 4](#step-4-optional-place-ap-groups-in-several-ov10-sites), a successful run records the placement in two places:

- the log ends with an `AP Group -> Site placement` block, listing every mapped group and the default Site used for the others;
- the HTML report has an **AP Group → Site Placement** table, naming the file the mapping came from.

Check one AP Group in the OV10 UI as well, to confirm that it is where you expected, before you move on to device migration.

### Step 7: Review the Configuration on OV10

Before you migrate any device, log in to OV10 and check the migrated configuration:

- AP Groups, RF Profiles, Provisioning Templates
- SSIDs and WLAN Services
- AAA Servers, AAA Profiles
- The UPAM External LDAP server (OVE only), migrated as non-secure (NS)
- Access Role Profiles, Policies
- Management User Templates, under the switch configuration
- Any other object that matters for your deployment

Then finish what the tool cannot do, while no device depends on it yet. Manual corrections are much easier before the devices are onboarded.

| What | When | What to do |
|---|---|---|
| Objects listed under Needs Attention | Always | Do what the report says for each one. |
| The UPAM External LDAP server | It used a certificate on OVE | Restore TLS by hand: [UPAM External LDAP Servers That Use a Certificate](#upam-external-ldap-servers-that-use-a-certificate-ove). |
| Data VPN Servers whose IP Range was widened | Listed under Needs Attention | Check that VPN clients may use the added addresses: [Data VPN Settings](#data-vpn-settings-data_vpn_setting). |
| Tunnel profiles that ended at a Data VPN Server on OV4 | Listed under Needs Attention | The tool sent the Server VPN IP of that server: link the tunnel to the Data VPN Server on OV10 if its traffic must go through the VPN ([Tunnel Profiles That End at a Data VPN Server](#tunnel-profiles-that-end-at-a-data-vpn-server)). |
| The external antenna gain of the RF profiles | Always, in particular where OV4 had 0 | The tool sends a value below 1.0 as 1.0 and above 16.0 as 16.0, without logging it, and no gain for a profile that uses the default settings: [RF Profile External Antenna Gain](#rf-profile-external-antenna-gain). |
| Objects listed under Not Supported by OV10 | Always | Follow the row of [Not Supported by OV10](#not-supported-by-ov10). |
| Management User Templates listed as Failed | They use SNMP credentials on OV4 | Expected in 1.0.3: OV4 4.9R3 does not expose the SNMP credentials ([Legacy OmniSwitch](#legacy-omniswitch-os6350--os6450)). If you need such a template on OV10, create it there by hand. |
| Captive portal pages | You use captive portals | Recreate the texts, images and logos of the pages: only the name, description and layout of each template are migrated. |
| Objects that the tool does not migrate | You use them | Recreate them by hand: [What's Not Migrated](#whats-not-migrated). |

## Phase 2: Device Migration (Maintenance Window Required)

Perform this phase during a scheduled maintenance window. You save the configuration of the APs, remove each device from OV4, migrate it, and check it. Start with one device, then go on in batches.

### Before You Migrate Devices

#### Precautions

- **Migrate an AP only once its AP Group is on OV10.** Step 5 creates the AP Groups; device migration does not. An AP whose AP Group is missing fails with `No device group with name: <group> in OVC10`.
- **Remove a device from OV4 before you migrate it.** Migrating a device that OV4 still actively manages may leave the AP in an inconsistent state. Make sure the device is ready to be cut over.
- **`--skip-devices` does not protect `--snapshot-file`.** It applies only with `--data-dir`, and is silently ignored otherwise: it does not keep you from migrating devices by accident.
- **Give `migrate` its snapshot file.** Every command of this phase names one with `--snapshot-file`. Without `--snapshot-file` or `--data-dir`, `migrate` takes them from the config file ([Step 5](#step-5-migrate-the-configuration)).

#### Save the AP Configurations

> [!CAUTION]
> Before you delete APs from OV4, save their running configuration, so that it is not lost.

1. Log in to the OV4 web interface.
2. Go to **Network > Managed Devices**.
3. Select the APs you want to migrate, or all APs.
4. Open the **Actions** menu.
5. Select **Save to Running**, which saves the configuration to persistent storage.
6. Wait for the confirmation that the configurations are saved ("Configuration Saved").

The saved configuration protects the network in these cases:

- **The AP reboots during the migration:** if an AP reboots before it is onboarded to OV10, it boots from the saved configuration and keeps advertising its SSIDs.
- **Onboarding problems:** if onboarding to OV10 runs into problems, the AP keeps serving the network with its saved configuration.
- **Call home after deletion:** when you delete an AP from OV4 and it calls home to the Activation Server, the server may answer "I don't know you", which can cause the configuration to be lost. The saved configuration prevents the service disruption.

Save again any AP whose configuration changes before you migrate it.

#### Remove the OV4 Settings From the Switches

OV4 pushes its own management or VPN IP to the switches, as the destination of their monitoring and analytics traffic. After the migration, these entries still point to the OV4 IP, which is no longer valid: traffic is misdirected, traps are not delivered, and the old system takes needless load. The migration tool does not change them.

While OV4 still manages the switches, before you delete them from OV4, remove these settings from the switches:

| Setting | What OV4 configured on the switch |
|---|---|
| SNMP Trap | OV4 as an SNMP trap receiver: the OV4 trap station VPN IP. |
| sFlow | The sFlow collector IP, the OV4 VPN IP, and its sampling settings. |
| Application Visibility | The analytics/DPI collector endpoint. |
| Policy Server | OV4 as the OmniVista policy server, used for Unified Policy enforcement: the policy server IP points to the OV4 VPN IP. |

After the migration, configure them again with the OV10 address. The UPAM/authentication-server IP of the UNP/UPAM profiles also points to OV4: you point it to the OV10 UPAM server after the migration ([Step 12](#step-12-point-the-switches-to-ov10)).

#### Choose the Switch Management User Template

Each switch is migrated with a Management User Template, the `mgmtUsersTemplate` of the snapshot:

- a template migrated from OV4 in Step 5;
- an `AUTO-SNMP-<hash>` template that `collect` built from the SNMP credentials of the switch; with OV4 4.9R3, which does not expose them, `collect` builds none ([Legacy OmniSwitch](#legacy-omniswitch-os6350--os6450));
- or the OV10 `Default Management User Template`.

`migrate` never invents a template: the only templates that device migration creates are the `AUTO-SNMP-<hash>` templates that `collect` wrote into `device_data.json`.

To use another template, pass `--mgmt-template "<name>"`, or set `mgmt_template` under `[ov10]` in `config.local.toml`. It names a template that already exists on OV10: the tool does not create it.

- One run uses one template for every switch it migrates.
- To use different templates for different groups of switches, run `migrate` once per group, each with its own `--serial` and `--mgmt-template`.

#### Legacy OmniSwitch (OS6350 / OS6450)

> [!IMPORTANT]
> **Not supported in 1.0.3.** Migration tool 1.0.3 does not migrate AOS 6x switches (OS6350, OS6450) or third-party devices, because OV4 4.9R3 does not expose their SNMP credentials. This is an OV4 limitation, not a migration tool error. Support is planned for a later version. Until then, you can add the AOS 6x switches to OV10 by hand, or keep them managed by OV4: see [Add AOS 6x Switches to OV10 by Hand](#add-aos-6x-switches-to-ov10-by-hand).

**Why OV10 needs the SNMP credentials.** OV10 10.6.1 supports OS6350 and OS6450 switches running AOS 6.7.2.222.R08 or later as Legacy OmniSwitch devices. They have no ovng-agent: OV10 manages them only over SNMP, with the SNMP credentials of their Management User Template. Third-party devices (OVE) are managed over SNMP the same way; OVC10 does not support third-party devices, OV Terra does. Earlier OV10 versions, such as 10.5.2, do not support AOS 6x switches: there, only APs and the AOS switches whose agent calls home can be added, and those do not depend on SNMP credentials.

**Why the tool cannot migrate them.** The tool takes the SNMP credentials from OV4. During `collect`, it reads the SNMP profile of each device, and for a device whose SNMP credentials are complete, it writes a template named `AUTO-SNMP-<hash>` into `device_data.json`; devices with the same credentials share one. When the devices migrate, the tool creates that template on OV10 just before the first switch that uses it. SNMPv3 profiles with `authNoPriv` (username and auth password) or `authPriv` are valid for the template. OV4 4.9R3 does not expose the SNMP credentials, so:

- `collect` builds no `AUTO-SNMP-<hash>` template;
- the Legacy OmniSwitches fail at `migrate`, with `!! LEGACY SWITCH WARNING — ACTION REQUIRED !!` in the log, and the third-party devices fail with `Third-party device '<name>' does not include a Management Users Template`;
- a Management User Template that uses SNMP credentials (`USE_SNMP`) on OV4 fails in Step 5, even when you migrate no AOS 6x switch.

##### Add AOS 6x Switches to OV10 by Hand

OV10 10.6.1 can manage AOS 6x switches: only the migration tool cannot move them, because it cannot read their SNMP credentials from OV4. You know these credentials, so you can add the switches to OV10 yourself, during the maintenance window:

1. Upgrade the switches on the GA build 6.7.2.221.R08 to 6.7.2.222.R08 or later while OV4 still manages them: 6.7.2.221.R08 cannot be onboarded to OVC10.
2. Create a Management User Template with the SNMP credentials of the switches on OV10. Switches with the same SNMP credentials can share one template.
3. Remove the switches from OV4, as for the other devices ([8.2](#82-remove-the-device-from-ov4)).
4. Add the switches on OV10, with the template of step 2.

Do not migrate these switches with the migration tool: leave their serial numbers out of `--serial` in [Step 8](#step-8-test-with-one-device) and [Step 9](#step-9-migrate-the-remaining-devices-in-batches).

### Step 8: Test With One Device

Start with a single device, one AP or one switch, to check the whole process before you migrate the others.

#### 8.1 Select the Device

Select the device in one of two ways:

- **With `--serial` (recommended):** `migrate` takes the device from `device_data.json` by its serial number. There is no file to edit: go to [8.2](#82-remove-the-device-from-ov4).
- **With a separate snapshot file:** copy the entry of the device into a new file, as below.

Open `data\device_data.json` and find the entry of the device. Copy that object into a new snapshot file, keeping the same wrapper format. The examples below are shortened: copy the whole entry as it is in `device_data.json`, without retyping or removing fields.

For an AP, save as `data\SSZ194603166.json`:

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

For a switch, save as `data\P39Q0178.json`:

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

The AOS 6x switches are in the `LEGACY_SWITCH_DEVICE` block: do not migrate them with the tool in 1.0.3, add them by hand ([Add AOS 6x Switches to OV10 by Hand](#add-aos-6x-switches-to-ov10-by-hand)). When the `mgmtUsersTemplate` of a switch names an `AUTO-SNMP-<hash>` template, that template is in the `MGMT_USERS_TEMPLATE` block of `device_data.json`: copy its entry into a `MGMT_USERS_TEMPLATE` block of the new file too, or use `--serial`. Without it, the tool does not create the template, and the switch refers to a template that OV10 does not have.

#### 8.2 Remove the Device From OV4

1. For an AP, Save to Running again ([Save the AP Configurations](#save-the-ap-configurations)) if its configuration changed since you saved it.
2. Run Collect Support Info on the device in OV4 (**Administration > Audit > Collect Support Info**), and keep the output to compare later.
3. In OV4, go to **Network > Device Catalog**.
4. Find the device by its serial number and select **Delete** (or **Remove**).
5. Confirm the removal, and check that it completed before you migrate.

#### 8.3 Migrate the Device

With `--serial`:

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\device_data.json `
                             --serial SSZ194603166
```

A switch, with another Management User Template:

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\device_data.json `
                             --serial P39Q0178 `
                             --mgmt-template "Template-SW-Core"
```

With a separate snapshot file:

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\SSZ194603166.json
```

#### 8.4 Check the Device

1. **Update the software:** edit the device in OV10 and upgrade it to the latest available firmware, AWOS for an AP and AOS for a switch. Optional, but recommended.
2. **Compare the configuration:** run Collect Support Info on the device in OV10, and compare it with the output you saved in [8.2](#82-remove-the-device-from-ov4). A difference shows a configuration that was not applied correctly.
3. **Check the clients:** confirm that wireless clients (APs) or network connectivity (switches) work as expected.

If something is wrong, follow [Troubleshooting](#troubleshooting), starting with [Device Problems](#device-problems): compare the Collect Support Info outputs of OV4 and OV10 side by side to find the configuration differences.

If your network includes AOS switches, repeat this step with one switch before you go on.

### Step 9: Migrate the Remaining Devices in Batches

Once the test run is successful, migrate the remaining devices in batches. For each batch:

1. Save to Running on the APs of the batch, if their configuration changed since you saved it.
2. Delete the devices of the batch from OV4, as in [8.2](#82-remove-the-device-from-ov4).
3. Migrate the batch with `--serial`, followed by the serial numbers of the batch, separated by commas:

   ```powershell
   # The devices of the batch, by serial number
   .\migration_tool.exe migrate --config-file config.local.toml `
                                --snapshot-file data\device_data.json `
                                --serial SN000001,SN000002,SN000003
   ```

   ```powershell
   # A batch of switches with a specific Management User Template
   .\migration_tool.exe migrate --config-file config.local.toml `
                                --snapshot-file data\device_data.json `
                                --serial SW000101,SW000102,SW000103 `
                                --mgmt-template "Template-SW-Branch"
   ```

4. Check the network before you start the next batch.

Without `--serial`, `migrate` sends every device of `device_data.json`: those that OV4 still manages, those of the earlier batches again, and in 1.0.3 the AOS 6x switches and third-party devices, which the tool does not support ([Legacy OmniSwitch](#legacy-omniswitch-os6350--os6450)). Use it only once every remaining device is deleted from OV4, and prefer `--serial` for the last batch too:

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\device_data.json
```

### Step 10: Migrate the AP Mesh Configuration

The mesh configuration is in `data\mesh_data.json`. Migrate it once its APs are on OV10: the tool finds each AP by its serial number before it applies its mesh settings.

```powershell
.\migration_tool.exe migrate --config-file config.local.toml `
                             --snapshot-file data\mesh_data.json
```

Check that the mesh APs come up as expected. If an AP was not migrated yet, migrate that device first, then run the mesh migration again.

### Step 11: Migrate the AP Settings and Schedulers That Waited for Devices

In Step 5, the per-AP settings and the backup and upgrade schedulers that name devices could not be migrated, because their devices were not on OV10 yet: the report listed them under Waiting for Devices on OV10. Now that the devices are migrated, migrate their files again:

```powershell
.\migration_tool.exe migrate --config-file config.local.toml --snapshot-file data\ap_config_data.json
.\migration_tool.exe migrate --config-file config.local.toml --snapshot-file data\backup_scheduler_data.json
.\migration_tool.exe migrate --config-file config.local.toml --snapshot-file data\scheduler_upgrade_data.json
```

- Each run sends every object of the file again, also those that Step 5 migrated. Afterwards, check on OV10 that each backup and upgrade schedule is there once.
- An object still listed under Waiting for Devices on OV10 refers to a device that is not on OV10 yet: migrate that device, then its file again.

### Step 12: Point the Switches to OV10

Once the devices are on OV10:

1. Configure again the settings you removed in [Remove the OV4 Settings From the Switches](#remove-the-ov4-settings-from-the-switches), with the management or VPN IP of OV10: the SNMP trap destination, the sFlow collector, Application Visibility, and the policy server IP.
2. Point the UNP/UPAM profiles to the OV10 UPAM server. The switches configured for UNP/UPAM authentication still point to the UPAM/authentication-server IP of OV4: the migration tool moves the configuration data, but does not re-point the switches, so authentication keeps targeting the old, now invalid address.
   1. In OV10, go to **Configure > Network Access > Authen Servers**, and record the IP address and port of the UPAM server.
   2. Edit every UNP profile concerned, and set this IP address and port as its RADIUS / authentication server.
3. Finish the onboarding of the devices that need it. Some switch and AP models are not immediately managed, although the migration reports success: for example, update the call-home URL and trigger the call home of certain switch types, or update the DHCP / Activation Server settings of the APs and reboot them, where it applies.

## Troubleshooting

Find the message of the log or of the report in this chapter. The problems are grouped by when they happen: connecting, collecting, migrating the configuration, migrating the devices.

### Read the Log

The tool writes `migration_tool.log` in the folder you run it from, and adds each run at the end of the file. `collect` and `migrate` share it, so read the last run, from the bottom up. Each line starts with its date and time.

To see more, enable DEBUG logging in the config file:

```toml
log_level = "DEBUG"
```

or on the command line:

```powershell
.\migration_tool.exe collect `
  --log-level DEBUG `
  --config-file config.local.toml
```

At DEBUG level, the log shows each request sent to OV4 and OV10 and each answer, with their whole body, so it grows large: use DEBUG to diagnose a problem, then go back to INFO. Passwords, secrets (such as the OV10 appSecret and RADIUS shared secrets) and access tokens are written as `***`.

### Connection and Certificates

#### Pre-flight Check FAILED: Wrong Address or Port in base_url

Before the first request, the tool connects to the server of `[ov4] base_url` (collect) or `[ov10] base_url` (migrate), and says what is wrong when it cannot reach it over HTTPS. To tell a wrong port from a server that is down, it pings the host, so a host that blocks ping looks down.

| Log message | Meaning | What to do |
|---|---|---|
| `Cannot resolve the host name '<host>'` | The DNS server does not know the host name. | Fix the host name, or use the IP address. |
| `<host> is reachable, but port <port> refuses the connection` | The host answers, but no service listens on that port. | Fix the port in `base_url`. |
| `<host> answers ping, but port <port> does not answer` | The host is up, but the port is wrong, closed, or blocked by a firewall. | Fix the port, or open it in the firewall. |
| `<host> accepts connections on port <port>, but port <port> is not an HTTPS port` | Another service, for example plain HTTP, listens on that port. This is not a certificate problem. | Fix the port in `base_url`. |
| `Cannot reach <host>: port <port> does not answer, and <host> does not answer ping` | The host does not answer at all. | Check the address, the VPN and the firewall. |

When port 443 of the host answers HTTPS, the log also says so. For example, with `base_url = "https://192.168.70.67:9443"` for an OVE that answers HTTPS on 443: `Port 443 of 192.168.70.67 answers HTTPS: set [ov4] base_url to https://192.168.70.67 (without :9443)`.

#### Certificate Messages

| Log message | Meaning | What to do |
|---|---|---|
| `The certificate of ... is not signed by a CA this computer trusts` | Not an error: the server uses a self-signed certificate, as an OVE (OV2500) does by default. The tool trusts it by its SHA-256 fingerprint and saves it under `[trusted_certs]` in `config.local.toml`: `ov4 = "..."` on collect, `ov10 = "..."` on migrate. | Nothing. To be sure, compare the fingerprint once with the one your browser shows ([Certificates](#certificates)). |
| `The certificate of ... has changed since its fingerprint was saved` | The server presents another certificate than the saved one. The tool trusts the new certificate and replaces the line. | Check that the change is expected, for example a renewed certificate. If it is not, check the network between this computer and the server. |
| `The fingerprint is trusted for this run only` | The tool could not write `config.local.toml`, for example because it is read-only. | Add the line that the log gives under `[trusted_certs]`, or leave it: each run trusts the certificate again. |

#### Messages About the Config File

| Log message | What to do |
|---|---|
| `... is still the example value ... from config.toml` | You have not filled in that setting: set it in `config.local.toml` ([Step 1](#step-1-fill-in-configlocaltoml)). |
| `[ov4] base_url must start with 'https://'` (or `[ov10]`) | Write the address with `https://`. |
| `org_id must be a 24-character lowercase hex string` (or `site_id`) | Copy the id again from OV10. |
| `Missing required arguments for <collect or migrate> operation: ...` | Set the settings it names, in the config file or on the command line. |
| `Missing required source for migrate operation: must specify either --snapshot-file or --data-dir` | Add `--data-dir` or `--snapshot-file` to the command. |

#### Token Authentication Failure

- Generate the token again in the OV4 UI: Security > External Apps.
- OVC4: check that the token has the permissions it needs, and that it is copied without extra quotes or spaces.
- OVE: check that the token has not expired, and test it from PowerShell. `curl.exe` comes with Windows 10 and 11; `-k` accepts the self-signed certificate of the OVE:

  ```powershell
  curl.exe -k -H "Authorization: Bearer YOUR_TOKEN" https://your-ove-server/api/about
  ```

  In PowerShell 7 (`pwsh`), `Invoke-WebRequest` does the same with `-SkipCertificateCheck`; Windows PowerShell 5.1 has no such option.

#### Network Connectivity

- The pre-flight check reports a wrong address or port by itself: see [Pre-flight Check FAILED](#pre-flight-check-failed-wrong-address-or-port-in-base_url).
- Check the connection:

  ```powershell
  Test-NetConnection -ComputerName your-server -Port 443
  ```

- Check that the firewall allows HTTPS (port 443).
- Use a VPN if you work remotely.

### Collection Problems

#### OVE Patch Not Applied

- Ask the OVE administrator to check that the patches are installed.
- The OVE server must run 4.9R3 with the required official patch, attached to the release as `ovshared-4.9R3.jar` ([Files You Need](#files-you-need)).

#### Passwords Missing From the Collected Data (OVE 4.9R3)

- **Symptom:** password fields are missing in the collected data, for example in AP Group and UPAM components, or the SNMP password of Management User Templates.
- **What to do:** install OVE 4.9R3 MR1 or a later patch level before you collect.

#### No Devices in device_data.json (OVE)

- **Symptom:** collect succeeds, but `device_data.json` has no devices. The collect log shows `Source server: OVC4`.
- **Cause:** the host name in `[ov4] base_url` contains `ovcirrus.com`, so the tool took the OVE for an OVC4 and asked it for its devices through the OVC4 API.
- **What to do:** set `base_url` to the IP address of the OVE, or to a host name without `ovcirrus.com`, then collect again ([Step 2](#step-2-collect-data-from-ov4)).

#### Other Collection Failures

- Check that `[ov4] base_url` is the right server, and that the log line `Source server: OVE` or `Source server: OVC4` names the right kind ([Step 2](#step-2-collect-data-from-ov4)). An OVC4 address is under `ovcirrus.com`; an OVE address is not.
- OVE: ask the OVE administrator to check the status of the services and their logs.
- OVC4, device collection: check that the devices are visible in the OVC4 UI under **Network > Device Catalog**, and check their connectivity status.

#### Collection Fails With "The read operation timed out"

- Each OV4 API request waits 3000 ms by default. A request that times out is sent twice more, first with twice and then with four times that timeout, and the log shows `GET ... timed out after 3.0 s; sending it again`. A collector fails only when all three attempts time out.
- If that happens, raise the timeout with `collect_request_timeout_ms` under `[ov4]` (or `--collect-request-timeout-ms`), for example to `10000`, then collect again.

#### Collection Is Slow or Unstable

- Adjust `max_workers` under `[ov4]`: higher is faster, lower is more stable. The default is 5; use a value from 1 to 15.
- Collect during off-peak hours.

#### Certificate Not Collected

- If OV4 serves an empty or unreadable file for a wireless (AP) certificate, only that certificate is skipped. The log names it (`Certificate '<name>' (<type>) is not collected`), and every AP Group and every other certificate is still collected.
- An expired certificate often has no file left on OV4: that is logged as a warning, with its expiry date. To migrate it, upload a valid certificate, then collect again.
- For a certificate that has not expired, the log shows an error: check it on OV4, upload it again if needed, then collect again.

#### Guest, Employee or Company Data Incomplete

- **Symptom:** collect looks successful, but the guest, company or employee snapshot files are incomplete when OV4 holds many records.
- **What to do:** use an OVE build that includes batch retrieval for guest, employee and account data. If the counts look wrong, compare the totals in the OVE UI with the collected JSON files before you migrate.

### Migration Problems

#### Migration Fails With "Request timeout: The read operation timed out"

- Each OV10 API request waits 30000 ms by default. OV10 sometimes takes more than 3 s to answer a request that it otherwise answers in under a second: this is an OV10 issue. A request that times out is sent again, up to three attempts in all (`migration_request_max_retries`), and the object fails only when every attempt times out.
- Each retry logs `OVC10 request timeout (attempt 1/3). Retrying in ...`. An object that OV10 never answers holds the run for about 90 s: the tool has not stopped.
- If an object fails this way, raise the timeout with `migration_request_timeout_ms` under `[migration]` (or `--migration-request-timeout-ms`), for example to `60000`, then migrate the file again. OV10 may have created the object after the tool stopped waiting: migrating it again then counts it as migrated.

#### Guest Accounts With the Same Phone Number or Email

OV4 lets several guest accounts share a phone number or an email. OV10 does not: each guest account must have its own phone number and its own email. Phone numbers are compared after the `+` prefix is added (`13456788888` becomes `+13456788888`), so `13456788888` and `+13456788888` are the same number. An empty phone number or email is never a duplicate.

- **During `collect`:** every guest account is collected, duplicates included.
- **During `migrate`, before any guest account is sent:** the first account in `guest_account_data.json` that uses a phone number or an email is migrated. Every later account that uses the same phone number, or the same email, is not sent to OV10, and is reported as Not Supported by OV10, not as failed.
  - The log has one warning per account, starting with `Guest account '<username>' was not sent to OV10: its phone number '<phone>' is already used by guest account '<other username>'` (or its email `'<email>'`).
  - A total follows, starting with `<count> of <total> Guest Accounts are not migrated because another guest account already uses their phone number or email`.
  - The Migration Summary and the HTML report list these accounts under Not Supported by OV10.

**What to do:** on OV4, change the phone number or the email of the account that was not sent, so that no other guest account uses them. Then collect again, and migrate.

#### Expired Guest Accounts

OV4 keeps a guest account after its expiration date. OV10 does not accept one: it rejects a guest account whose expiration date (`accountValidityPeriod`) is in the past with `The expiration date must not be in the past & must be in milliseconds`. The tool compares the expiration date of each guest account with the current time twice:

- **During `collect`:** an expired account is not collected, and its password is not fetched.
  - The log has one warning per account, starting with `Guest account '<username>' is not collected: it expired on OVE on <date> UTC`.
  - A total follows, starting with `<count> of <total> Guest Accounts are not collected because they have expired on OVE`.
  - If every guest account has expired, `guest_account_data.json` is saved with no accounts, so that an older snapshot is not migrated instead.
- **During `migrate`, just before each account is sent:** an account that has expired since it was collected, or one in a snapshot collected with an earlier version of the tool, is not sent to OV10, and is reported as Not Supported by OV10, not as failed.
  - The log has one warning per account, starting with `Not supported by OV10 GUEST_ACCOUNT: <username> - Guest account '<username>' was not sent to OV10: it expired on OVE on <date> UTC`.
  - The Migration Summary at the end of the log has one line per reason, starting with `Not supported by OV10 - GUEST_ACCOUNT: <count> item(s)`.
  - The HTML report lists the account under Not Supported by OV10, with how many accounts were not sent, why and what to do; click an account to see its expiration date.

The accounts that have not expired are migrated as they are on OV4.

**What to do:** on OV4, set an expiration date in the future on the guest account. Then collect again, and migrate.

#### Guest Phone Numbers Without a Country Code

- **Symptom:** `migrate` fails with a phone validation error on a guest account.
- **Cause:** the tool adds the `+` prefix to a guest phone number that has none (`13456788888` becomes `+13456788888`), but it adds no country code. A number written without one on OV4, such as `0912345678`, is sent as `+0912345678`, which is not an international number.
- **What to do:** on OV4, write the number with its country code (`+<country><number>`), collect again, then migrate.

#### Employee Accounts With a Short Username

- **Symptom:** `migrate` logs `Employee account '<username>' was not sent to OV10: its username has <n> character(s)`, and the HTML report lists the account under Not Supported by OV10.
- **Cause:** OV10 does not accept an employee account username shorter than 3 characters.
- **What to do:** create the account on OV10 by hand, with a longer username.

#### Management User Templates and Value Mappings With a Short Name

- **Symptom:** `migrate` logs `Management User Template '<name>' was not sent to OV10: its name has <n> character(s)` (or `Value Mapping '<name>' ...`), and the HTML report lists it under Not Supported by OV10. The Value Mappings and the switches that use such a Management User Template are listed there too, as requiring it.
- **Cause:** OV10 does not accept a Management User Template or a Value Mapping whose name is shorter than 6 characters. Sent, it would be rejected with `'"data.name" length must be at least 6 characters long'`.
- **What to do:** rename it on OV4 to at least 6 characters, collect again, then migrate. Or create it on OV10 by hand with a longer name; for the switches, pass that name with `--mgmt-template` ([Choose the Switch Management User Template](#choose-the-switch-management-user-template)).

#### Access Role Profiles Bound to VLAN 1 or Above 4090

- **Symptom:** `migrate` logs `Access Role Profile '<name>' was not sent to OV10: its binding to AP Group '<group>' (VLAN <vlans>) maps a VLAN that OV10 does not accept`, and the HTML report lists it under Not Supported by OV10. The SSIDs that use the profile are listed there too, as requiring it.
- **Cause:** OV10 maps an Access Role Profile to the VLANs of an AP Group only as VLAN 0 (untagged) or VLANs 2 to 4090, single or as a range. OV4 also accepts VLAN 1 and VLANs above 4090. Sent, the profile would be rejected with `The VLAN mapping to the AP should be in the range of 0 and 2 to 4090`.
- **What to do:** on OV4, change the VLANs of the AP Group binding that the message names to VLANs OV10 accepts, collect again, then migrate.

#### Expired Local LDAP Certificates

- **Symptom:** `migrate` logs `Local LDAP certificate '<name>' was not sent to OV10: its server certificate expired on <date> UTC`, and the HTML report lists it under Not Supported by OV10.
- **Cause:** OV10 does not accept an expired certificate. Sent, it would be rejected with `This certificate is expired.` The tool reads the expiry date of the server certificate when it migrates, so a certificate that was valid at collect but has expired since is not sent either.
- **What to do:** upload a valid certificate on OV4, collect again, then migrate. Or upload a valid certificate on OV10 by hand.

#### Service Groups With a Service That Has Both Ports (OVE Bug)

- **Symptom:** `migrate` logs `Service group '<name>' was not sent to OV10: its service '<service>' (source port <protocol>/<port>, destination port <protocol>/<port>) has both a source port and a destination port. OVE bug: ...`, and the HTML report lists the group under Not Supported by OV10.
- **Cause:** an OVE bug. A service has either a source port or a destination port, and OVE does not let a service be created with both, yet such services can exist on OVE. OV10 rejects a service group that holds one, with `All services must have only Source Ports or only Destination Ports.` The service itself is migrated on its own; only the groups that hold it are not.
- **What to do:** on OVE, fix the service so that it has only a source port or only a destination port, or replace it in the group with such a service. Then collect again, and migrate.

#### Access Policies That Authenticate Against External LDAP/AD

- **Symptom:** `migrate` logs `Access policy '<name>' was not sent to OV10: it authenticates against External LDAP/AD, but no External LDAP server was collected from OVE`, and the HTML report lists the policy under Not Supported by OV10. The SSID that uses the policy is listed there too, as requiring the access policy.
- **Cause:** OV10 keeps External LDAP and External AD apart, so the tool sends such a policy as External LDAP, which needs the External LDAP server. Collect skips the server when **UPAM > Settings > LDAP/AD Configuration** on OVE has no server IP. An External AD server is not migrated.
- **What to do:** configure the LDAP server on OVE, collect again, then migrate. Or create the server, the access policy and its SSID on OV10 by hand.

#### External LDAP Server Fails With "The maximum number of External LDAP Server is reached"

- **Symptom:** the HTML report counts the `EXTERNAL_LDAP_SERVER` as failed, and `migrate` logs `Error when creating EXTERNAL_LDAP_SERVER object with name <name>. Error: OV10 holds only one External LDAP server per organization and already has one`. The message, also shown when you click the server in the report, ends with what OV10 answered: `{'errorCode': 400, 'errorMsg': 'Bad Request', 'errors': [{'type': 'allowed.items.max.reached', 'field': 'external_ldap_server', 'errorMsg': 'The maximum number of External LDAP Server is reached'}]}`.
- **Cause:** OV10 holds only one External LDAP server per organization, and the organization already has one: either an earlier run of `migrate` created it from `upam_ldap_data.json`, or it was created on OV10 another way. For this object OV10 does not answer that the server already exists, so the tool cannot tell the two cases apart, and counts the server as failed in both.
- **What to do:** on OV10, look at the External LDAP server of the organization.
  - It has the name of the OVE server and an earlier run created it: nothing to do, the server is migrated. If it used a certificate on OVE, restore TLS as [UPAM External LDAP Servers That Use a Certificate](#upam-external-ldap-servers-that-use-a-certificate-ove) says, if you have not done it yet.
  - It is another server: compare it with the OVE server under **UPAM > Settings > LDAP/AD Configuration**. Edit it on OV10 to match, or delete it on OV10 and migrate `upam_ldap_data.json` again.
  - In both cases, the access policies that authenticate against External LDAP/AD are still sent, and authenticate against the server that OV10 holds.

#### Access Role Profiles That Use Web Content Filtering

- **Symptom:** `migrate` logs `Access Role Profile '<name>' was not sent to OV10: it uses the WCF profile '<WCF profile>'`, and the HTML report lists it under Not Supported by OV10. An SSID that uses the profile is listed there too, as requiring the profile.
- **Cause:** OV10 does not support Web Content Filtering yet.
- **What to do:** disable WCF in the Access Role Profile on OV4, collect again, then migrate. The clients on that role then have no web content filtering on OV10.

#### Guest Access Strategies in IP Mode

- **Symptom:** `migrate` logs `Guest access strategy '<name>' was not sent to OV10: it uses fqdnMode IP`, and the HTML report lists it under Not Supported by OV10, with the access policy that uses it and the SSID that uses that policy.
- **Cause:** OV10 supports a guest access strategy in FQDN mode only.
- **What to do:** switch the strategy to FQDN mode on OVE, collect again, then migrate.

#### Settings Waiting for Devices on OV10

- **Symptom:** the HTML report lists AP IP mode, IoT radio, radio or QoE/RTLS settings, mesh settings or schedulers under Waiting for Devices on OV10. Typical messages:
  - `Backup scheduler '<name>' cannot be migrated because device(s) [...] have not been migrated to OVC10 yet ...`
  - `Upgrade scheduler '<name>' cannot be migrated because device '<serial>' has not been migrated to OVC10 yet ...`
- **Cause:** they apply to devices that OV10 does not manage yet. A backup or upgrade scheduler that names a device that is not on OV10 is not sent at all, rather than partly. These objects are not counted as failed.
- **What to do:** migrate the devices (Phase 2), then migrate the same snapshot file again, for example `ap_config_data.json`, `backup_scheduler_data.json` or `scheduler_upgrade_data.json` ([Step 11](#step-11-migrate-the-ap-settings-and-schedulers-that-waited-for-devices)).

#### Data VPN Settings (DATA_VPN_SETTING)

OV4 accepts a Server VPN IP outside the Start IP - End IP range. OV10 does not: it rejects such a server with `The Server VPN IP must be in the IP Range`. During `migrate`, for a server that uses a mask string (`maskStr`), the tool checks the range:

- **Server VPN IP inside the range:** the server is migrated as it is on OV4.
- **Server VPN IP outside the range, in the same subnet:** the range is widened just enough to contain the Server VPN IP: the Start IP moves down to it, or the End IP moves up to it. For example, Server VPN IP `66.66.66.66` with range `66.66.66.67 - 66.66.66.68` is created on OV10 with range `66.66.66.66 - 66.66.66.68`.
  - The log has a warning starting with `Data VPN Server '<name>': the Server VPN IP ... is outside the IP Range`, which gives the old and the new range.
  - The HTML report shows the server as migrated, and also lists it under Needs Attention, with the old and the new range.
  - If the server IP is not next to the range, the wider range also gives the addresses in between to VPN clients. The warning and the report give their number. Check in OV10, under **Network > AP Registration > Data VPN Servers**, that clients may use them.
- **Server VPN IP in another subnet:** the range cannot be widened, so the server is sent as collected, and OV10 rejects it. The log has an error starting with `Data VPN Server '<name>': the Server VPN IP ... is outside the IP Range ... and not in the same subnet`. To fix it:
  1. On OV4, open **Network > AP Registration > Data VPN Servers**.
  2. Change the Server VPN IP or the IP Range, so that the range contains the Server VPN IP.
  3. Collect again, then migrate.

A server that uses a CIDR length (`maskInt`) has no End IP, and is migrated as it is on OV4.

#### Tunnel Profiles That End at a Data VPN Server

On OV4 the far end of a tunnel profile can be a Data VPN Server, chosen by its name: the tunnel then holds that name, for example `DataVPN`, as its server IP. OV10 needs an IP address there.

- During `migrate`, the tool replaces the name with the Server VPN IP of that Data VPN Server, taken from `ap_group_data.json`: the one in the snapshot folder, or the snapshot itself. The backup server IP is handled the same way. Snapshots collected before 1.0.3 do not need to be collected again.
- The tunnel counts as Migrated, and the HTML report also lists it under Needs Attention. The log has a warning starting with `Tunnel profile '<name>': On OVE this tunnel ends at a Data VPN Server`.
- The tool does not link the tunnel to the Data VPN Server on OV10. Check the tunnel on OV10, and link it to that server if its traffic must go through the VPN.
- If `ap_group_data.json` is not in the folder of the snapshot, or holds no Data VPN Server of that name, the name is sent as it is and OV10 rejects the tunnel. The log has a warning starting with `Tunnel profile '<name>': serverIP '<name>' is not an IP address`. Collect `ap_group_data.json` into the same folder, then migrate again.

Only the standalone tunnel profiles are handled: the tunnel of an Access Role Profile's AP Group binding, and of an SSID, is sent as collected.

#### RF Profile External Antenna Gain

- OV10 accepts an external antenna gain (`externalAntennasGain`) from 1.0 to 16.0 only; OV4 accepts other values, such as 0.
- Nothing is needed to migrate the RF profile: the tool sends a value below 1.0, or an empty one, as 1.0, and a value above 16.0 as 16.0. It does not log the change, so check the external antenna gain of the RF profiles on OV10, in particular where OV4 had 0.
- An RF profile that uses the default settings (`defaultSetting` on) takes no external antenna gain on OV10: OV10 rejects one with `{'type': 'valid.required', 'field': 'externalAntennasGain2', 'errorMsg': 'The field must be default value, default = null'}`, and the same on `externalAntennasGain5A`, `5L`, `5H` and `6`. The tool sends no gain for such a profile, so it migrates without one.

### Device Problems

#### Legacy OmniSwitch Without SNMP Credentials

- **Symptom:** `migrate` logs `!! LEGACY SWITCH WARNING — ACTION REQUIRED !!` with `<n> Legacy OmniSwitch (AOS 6x) device(s) have no Management Users Template: <serials>`, and those switches fail. A Management User Template that uses SNMP credentials on OV4 fails to migrate too. A third-party device fails for the same reason, with `Third-party device '<name>' does not include a Management Users Template`.
- **Cause:** AOS 6x switches and third-party devices are not supported in 1.0.3. OV10 10.6.1 manages them over SNMP only, and OV4 4.9R3 does not expose their SNMP credentials, so `collect` builds no `AUTO-SNMP-<hash>` template for them. This is an OV4 limitation, not a migration tool error.
- **What to do:** add the AOS 6x switches to OV10 by hand, with their SNMP credentials, or keep them managed by OV4 until a later version supports them: see [Add AOS 6x Switches to OV10 by Hand](#add-aos-6x-switches-to-ov10-by-hand).

#### AP Fails With "No device group with name"

- **Message:** `No device group with name: <group> in OVC10`.
- **Cause:** the AP Group of the AP is not on OV10. Step 5 creates the AP Groups; device migration does not.
- **What to do:** migrate `ap_group_data.json`, check that the AP Group is on OV10, then migrate the AP again.

#### Device Not Behaving as Expected After Migration

- Run Collect Support Info on the device in OV10: **Devices > select the device > Collect Support Info**.
- Compare it with the Collect Support Info captured from OV4 before the deletion ([8.2](#82-remove-the-device-from-ov4)). The differences point to the configurations that were not applied correctly.
- Check that the AP Group, SSID and RF Profile assignments match between OV4 and OV10.

#### Device Not Managed After Migration

- **Symptom:** the migration reports success, but some devices are not immediately in the managed state.
- **What to do:** finish their onboarding, for example update the call-home URL and trigger the call home of certain switch types, or update the DHCP / Activation Server settings of the APs and reboot them, where it applies ([Step 12](#step-12-point-the-switches-to-ov10)).

#### Configuration Not Saved

- Always save the AP configurations before you delete the APs: **Network > Managed Devices > Actions > Save to Running** ([Save the AP Configurations](#save-the-ap-configurations)).
- Check the "Configuration Saved" status before the deletion.

#### Device Deletion Issues

- Make sure the devices are not assigned to critical services.
- Delete them in the UI: **Network > Device Catalog > Delete**.
- Check that the deletion completed before you migrate.

### Getting More Help

The tool describes its commands and options:

```powershell
.\migration_tool.exe --help
.\migration_tool.exe collect --help
.\migration_tool.exe migrate --help
```

For a problem not covered above:

1. Enable DEBUG logging: `log_level = "DEBUG"` ([Read the Log](#read-the-log)).
2. Run the command again, and read `migration_tool.log`.
3. Contact support with the log file and the error messages.

## Reference

### Command-Line Options

A value given on the command line overrides the same setting in the config file. Each option below has its config file key in [Configuration File](#configuration-file).

For `collect` and `migrate`:

| Option | What it does |
|---|---|
| `--config-file <file>` | The config file, such as `config.local.toml`. |
| `--secret-key-pass <passphrase>` | Instead of `secret_key_pass` in the config file. |
| `--log-level <level>` | `DEBUG`, `INFO`, `WARNING`, `ERROR` or `CRITICAL`. Default: `INFO`. |
| `--help` | Describes the command and its options. |
| `--version` (or `-v`), before the command | Shows the version of the tool. |

For `collect`:

| Option | What it does |
|---|---|
| `--data-dir <folder>` | The folder the snapshot files are written to. Default: the current folder. |
| `--ov4-base-url <url>`, `--ov4-token <token>` | Instead of `base_url` and `token` under `[ov4]`. |
| `--ov4-trusted-cert <fingerprint>` | A SHA-256 fingerprint to trust for OV4. Not needed: the tool trusts and saves a self-signed certificate itself ([Certificates](#certificates)). |
| `--max-workers <n>` | Parallel collection workers. Default: 5. |
| `--account-processing-max-workers <n>` | Parallel workers that fetch the passwords of guest and employee accounts. Default: 16. |
| `--collect-request-timeout-ms <ms>` | How long each OV4 request waits. Default: 3000. |

For `migrate`:

| Option | What it does |
|---|---|
| `--data-dir <folder>` | Migrates every snapshot file of the folder, in the order of [Snapshot Files](#snapshot-files). |
| `--snapshot-file <file>` | Migrates one snapshot file. Given with `--data-dir`, `--snapshot-file` is used and `--data-dir` ignored, with a warning. |
| `--skip-devices` | With `--data-dir`: leaves out the devices. Ignored with `--snapshot-file`. |
| `--serial <serial numbers>` (or `--device-serial`) | Migrates only the devices with these serial numbers, separated by commas or spaces. |
| `--mgmt-template "<name>"` | The OV10 Management User Template of every switch of the run. It must exist on OV10 ([Choose the Switch Management User Template](#choose-the-switch-management-user-template)). |
| `--map-ap-groups-to-sites <file>` | The AP Group to Site mapping file ([Step 4](#step-4-optional-place-ap-groups-in-several-ov10-sites)). |
| `--ov10-base-url`, `--org-id`, `--site-id`, `--app-id`, `--app-secret` | Instead of the same keys under `[ov10]`. |
| `--ov10-trusted-cert <fingerprint>` | A SHA-256 fingerprint to trust for OV10. Not needed, as for OV4. |
| `--migration-request-timeout-ms <ms>` | How long each OV10 request waits. Default: 30000. |
| `--migration-request-max-retries <n>` | Attempts per OV10 request, the first one included. Default: 3. |
| `--migration-request-retry-backoff-ms <ms>` | The base wait before a retry, doubled at each attempt. Default: 500. |
| `--migration-request-interval-ms <ms>` | The minimum wait between two OV10 requests, for example 100 to reduce the load on OV10. Default: 0. |
| `--migration-parallel-standalone` | Migrates in parallel the object types that depend on no other. |
| `--migration-max-workers <n>` | Parallel workers for guest and employee accounts, and for the independent object types with `--migration-parallel-standalone`. Lower it to reduce the load on OV10. Default: 8. |

The names used before 1.0.3 still work: `--ovc4-base-url`, `--ovc4-token`, `--ovc4-trusted-cert`, `--ovc10-base-url` and `--ovc10-trusted-cert`. `--device-source` is no longer used: the tool ignores it and logs a warning.

### Configuration File

`config.local.toml` is your copy of `config.toml` ([Step 1](#step-1-fill-in-configlocaltoml)). Only `secret_key_pass`, `[ov4]` and `[ov10]` are required; every other key is optional, and documented in `config.toml` itself.

| Section | Key | Default | What it does |
|---|---|---|---|
| (top of the file) | `secret_key_pass` | Required | Your passphrase, which encrypts the passwords in the snapshot files ([secret_key_pass](#secret_key_pass)). |
| (top of the file) | `data_dir` | The current folder | The snapshot folder: `collect` writes to it. `migrate` migrates it when the command gives neither `--data-dir` nor `--snapshot-file`. |
| (top of the file) | `snapshot_file` | None | The one snapshot file that `migrate` sends when the command gives neither `--data-dir` nor `--snapshot-file`. It wins over `data_dir`. |
| (top of the file) | `log_level` | `INFO` | `DEBUG`, `INFO`, `WARNING`, `ERROR` or `CRITICAL`. |
| `[trusted_certs]` | `ov4`, `ov10` | Written by the tool | The SHA-256 fingerprints of self-signed certificates ([Certificates](#certificates)). |
| `[ov4]` | `base_url`, `token` | Required | The address of OV4 and its API token. |
| `[ov4]` | `max_workers` | 5 | Parallel collection workers. Use a value from 1 to 15. |
| `[ov4]` | `account_processing_max_workers` | 16 | Parallel workers that fetch the passwords of guest and employee accounts. |
| `[ov4]` | `collect_request_timeout_ms` | 3000 | How long each OV4 request waits. A request that times out is sent twice more, with 2 and then 4 times this timeout. |
| `[ov10]` | `base_url`, `org_id`, `site_id`, `app_id`, `app_secret` | Required | The address of OV10, the organization, the default Site and the application. |
| `[ov10]` | `skip_devices` | `false` | `true` leaves out the devices with `--data-dir`. |
| `[ov10]` | `mgmt_template` | None | As `--mgmt-template`. |
| `[ov10]` | `mapAPGroupsToSites` | `ov4APGroupToOV10SiteMapping.csv`, if it is in the current folder | The AP Group to Site mapping file. |
| `[migration]` | `migration_request_timeout_ms` | 30000 | How long each OV10 request waits. The Guest Access global configuration and the AAA profiles always wait at least 30000. |
| `[migration]` | `migration_request_max_retries` | 3 | Attempts per OV10 request, the first one included. |
| `[migration]` | `migration_request_retry_backoff_ms` | 500 | The base wait before a retry, doubled at each attempt. |
| `[migration]` | `migration_request_interval_ms` | 0 | The minimum wait between two OV10 requests. |
| `[migration]` | `migration_parallel_standalone` | `false` | `true` migrates in parallel the object types that depend on no other. |
| `[migration]` | `migration_max_workers` | 8 | Parallel workers for guest and employee accounts, and for the independent object types with `migration_parallel_standalone`. |

#### Files Written for an Earlier Version

A `config.local.toml` from an earlier version works as it is:

- The tool still reads the names it uses, `[ovc4]`, `[ovc10]`, and the `ovc4` and `ovc10` keys under `[trusted_certs]`, as `[ov4]`, `[ov10]`, `ov4` and `ov10`. When a file has both names for a section, a key set in both is read from the new name, and the tool logs a warning.
- It ignores `device_source`, and `email` and `password` under `[ovc10]`, with a warning. Remove them, in particular the OV10 password, which the file no longer needs to hold.
- `device_source` and `--device-source` chose the OV4 device API before 1.0.3. The tool now tells OVE from OVC4 by the host name of `[ov4] base_url` ([Step 2](#step-2-collect-data-from-ov4)).

### Snapshot Files

`collect` writes one snapshot file per group of objects into the `--data-dir` folder. `migrate --data-dir` sends them in the order below; `migrate --snapshot-file` sends one of them. The passwords and secrets in them are encrypted with your `secret_key_pass`.

| File | What it holds | Migrated in |
|---|---|---|
| `ap_group_data.json` | AP Groups, provisioning templates, RF profiles, IoT/location servers, Data VPN servers, wireless certificates | Step 5 |
| `switch_config_template_data.json` | Switch configuration templates, value mappings, Management User Templates | Step 5 |
| `aaa_server_data.json` | RADIUS and LDAP servers | Step 5 |
| `aaa_profile_data.json` | AAA profiles | Step 5 |
| `access_policy_data.json` | Location and period policies | Step 5 |
| `access_role_profile_data.json` | Access Role Profiles, tunnel profiles, global configuration settings | Step 5 |
| `policy_data.json` | Unified policies and policy lists, MAC and IP groups, services, service groups and service ports | Step 5 |
| `access_auth_profile_data.json` | Access Auth Profiles | Step 5 |
| `iot_category_data.json` | IoT categories | Step 5 |
| `employee_account_data.json` | Employee accounts | Step 5 |
| `guest_account_data.json` | Guest accounts | Step 5 |
| `company_property_data.json` | Company properties | Step 5 |
| `scheduler_upgrade_data.json` | Upgrade schedulers | Step 5, and again in Step 11 |
| `basic_setting_data.json` | The organization inactivity timeout | Step 5 |
| `event_responder_data.json` | Event responders | Step 5 |
| `dhcp_option_82_data.json` | The DHCP Option 82 setting | Step 5 |
| `backup_scheduler_data.json` | Backup schedulers | Step 5, and again in Step 11 |
| `upam_ldap_data.json` | The UPAM External LDAP server (OVE only) | Step 5 |
| `upam_data.json` | UPAM access policies, captive portal templates, guest and BYOD access strategies, External RADIUS servers | Step 5 |
| `service_level_data.json` | Guest service levels | Step 5 |
| `guest_access_global_config_data.json` | The Guest Access global configuration | Step 5 |
| `wips_policy_data.json` | The WIPS policy | Step 5 |
| `ssid_data.json` | SSIDs and WLAN services | Step 5 |
| `device_data.json` | The devices, and the `AUTO-SNMP-<hash>` Management User Templates built from the switch SNMP credentials | Steps 8 and 9 |
| `mesh_data.json` | The mesh settings of the APs | Step 10 |
| `ap_config_data.json` | The IP mode, IoT radio, radio and QoE/RTLS settings of each AP | Step 5, and again in Step 11 |

### What Gets Migrated

| Area | Objects | Notes |
|---|---|---|
| Network infrastructure | AP Groups, Provisioning Templates, RF Profiles | Only the RF profiles used by at least one AP Group are collected and migrated. A profile that no AP Group uses, for example because the AP Group uses the "default" RF profile instead, is not: assign it to an AP Group on OV4 before you collect, or recreate it on OV10. |
| Switch configuration | Switch Templates, Value Mappings, Management User Templates | The Management User Templates are collected and migrated. Each switch is migrated with the template of its snapshot entry, or with the one that `--mgmt-template` (or `mgmt_template`) sets for the whole run ([Choose the Switch Management User Template](#choose-the-switch-management-user-template)). |
| Wireless networks | SSIDs and WLAN Services | |
| AP mesh | The mesh configuration of the APs | In `mesh_data.json`, migrated once the AP devices are on OV10 ([Step 10](#step-10-migrate-the-ap-mesh-configuration)). |
| AP configuration | `AP_IP_MODE`, `AP_IOT_RADIO`, `AP_RADIO` and `AP_QOE_RTLS_STATUS`: the IP mode and static IP settings, IoT radio settings, radio settings and QoE/RTLS status of each AP | In `ap_config_data.json`, migrated once the AP devices are on OV10. Migrated before, they are listed under Waiting for Devices on OV10: migrate the file again once the APs are on OV10 ([Step 11](#step-11-migrate-the-ap-settings-and-schedulers-that-waited-for-devices)). |
| Authentication | AAA Servers (RADIUS, LDAP), AAA Profiles | |
| UPAM | Access policies, guest and BYOD access strategies, External RADIUS servers, captive portal templates, guest service levels, the Guest Access global configuration | Captive portal templates: the layout only ([Other Configurations Requiring Manual Action](#other-configurations-requiring-manual-action)). |
| UPAM External LDAP server (OVE only) | The server under UPAM > Settings > LDAP/AD Configuration | Migrated as a non-secure (NS) server only. LDAP servers that use a certificate are not supported: their LDAPS certificates and the UPAM Additional Trust CAs are not migrated ([UPAM External LDAP Servers That Use a Certificate](#upam-external-ldap-servers-that-use-a-certificate-ove)). OV10 holds only one External LDAP server per organization: the server fails when OV10 already has one ([details](#external-ldap-server-fails-with-the-maximum-number-of-external-ldap-server-is-reached)). |
| Access control | Access Role Profiles, Access Auth Profiles, tunnel profiles | |
| Policies | Location Policies, Period Policies, Unified Policies and Policy Lists, MAC and IP Groups, services, service groups and service ports | |
| External services | IoT/Location Servers (External Engines) | |
| Security | VPN Settings, Certificates (various types) | |
| User management | Employee Accounts, Guest Accounts, Company Properties | Except the accounts that OV10 rejects ([Not Supported by OV10](#not-supported-by-ov10)). |
| Organization settings | IoT categories, the inactivity timeout, DHCP Option 82, the WIPS policy | |
| Scheduling | Upgrade Schedulers, Backup Schedulers, Event Responders | |
| Devices | AOS 8 switches and Access Points | Not AOS 6x switches or third-party devices in 1.0.3: see below. |

The devices:

- **AOS switches** (`deviceFamily: AOS`): standard AOS 8 switches.
- **Not supported in 1.0.3:** Legacy OmniSwitches (`deviceFamily: LEGACY_OMNISWITCH`), the OS6350 and OS6450 switches running AOS 6.7.2.222.R08 or later, and third-party devices (OVE). OV10 manages them over SNMP, and OV4 4.9R3 does not expose their SNMP credentials. Collect still writes them into `device_data.json`, but they fail at `migrate`: see [Legacy OmniSwitch](#legacy-omniswitch-os6350--os6450).
- **Skipped automatically**, because OV10 does not support them:
  - OS2260, OS2360 (AOS 5x)
  - OS2220 (Websmart AOS 8.3.1)
  - OS6250 (AOS 6.7.1), OS6850/E, OS6855 (AOS 6.4.6), OS9700/E, OS9800/E (AOS 6.6.4)

To migrate selected devices only, use `--serial`, or its other name `--device-serial`.

### What's Not Migrated

#### AOS 6x Switches and Third-Party Devices (1.0.3)

Not supported in 1.0.3, and neither are the Management User Templates that use SNMP credentials on OV4: OV10 needs the SNMP credentials to manage these devices, and OV4 4.9R3 does not expose them. You can add the AOS 6x switches to OV10 by hand, with their SNMP credentials: see [Add AOS 6x Switches to OV10 by Hand](#add-aos-6x-switches-to-ov10-by-hand).

#### Not Supported by OV10

These objects are collected, but OV10 does not support them as they are on OV4, so the tool does not send them. The migration report lists each one under Not Supported by OV10, with the reason and what to do, and does not count it as failed: this is an OV10 limitation, or an OV4 bug where the reason says so, not a migration tool error.

| Object | When OV10 does not support it | What to do |
|---|---|---|
| Access Role Profile | It uses a Web Content Filtering (WCF) profile. OV10 does not support WCF yet. | Disable WCF in the profile on OV4, collect again, then migrate. Clients on that role then have no web content filtering on OV10. [Details](#access-role-profiles-that-use-web-content-filtering) |
| Guest Access Strategy | It uses the IP mode (`fqdnMode IP`). OV10 supports the FQDN mode only. | Switch it to FQDN mode on OVE, collect again, then migrate. [Details](#guest-access-strategies-in-ip-mode) |
| Guest account | Its expiration date is in the past. | Set an expiration date in the future on OV4, collect again, then migrate. [Details](#expired-guest-accounts) |
| Guest account | Another guest account already uses its phone number or email. | Change its phone number or email on OV4, collect again, then migrate. [Details](#guest-accounts-with-the-same-phone-number-or-email) |
| Employee account | Its username is shorter than 3 characters. | Create the account on OV10 by hand, with a longer username. [Details](#employee-accounts-with-a-short-username) |
| Management User Template, Value Mapping | Its name is shorter than 6 characters. | Rename it on OV4 to at least 6 characters, collect again, then migrate; or create it on OV10 by hand with a longer name. [Details](#management-user-templates-and-value-mappings-with-a-short-name) |
| Access Role Profile | It binds an AP Group to VLAN 1, or to a VLAN above 4090. OV10 accepts 0 (untagged) or 2 to 4090 only. | Change the VLANs of that binding on OV4, collect again, then migrate. [Details](#access-role-profiles-bound-to-vlan-1-or-above-4090) |
| Local LDAP certificate | Its server certificate has expired. | Upload a valid certificate on OV4, collect again, then migrate. [Details](#expired-local-ldap-certificates) |
| Service group | OVE bug: it holds a service that has both a source port and a destination port. OVE does not let such a service be created, and OV10 rejects a group that holds one. | Fix the service on OVE so that it has only one of the two ports, collect again, then migrate. [Details](#service-groups-with-a-service-that-has-both-ports-ove-bug) |
| Access policy | It authenticates against External LDAP/AD, but no External LDAP server was collected from OVE. | Configure the LDAP server under UPAM > Settings > LDAP/AD Configuration on OVE, collect again, then migrate; or create the server and the policy on OV10 by hand. [Details](#access-policies-that-authenticate-against-external-ldapad) |
| An object that requires one of the above | For example, an access policy that uses such a guest access strategy, an SSID that uses such an Access Role Profile or access policy, or a Value Mapping or a switch that uses such a Management User Template. The report names the object it requires. | Migrate the object it requires first, as that object's own row says, then migrate this one again ([Knock-On](#knock-on)). |

The settings of a device that OV10 does not manage yet are not in this list: the report lists them under Waiting for Devices on OV10, and they migrate once the devices do ([Step 11](#step-11-migrate-the-ap-settings-and-schedulers-that-waited-for-devices)).

#### UPAM External LDAP Servers That Use a Certificate (OVE)

> [!IMPORTANT]
> **Not supported:** the migration tool does not support LDAP servers that use a certificate (TLS / LDAPS). Only a non-secure (NS) LDAP server is migrated.

What the tool does with the server under **UPAM > Settings > LDAP/AD Configuration**:

- It is always created in OV10 as a non-secure (NS) server.
  - A server without a certificate is migrated with its LDAP settings as they are on OVE: server and backup IP, port, admin DN and password, search base, object class, user name and password attributes. OVE has no password type, so it is set to Cleartext, the OV10 default.
  - A server with a certificate is converted to NS: TLS off and no LDAPS certificate. An LDAP port of 636 (LDAPS) becomes 389, because plain LDAP on 636 cannot connect; any other port is kept. Every other setting is migrated as for a server without a certificate.
  - A server that is configured but switched off on OVE is still migrated: an OV10 External LDAP server has no on/off switch. A record with no server IP means LDAP was never set up on OVE, so nothing is migrated.
  - The OVE settings with no OV10 equivalent are not migrated: retries, timeout, and the Windows AD settings (OV10 keeps an AD server as a separate External AD server).
  - OV10 holds only one External LDAP server per organization. When it already has one, created by an earlier run or another way, OV10 rejects the server and the report counts it as failed: see [External LDAP Server Fails With "The maximum number of External LDAP Server is reached"](#external-ldap-server-fails-with-the-maximum-number-of-external-ldap-server-is-reached).
- Its bind password is encrypted in the snapshot with your `secret_key_pass`, like every other collected password.
- Its LDAPS certificates and the UPAM Additional Trust CAs are not collected or migrated.
- If the server uses a certificate on OVE, both `collect` and `migrate` log a warning naming it, starting with `External LDAP server '<name>' uses TLS on OVE`. The HTML report shows it as migrated, and also lists it under Needs Attention, with what was changed.

A server that used a certificate on OVE is therefore still created in OV10, but without TLS, so authentication against it can fail until you restore TLS by hand.

**Restore TLS on the UPAM External LDAP server** (only if it used a certificate on OVE):

1. Before the migration, record the TLS setting and the port of the server on OVE, and keep a copy of its LDAPS certificate files and of the Trust CA it relies on: the tool does not download them.
2. In OV10, upload the LDAPS certificate and the Trust CA.
3. Edit the migrated External LDAP server: turn TLS back on, select the certificate, and set the port back to the one you recorded, for example 636 (the tool changed it to 389).
4. Test authentication through a policy that uses the LDAP server, before you migrate devices.

#### Objects With No Equivalent Setting in OV10

These objects exist on OV4 but have no matching configuration in OV10, so they are not migrated. Recreate them by hand in OV10 where an equivalent function exists; otherwise, they do not apply in the new system.

| Where on OV4 | What is not migrated | Note |
|---|---|---|
| UPAM | Switch User Account | Employee and guest accounts are migrated. |
| Settings page (OVE only) | Email Server | |
| WLAN > Floor Plan | Floor Plan | SSIDs are migrated. |
| Unified Access > Web Content Filtering | WCF profiles | OV10 does not support Web Content Filtering yet. For the Access Role Profiles that use one, see [Not Supported by OV10](#not-supported-by-ov10). |
| Security > Authentication Servers (OVE only) | ACE Server | RADIUS and LDAP servers are migrated. |
| Security > Users and Groups | Roles, Users, Groups | |
| Unified Access | AAA Global Setting | AAA profiles and Access Role Profiles are migrated. |
| Configuration > Report | Report | |
| Configuration > CLI Scripting | CLI Scripting | |
| Configuration > Resource Manager > Upgrade Image (OVE only) | Upgrade Image | Upgrade schedulers are migrated. |
| Network > Discovery Profile (OVE only) | Discovery Profile | |
| Network > Topology | Topology Map | |
| Network > Analytics | Analytic Profiles, Chart View Profiles | |
| Network > IOT | Custom VRF | IoT categories are migrated. |
| Network > Trap Configuration | Trap Configuration | |

#### Objects Supported in OV10 But Not Migrated by This Tool

These objects have an equivalent in OV10, but the tool does not migrate them yet: recreate them by hand.

| Where on OV4 | What is not migrated | Note |
|---|---|---|
| UPAM | Guest Operator | Guest accounts are migrated. |
| UPAM > Settings > LDAP/AD Configuration (OVE only) | The TLS/LDAPS setting of the External LDAP server, its LDAPS certificates and the Additional Trust CAs | The server itself is migrated as non-secure (NS): see [UPAM External LDAP Servers That Use a Certificate](#upam-external-ldap-servers-that-use-a-certificate-ove). |
| AP Registration > Access Points | The remaining per-AP override settings, such as AP Configuration Update, RF Profile Override and similar AP-level overrides | The IP mode is migrated through `ap_config_data.json`, and the mesh through `mesh_data.json`. |

**For AP migrations, this is critical:** the tool does not apply the remaining per-AP overrides of OV4 again, so you may need to recreate them by hand once the APs are onboarded.

#### Switch Settings That Point to OV4

The SNMP trap receiver, the sFlow collector, Application Visibility and the policy server on the switches point to OV4, and so does the UPAM/authentication-server IP of the UNP/UPAM profiles. The tool does not change them. Remove the first four before you delete the switches from OV4 ([Remove the OV4 Settings From the Switches](#remove-the-ov4-settings-from-the-switches)); after the migration, configure them for OV10 and point the UNP/UPAM profiles to the OV10 UPAM server ([Step 12](#step-12-point-the-switches-to-ov10)).

#### Other Configurations Requiring Manual Action

- **Historical data:** the logs, alarms, event history and performance statistics of OV4 are not transferred to OV10.
- **Network topology and maps:** recreate in OV10 the custom network maps and floor plans configured on OV4.
- **Captive portal pages:** the tool creates each captive portal template on OV10 with its name, description and layout only. The content and branding of the pages, such as texts, images and logos, are not migrated: recreate them on OV10.
- **Third-party device MIBs:** the third-party device MIBs imported by the user on OVE are not migrated.

### Security Best Practices

1. **Protect the config file.** `config.local.toml` holds the OV4 token and the OV10 `app_secret` (no OV10 password). Store it in a secure location, with restricted access. In PowerShell, you can leave your own Windows account as the only one that can read and change it:

   ```powershell
   icacls config.local.toml `
     /inheritance:r `
     /grant:r "${env:USERDOMAIN}\${env:USERNAME}:(M)"
   ```

   Type the braces and the domain as shown: without the braces, PowerShell drops the user name, and without the domain, a computer or another account with the same name can receive the access instead of you. Keep `(M)`: the tool writes the certificate fingerprints into this file. Read-only, with `(R)`, it cannot: it then trusts a self-signed certificate on each run without saving it, and logs a warning.
2. **Handle the snapshot files securely.** They contain sensitive data: delete them after the migration, or store them securely.
3. **Check the certificate fingerprints.** The tool trusts a self-signed certificate by itself and logs its fingerprint: compare it once with the certificate of your server, and look into any `has changed since its fingerprint was saved` warning.
4. **HTTPS only.** All API communications use encrypted connections.

## Release Notes

### Version 1.0.3

**Known limitation**

- AOS 6x switches (OS6350, OS6450) and third-party devices are not supported. OV10 10.6.1 manages them over SNMP, and OV4 4.9R3 does not expose their SNMP credentials, so the tool cannot migrate them, nor the Management User Templates that use SNMP credentials on OV4. Support is planned for a later version. Until then, you can add the AOS 6x switches to OV10 by hand, with their SNMP credentials ([Add AOS 6x Switches to OV10 by Hand](#add-aos-6x-switches-to-ov10-by-hand)).

**Validation and guide**

- The [Officially Validated Migration Matrix](#officially-validated-migration-matrix) covers OVE and OVC4 to OVC10 and OV Terra, with OV4 4.9R3 and its official patch, and OV10 10.6.1.
- The guide is reorganized by phase, with a [Quick Start](#quick-start), the [Prerequisites](#prerequisites) on OV4 and OV10, the [Troubleshooting](#troubleshooting) grouped by stage, and a [Reference](#reference) of the options and config keys. It explains `secret_key_pass`, where to create the OV10 application and where the switch Management User Templates come from.

**Configuration**

- The config file names the source server `[ov4]` and the target server `[ov10]`; a file that still uses `[ovc4]` and `[ovc10]` works as before.
- `migrate` authenticates with the OV10 application (`app_id` and `app_secret`) only, so `config.local.toml` no longer holds an OV10 email or password.
- The tool tells OVC4 from OVE by the host name of the source server (`ovcirrus.com` for OVC4), so `device_source` is no longer used.
- Trusts a self-signed certificate, such as the one of an OVE, and saves its fingerprint under `[trusted_certs]` in `config.local.toml` itself, instead of stopping for you to copy it.
- Reports a wrong address or port in `base_url` as such, for example a port that refuses the connection or does not answer HTTPS, instead of as a certificate problem.
- Stops `collect` or `migrate` when `base_url`, `token`, `app_id` or `app_secret` still holds the example value from `config.toml`, and warns when `secret_key_pass` does ([What the Tool Checks Before It Starts](#what-the-tool-checks-before-it-starts)).
- Each OV10 request waits up to 30 s by default, instead of 3 s, because OV10 sometimes takes more than 3 s to answer. An AAA profile request waits at least 30 s, even with a shorter `migration_request_timeout_ms`.

**What is sent to OV10**

- Migrates the OVE UPAM External LDAP server as a non-secure (NS) server. LDAP servers that use a certificate are not supported (see [UPAM External LDAP Servers That Use a Certificate](#upam-external-ldap-servers-that-use-a-certificate-ove)). Snapshots collected with 1.0.1 hold this server under its OVE field names, and must be collected again.
- Does not migrate an expired guest account, or a guest account whose phone number or email another guest account already uses, because OV10 rejects both. The migration report lists them under Not Supported by OV10 ([Not Supported by OV10](#not-supported-by-ov10)).
- Widens the IP Range of a Data VPN Server to contain its Server VPN IP when both are in the same subnet, as OV10 requires ([Data VPN Settings](#data-vpn-settings-data_vpn_setting)).
- Does not send an Access Role Profile that uses a WCF profile, because OV10 does not support Web Content Filtering yet. Before 1.0.3, the profile failed with `Not Found Error. Hint: Looks like the object depends on another object that has not been created yet.`
- Does not send a Guest Access Strategy in IP mode (`fqdnMode IP`), which OV10 does not support. Before 1.0.3, OV10 answered `{'errorCode': 500, ... 'errorMsg': 'samp.server.paramsError' ...}`, and the access policy and the SSID failed after it.
- Sends an OVE access policy that authenticates against External LDAP/AD as External LDAP, and does not send it when no External LDAP server was collected. Before 1.0.3, the SSID that uses it failed with `No access policy with name: <name> in OVC10`.
- Creates the External RADIUS server before the access policy that authenticates against it.
- Sends a tunnel profile whose server is a Data VPN Server on OV4, named instead of an IP address, with the Server VPN IP of that server, and lists it under Needs Attention ([Tunnel Profiles That End at a Data VPN Server](#tunnel-profiles-that-end-at-a-data-vpn-server)). Before 1.0.3 OV10 rejected it, as the name is not an IP address.
- Does not send an employee account whose username is shorter than 3 characters.
- Does not send a Management User Template or a Value Mapping whose name is shorter than 6 characters, an Access Role Profile that binds an AP Group to VLAN 1 or to a VLAN above 4090, or a Local LDAP certificate whose server certificate has expired, which OV10 rejects. Before 1.0.3 they failed with `'"data.name" length must be at least 6 characters long'`, `The VLAN mapping to the AP should be in the range of 0 and 2 to 4090` and `This certificate is expired.`; they are now listed under Not Supported by OV10 ([Not Supported by OV10](#not-supported-by-ov10)).
- Does not send a service group that holds a service with both a source port and a destination port, which an OVE bug leaves on OVE although OVE does not let such a service be created. Before 1.0.3 the group failed with `All services must have only Source Ports or only Destination Ports.`; it is now listed under Not Supported by OV10, as an OVE bug ([Service Groups With a Service That Has Both Ports](#service-groups-with-a-service-that-has-both-ports-ove-bug)).

**Collection and release**

- During collection, a request that times out is sent again with a longer timeout, and the timeout can be set with `collect_request_timeout_ms`. A wireless certificate that OV4 cannot serve is skipped, instead of failing the whole collection.
- The release lists the SHA-256 checksum of each asset in `sha256sum.txt`.

**Report and log**

- The report and the log list the objects that OV10 does not support, and the objects that require them, under Not Supported by OV10, and the settings of the devices that OV10 does not manage yet under Waiting for Devices on OV10; neither is counted as failed ([Step 6](#step-6-read-the-migration-report)). Before 1.0.3, the settings that waited for devices failed with `... cannot be migrated because the device has not been migrated to OVC10 yet` or `No Devices found with deviceMacAddresses: ...`.
- Names an object that is not migrated because an object it requires was not, such as an SSID whose access policy failed or is not supported by OV10, a knock-on: its message names the object it requires and the first object of the chain, and the report shows a failed knock-on in amber ([Knock-On](#knock-on)).
- Says why OV10 rejects the UPAM External LDAP server when the organization already has one: OV10 holds only one External LDAP server per organization, also when an earlier run created it. Before 1.0.3, the report gave only what OV10 answered, `{'errorCode': 400, ... 'type': 'allowed.items.max.reached', 'field': 'external_ldap_server', 'errorMsg': 'The maximum number of External LDAP Server is reached'}` ([details](#external-ldap-server-fails-with-the-maximum-number-of-external-ldap-server-is-reached)).
- Does not log passwords, secrets or access tokens, even at DEBUG level. At DEBUG level, logs the whole body of each request and answer, instead of its first 500 characters.

**Fixes**

- Finds an Access Role Profile whose name has a dot, such as `__test-802.1x`, which OV10 does not find by its name, so the SSID that uses it no longer fails with `HTTP Error 404: Not Found`.
- **SSID rejected for its client session logging or MLO band.** An SSID failed with `Disable client session logging status only accepts logging level: NONE`, or with `ssid.mloBand must be one of [null, All, Band_24, ...]` or `MLO bands must match the allowed Allow bands` although MLO is disabled on OV4. 1.0.3 sends the client session logging status, and the allowed bands of the SSID as its MLO band when MLO is off or no MLO band is set. Migrate the SSIDs again with 1.0.3; the snapshot does not need to be collected again.
- **AP Group rejected for its SSH password while SSH is disabled.** A provisioning template failed with `{'type': 'null.required', 'field': 'sshSupportPassword', 'errorMsg': 'This field is not required.'}` (or the same on `sshRootPassword`). Its AP Group failed as a knock-on, then the Access Role Profiles assigned to that AP Group with `No device group with name: <AP Group>.`, and the SSIDs that use those profiles. OVE keeps the SSH support and root passwords of an AP Group after SSH is disabled, and OV10 rejects them while SSH is off, so 1.0.3 no longer sends them when SSH is disabled. Migrate `ap_group_data.json` again with 1.0.3, then the file that holds the Access Role Profiles and SSIDs; the snapshot does not need to be collected again.
- **RF profile rejected for its external antenna gain while it uses the default settings.** An RF profile with `defaultSetting` on failed with `{'type': 'valid.required', 'field': 'externalAntennasGain2', 'errorMsg': 'The field must be default value, default = null'}` (and the same on `externalAntennasGain5A`, `5L`, `5H` and `6`), and the provisioning templates that use it and their AP Groups failed as knock-ons. OV10 takes no external antenna gain on such a profile, so 1.0.3 sends none ([RF Profile External Antenna Gain](#rf-profile-external-antenna-gain)). Migrate `ap_group_data.json` again with 1.0.3; the snapshot does not need to be collected again.

### Version 1.0.1

- Supports migration to OV Terra.
- Places OV4 AP Groups in different OV10 Sites automatically ([Step 4](#step-4-optional-place-ap-groups-in-several-ov10-sites)).

### Version 0.2.0

Initial version.
