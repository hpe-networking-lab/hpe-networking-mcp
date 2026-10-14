---
name: aos8-controller-recovery
title: Recover an Aruba 70xx/72xx/90xx controller — unknown admin password or still bound to a Mobility Conductor
description: |
  Console/bootloader runbook for an Aruba 70xx/72xx/90xx controller you cannot
  log into, or a second-hand unit still bound to someone else's Mobility
  Conductor. Use when the admin password is unknown, when configure terminal is
  refused with "This controller is managed by a Mobility Conductor" (or
  "Mobility Master"), when a controller reports an interface up/up but is
  unreachable, or when a bootloader factory reset left it unable to finish
  starting.

  Match phrases include: "forgot controller password", "password recovery",
  "forgetme", "managed by a Mobility Conductor", "managed by a Mobility Master",
  "used / second-hand Aruba controller", "write erase all permission denied",
  "format 0:2", "cpboot", "apboot", "Ancillary files are not present",
  "additional image upgrade is required", "controller up but not pingable",
  "VLAN 4094", "AOS 8 enable mode", "bootdelay", "device already exists in
  workspace".
platforms: [aos8]
tags: [aos8, controller, recovery, password, console, bootloader, factory-reset, mobility-conductor, troubleshooting]
tools:
  - health
  - aos8_get_version
  - aos8_get_licenses
  - aos8_get_controllers
  - aos8_show_command
---

# Recover an Aruba controller from unknown or second-hand state

## Objective

Regain administrative control of an Aruba 70xx/72xx/90xx controller whose admin
password is unknown or which is still bound to a previous owner's Mobility
Conductor, return it to a known standalone state, and make it reachable on the
network. Every step was executed end to end on a 7005, including the mistakes;
following the order below avoids most of the time those cost.

## Prerequisites

- **Serial-console access is required — almost everything here happens on the
  console or in the bootloader**, where no MCP tool can reach. Console settings:
  9600 8N1. The recovery account works on the serial console only, never over
  SSH, by design.
- **MCP tools apply only once the controller is reachable again** and configured
  as this server's AOS8 target. Use them for post-recovery verification
  (Step 8): `health(platform="aos8")`, `aos8_get_version`, `aos8_get_licenses`,
  `aos8_show_command`. Before that point, the procedure is CLI-only.
- **Know which OS generation it runs before choosing a procedure.** AOS 6, AOS 8
  and AOS 10 differ, and guidance written for one strands you in another. The
  boot banner names it. Getting this wrong is the single biggest time sink.
- **Confirm a recovery image is reachable (TFTP) before any destructive step.**
  Skipping this check turned a locked controller into a non-starting one for two
  hours.
- **If the unit is second-hand, assume the previous owner still holds the
  GreenLake claim.** That is separate from anything on the box and no console
  work releases it (Step 9).
- If the unit is PoE-powered, have its PSU available before re-cabling (Step 7).

## Why the documented recovery deadlocks

The documented recovery is: log in on the console with the vendor recovery
account (user `password`, password `forgetme!`), then `configure terminal`,
then `mgmt-user admin root`. That account is enabled by default.

On a Conductor-bound controller this deadlocks:

| As the recovery user | Result |
|---|---|
| `configure terminal` | `This controller is managed by a Mobility Conductor.` |
| `write erase all` | `You do not have permission to execute this command` |
| `show version` | `You do not have permission to execute this command` |
| `disaster-recovery on` | permission denied |

The recovery user's only privilege is resetting the admin password; that needs
config mode; config mode is what the Conductor binding blocks.

## Procedure

### Step 1 — Prepare the bootloader

**Tool:** console — power on, interrupt boot with **`Ctrl+X`** (not Enter; the banner says so), then at `cpboot>`:
```
setenv bootdelay 10
setenv netretry yes
saveenv
osinfo
```
**Why:** `bootdelay` is `0` out of the box, so you are racing a zero-second
countdown. `netretry=no` by default makes any network transfer give up on the
first hiccup. `osinfo` reads and verifies both boot partitions.
**Expected result:** `osinfo` lists both partitions with their versions and
image-verify result, e.g.
```
Partition 0:0   ArubaOS <version>   image verify PASS
Partition 0:1   ArubaOS <version>   image verify PASS
```
**If anomaly:** `osinfo` reads the whole image and then hands control back to
autoboot — expect to catch the prompt again. `purgeenv` restores bootloader
defaults. `reload` does not exist at `cpboot>`; the command is `reset`.

### Step 2 — Open config mode at `/mm` (non-destructive; try first)

**Tool:** console, logged in as the recovery user:
```
cd /mm
configure terminal
mgmt-user admin root
```
**Why:** AOS 8 has a node hierarchy. Config mode is refused at the default node
but opened at `/mm`. This is what broke the deadlock.
**Expected result:** config mode opens; `mgmt-user admin root` prompts for the
new admin password. Then `exit`, `write memory`, log back in as `admin`.
**If anomaly:** If config mode is still refused, go to Step 4. Do not jump to the
bootloader before trying this.

### Step 3 — As ADMIN, clear the config and the deployment role

**Tool:** console, logged in as `admin`:
```
write erase all
```
then restart.
**Why:** As admin this clears the config **and the deployment role** and boots
into the Initial Setup dialog. It also drops the licence database — fine on
second-hand kit, whose licences were never yours. It fails as the recovery user
(see the deadlock table).
**Expected result:** The controller boots into Initial Setup. Choose
**standalone** for a controller that is not joining a Conductor.
**If anomaly:** On AOS 6, see Step 6 for the `full-setup` prompt.

### Step 4 — Last resort: bootloader `format 0:2`

**Tool:** console, at `cpboot>`:
```
format 0:2
reset
```
**Why:** Breaks the Conductor binding when nothing else will.
**Expected result:** The binding is gone — **and the ancillary filesystem is
destroyed with it.** Afterwards the box boots partway and stops (Step 5).
**If anomaly:** Do not run this before confirming a recovery image is
reachable. Do not substitute `wipe out flash` as a "smaller hammer" — it erases
the OS image as well and leaves you recovering over TFTP from the bootloader.
`format 0:2` targets the config partition; prefer it.

The community `format 0:2` procedure dates from the AOS 6-era 70xx/72xx. It still
breaks the lock, but on a newer flash layout it takes the ancillary filesystem
with it — a consequence not documented in that guidance.

### Step 5 — "Ancillary files are not present": write the image TWICE

**Tool:** console. After `format 0:2`, boot halts with:
```
Installing ancillary FS   [ FAIL : Ancillary files are not present ]
...
WARNING: An additional image upgrade is required to complete the installation of the
         AP and WebUI files. Please upgrade the boot partition again and reload.
```
**Why:** Read it literally. The first image write installs the OS; a second write
installs the AP/WebUI ancillary files. One write leaves a box that will not finish
starting.
**Expected result:** After two writes of the image to the boot partition and a
restart, the controller boots fully.
**If anomaly:** This is not version-specific — a fresh AOS 8.10 image failed at
exactly the point an AOS 10.2 beta did. The missing files live on shared flash,
not inside either image, which is why the failure follows you across partitions.
If no CLI is reachable to run the copy, use Step 5a.

#### Step 5a — Boot the other partition to get a working CLI

**Tool:** `cpboot>`:
```
def_part 1
reset
```
**Why:** A second bootable image is a working CLI, and a working CLI turns a
bootloader rescue into a routine `copy tftp:`. On the 7005, partition 0:0 (AOS 10.2
beta) was broken by the format while partition 0:1 (AOS 6.5) passed image verify —
that is what saved it.
**Expected result:** AOS 6 tolerated the missing ancillary files and started; AOS 8
and AOS 10 did not. From the working CLI, TFTP the target image, **write it twice**,
then restart.
**If anomaly:** If neither partition boots, recover over TFTP from the bootloader.

### Step 6 — AOS 6 vs AOS 8 differences during setup

**Tool:** console.
**Why:** Commands vanish between generations.
**Expected result / rules:**
- **AOS 8 has no enable mode.** `enable secret` does not exist; you log in
  already privileged (confirmed on the vendor community forum). AOS 6 does need
  `enable`.
- AOS 6 offers `full-setup` / `mini-setup` at an auto-provisioning prompt. **Take
  `full-setup`** — `mini-setup` is the branch role and never offers the
  deployment role.
- That prompt has a short confirm window. Pre-stage `full-setup` and `yes` on the
  clipboard and paste both.
**If anomaly:** Missing the confirm leaves a booted, unconfigured box that will
not re-offer the menu until you `write erase all` again.

### Step 7 — Interface reports up/up but the controller is unreachable

**Tool:** console:
```
show ip interface brief
show port status
show vlan
```
**Why:** Check port-to-VLAN membership before suspecting anything else.
**Expected result (factory-default 7005):**
```
show ip interface brief
  vlan 1     <controller-ip>/24    up   up

show vlan
  VLAN 1      GE0/0/1-0/3     <- where your address probably is
  VLAN 4094   GE0/0/0         <- where your cable probably is
```
**GE0/0/0 sits in VLAN 4094, not VLAN 1.** An address on VLAN 1 reads `up/up` and
is unreachable because VLAN 1 holds no live port. The upstream switch still
learns the MAC (frames arrive untagged), so link state, speed and MAC table all
look healthy and confirm the wrong hypothesis.
**Fix:** move the cable to **GE0/0/1**, already in VLAN 1. No config change, no
restart.
**If anomaly — GE0/0/0 may be POWERING the controller.** The 7005 accepts PoE
and GE0/0/0 is the PoE port, so moving that cable cuts power. Fit the PSU first.
The tell is in the boot log:
```
Power: 802.3at POE+
Cause: POE Power Cycle
```

### Step 8 — Post-recovery verification (MCP)

Once the controller answers on the network and is configured as the server's
AOS8 target:

**Tool:** `health(platform="aos8")`
**Why:** Confirms the server can authenticate to the recovered controller.
**Expected result:** `aos8` reported reachable.
**If anomaly:** Re-check Step 7 cabling/VLAN and the admin credential set in Step 2.

**Tool:** `aos8_get_version()`
**Why:** Confirms the running image and that the intended partition booted.
**Expected result:** The version you wrote in Step 5.

**Tool:** `aos8_get_licenses()`
**Why:** `write erase all` drops the licence database; expect it empty until you
install your own.

**Tool:** `aos8_show_command(command="show switches")` (or `aos8_get_controllers()` if this controller is a Conductor)
**Why:** Confirms the role is standalone / no previous Conductor binding.

**Tool:** `aos8_show_command(command="show vlan")` and `aos8_show_command(command="show port status")`
**Why:** Records the port-to-VLAN state that caused Step 7, as a baseline.

### Step 9 — The GreenLake claim is NOT on the box

**Tool:** GreenLake workspace (device inventory), then an HPE support case.
**Why:** A second-hand controller is very likely still claimed in the previous
owner's GreenLake workspace. Adding it to yours returns:
```
device already exists in workspace or invalid serial/MAC combination
```
That string conflates two causes and cannot distinguish them — do not read it as
a bad serial.
**Expected result:** Confirm the serial is absent from your own workspaces, then
open an HPE support case to have it released.
**If anomaly:** Nothing done on the console changes this.

## Decision matrix

| Condition | Action |
|---|---|
| Recovery user, config mode refused "managed by a Mobility Conductor" | Step 2 (`cd /mm` first) |
| `cd /mm` opened config mode | Set admin password, `write memory`, then Step 3 |
| Admin access obtained | `write erase all`, restart, choose standalone (Step 3 / Step 6) |
| `cd /mm` refused and no admin | Confirm recovery image reachable, then Step 4 |
| Boot halts "Ancillary files are not present" | Step 5 — write the image twice; Step 5a if no CLI |
| Other partition verifies PASS | `def_part`, boot it, TFTP the image from its CLI |
| AOS 6 auto-provisioning prompt | `full-setup`, paste confirm quickly |
| up/up but unreachable | Step 7 — check `show vlan`; move cable to GE0/0/1 (PSU first if PoE) |
| "already exists in workspace" | Step 9 — support case, not a console problem |

## What NOT to do

- Do not run `format 0:2` before confirming a recovery image is reachable.
- Do not trust community guidance without checking the OS generation.
- Do not read "already exists in workspace" as a bad serial.
- Do not call a port or cable dead when the interface reports up/up — check VLAN
  membership first.
- Do not move the GE0/0/0 cable on a PoE-powered unit without the PSU fitted.

## Output formatting

Report as:
1. **Starting state** — OS generation (from the banner), partitions and versions
   from `osinfo`, whether Conductor-bound.
2. **Steps taken** — numbered, each with the exact command and the line of output
   that confirmed it.
3. **Final state** — table from Step 8: version, licences, role, management VLAN /
   port, reachability.
4. **Open items** — GreenLake release case, licence installation, anything not
   verified.

Never include the new admin password in the report.

## Example

> "I bought a used Aruba 7005 and configure terminal says it's managed by a Mobility Conductor"
> "After format 0:2 my controller says Ancillary files are not present"
> "The controller shows vlan 1 up/up but I can't ping it"
