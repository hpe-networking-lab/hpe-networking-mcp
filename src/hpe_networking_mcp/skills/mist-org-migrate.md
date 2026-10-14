---
name: mist-org-migrate
title: Juniper Mist org-to-org migration — clone config, move devices, verify
description: |
  TRIGGERS — call this when the user asks to consolidate, merge or migrate
  one Juniper Mist org into another, move APs / switches / gateways between
  Mist orgs, decommission an old org and bring its devices across, or plan
  a tenant migration. Match phrases: "Mist org migration", "consolidate
  Mist orgs", "merge the two Mist orgs", "move devices to the other org",
  "release and claim", "re-claim switches into the new org", "org
  decommission". Order of work: baseline both orgs → clone config into the
  target inert → release from source → claim + assign in target → verify
  against the consuming objects. Orgs and sites are addressed by ID only.
  Flags the switch-release zeroize trap before any switch is moved.
platforms: [mist]
tags: [mist, migration, org, inventory, claim, release, change-management]
tools: [health, mist_get_self, mist_get_org_settings, mist_get_org_licenses_summary, mist_list_org_sites, mist_list_org_site_groups, mist_get_site_setting, mist_update_site_settings, mist_list_org_templates, mist_list_org_wlans, mist_list_org_rf_templates, mist_list_org_network_templates, mist_list_org_site_templates, mist_create_org_template, mist_create_org_wlan, mist_create_org_rf_template, mist_create_org_network_template, mist_create_org_site_template, mist_create_org_site, mist_clone_org, mist_get_org_inventory, mist_search_org_inventory, mist_update_org_inventory_assignment, mist_add_org_inventory, mist_search_org_devices, mist_list_site_devices, mist_get_site_device, mist_get_site_device_config_cmd, mist_search_org_device_events, mist_search_org_alarms, mist_list_org_audit_logs, mist_get_tool_schema]
---

# Juniper Mist org-to-org migration — clone config, move devices, verify

## Objective

Move configuration and devices from a **source** Mist org into a **target**
Mist org without losing config, stranding devices, or leaving the target
half-built. The order is fixed:

1. Baseline both orgs.
2. Clone the needed config into the target **inert** (nothing bound,
   WLANs disabled).
3. Release devices from the source in planned batches.
4. Claim them into the target and assign to the correct target site.
5. Verify against the objects the platform actually consumes, then
   activate site-by-site.
6. Decommission the source only after the target is verified.

This is **change-making on two orgs**. Every write needs operator approval
covering both orgs, a change window and a revert path. Template work
follows `mist-template-rollout`; snapshot with `change-pre-check` before
and `change-post-check` after each batch.

## Prerequisites

- `health(platform="mist")` ok; `ENABLE_MIST_WRITE_TOOLS` set for the
  write steps.
- **Source and target `org_id` resolved from the token** via
  `mist_get_self()` (`privileges[].org_id`) — never by org name. Two orgs
  can carry the same display name; a name-based lookup can write to the
  wrong one.
- The list of devices to move (name + MAC / device id) and the target
  `site_id` for each.
- Claim / activation codes held **out-of-band** — they are credentials.
  Never put them in chat or version control.
- Target org **subscriptions** checked (Step 1): a device claimed into an
  org without the matching subscription is adopted but stays
  monitor-only (for switches, the config UI is locked) until the
  subscription is present in that org.

## Procedure

### Step 1 — Baseline both orgs

**Tools (run against each org_id):** `mist_get_org_settings`,
`mist_list_org_sites`, `mist_list_org_site_groups`,
`mist_list_org_templates`, `mist_list_org_wlans`,
`mist_list_org_rf_templates`, `mist_list_org_network_templates`,
`mist_list_org_site_templates`, `mist_get_org_inventory`,
`mist_get_org_licenses_summary`, `mist_get_site_setting(site_id=...)` per
affected site, `mist_search_org_alarms(org_id=..., duration="7d")`.
**Why:** the diffable before-state and the rollback reference for both
orgs.
**Expected result:** saved JSON for every object above plus a device
manifest (name, MAC, serial, model, source site_id, target site_id).
**If anomaly:** stop if either baseline is incomplete.

### Step 2 — Reconcile what actually needs moving

**Tools:** `mist_search_org_inventory(org_id=<target>, mac=...)` and
`mist_search_org_inventory(org_id=<source>, mac=...)` for each device.
**Why:** devices are often already claimed in the target (e.g. claimed at
order time). Releasing/claiming those is a no-op; only move what is
genuinely in the source.
**Expected result:** each device classified as *in source only*, *already
in target*, or *missing from both*.
**If anomaly:** a device present in both orgs, or a device that appears in
the target during the migration that no step claimed (e.g. a powered-on
device auto-adopting) → stop and confirm with the operator before
continuing; it may indicate a second operator working the org.

### Step 3 — Classify every switch before planning its move

**Tools:** `mist_get_site_device(site_id=..., device_id=...)` and
`mist_get_site_device_config_cmd(site_id=..., device_id=...)`.
**Why:** releasing a switch that is **not cloud-native** (a brownfield EX
adopted into Mist via CLI rather than claim-code ZTP) from an org
triggers a **zeroize + restart**: it wipes to factory default, drops its
data plane, and will **not** phone home — it returns only through a
**manual adoption at the console**. Its Mist device config object is also
purged on release (the device GET returns 404 afterwards).
**Expected result:** each switch marked *cloud-native* or *CLI-adopted*;
device-level config and the rendered `config_cmd` saved for every switch.
**If anomaly:** a CLI-adopted switch is a **maintenance-window outage on
that device**, not a management blip. State it in those terms, get
explicit go, have console / on-site access and the target-org adoption
command ready, and confirm the management / uplink path survives the
wipe.

### Step 4 — Clone config into the target, inert

**Tools:** for each object: GET from the source, strip `id`, `org_id`,
`site_id`, `for_site` and timestamp fields, remap any references (e.g. a
WLAN's `template_id`) to the new target IDs, then POST with
`mist_create_org_template`, `mist_create_org_wlan`,
`mist_create_org_rf_template`, `mist_create_org_network_template`,
`mist_create_org_site_template`; create target sites with
`mist_create_org_site` if needed. Check each body shape with
`mist_get_tool_schema(name=...)` before writing.
**Why:** the target must be ready before any device moves so downtime is
bounded. Cloning live source objects keeps every sub-structure the UI
renders.
**Inert rules:** WLANs `enabled: false`; no template bound to a site.
Record every created object (name + new id) in a revert manifest.
**Site variables:** write them on the **Site Setting** object
(`mist_update_site_settings`), not the Site object. Read the target site's
existing `vars` first and reuse existing names (case-insensitive match)
rather than minting duplicates. A PUT to site settings replaces whole
nested objects — send the complete object, not one key.
**Alternative when the target org does not exist yet:**
`mist_clone_org` creates a new org from the source; it copies org
settings, templates, WxLAN tags/tunnels/rules and org WLANs, but **not**
sites or site groups, so the copied templates are not applied anywhere.
**Expected result:** all objects created; alarms on the target 0 → 0;
lint and the render gate from `mist-template-rollout` clean.
**If anomaly:** revert by deleting the manifest objects (WLANs first, then
templates).

Secrets inside cloned objects (RADIUS secrets, WLAN passphrases) may be
absent from the source read or may travel machine-to-machine. Verify
**presence** on the target, never print values; set missing ones
out-of-band.

### Step 5 — Release devices from the source (per batch)

**Tool:** `mist_update_org_inventory_assignment(org_id=<source>, body={...})`
— `PUT /orgs/{org_id}/inventory`, which supports assign, unassign and
delete operations keyed by MAC or serial. Releasing from the org is the
delete operation; confirm the exact body with
`mist_get_tool_schema(name="mist_update_org_inventory_assignment")`.
**Why:** a device can be claimed into only one org.
**Before releasing:** record each device's claim code (out-of-band) — it
is the revert path — and confirm Step 3 config backups exist.
**Expected result:** device absent from the source inventory.
**If anomaly:** stop the batch; do not continue to the next device.

### Step 6 — Claim into the target and assign to the site

**Tools:** `mist_add_org_inventory(org_id=<target>, body=[<claim codes>])`
(`POST /orgs/{org_id}/inventory`), then
`mist_update_org_inventory_assignment(org_id=<target>, body={"op": "assign", "site_id": ..., "macs": [...]})`.
**Why:** puts the device in the target org on the intended site.
**Expected result:** device in target inventory with the correct
`site_id`. Expect `connected: false` until the device next checks in
(APs not yet powered stay that way until bring-up). Assigning to a site
with no bound templates pushes no config.
**If anomaly:** a CLI-adopted switch will not reconnect by itself — run
the target-org adoption at the console (Step 3). Revert for an assign is
the unassign operation on the same MACs.

### Step 7 — Recover switch config if it was lost

**Tool:** `mist_search_org_device_events(org_id=<source>, mac=..., duration="30d")`.
**Why:** if a switch's device config was purged on release before it was
backed up, the source org still holds its device events. Concatenating
the `config_diff` fields oldest → newest reconstructs the operator-applied
Junos config.
**Expected result:** a `show | compare`-style history.
**Caveat:** it is the applied history with collapsed context, not a
guaranteed-complete running config; reset any secrets out-of-band.

### Step 8 — Verify against the consuming objects

**Tools:** `mist_search_org_devices(org_id=<target>, mac=...)`,
`mist_list_site_devices(site_id=...)`,
`mist_get_site_device_config_cmd(site_id=..., device_id=...)` for
switches, `mist_get_site_setting(site_id=...)` for vars,
`mist_search_org_alarms` on **both** orgs,
`mist_list_org_audit_logs(org_id=..., duration="1d")`.
**Why:** a write returning 200 proves nothing. Read what the platform
uses: the site's Site Setting for vars, the rendered `config_cmd` for
switches, device connection state for adoption.
**Expected result:** every moved device is in the target (by MAC), on the
right `site_id`, connected (or explicitly deferred to bring-up), gone
from the source; no new alarms on either org; dependent integrations (for
example RADIUS / CoA with ClearPass — see `mist-clearpass-guest-coa`)
still work.
**If anomaly:** revert the batch from the manifest and claim codes.

### Step 9 — Activate, then decommission the source

Bind templates / enable WLANs **one site at a time** per
`mist-template-rollout`, re-running alarms and the render gate after each.
Decommission the source org only after the target is verified, and keep
the Step 1 baseline as the rollback point.

## Decision matrix

| Condition | Action |
|---|---|
| Org identified only by name | Resolve the `org_id` via `mist_get_self()` first; never write by name |
| Device already claimed in target | Skip release/claim; only assign to site |
| Switch is CLI-adopted (not cloud-native) | Treat release as a windowed outage + manual console re-adopt; back up config first |
| Target org lacks the matching subscription | Device will be monitor-only after claim — resolve the subscription before relying on config |
| Unexpected device appears in target mid-migration | Pause; confirm it is expected bench gear and that no second operator is working the org |
| Template references site variables | Write vars on Site Setting; reuse existing names; don't write vars for templates that are not yet bound |
| Target org does not exist yet | `mist_clone_org` is an option (no sites / site groups copied) |
| Mist `unavailable` | Stop |

## Output formatting

```
## Mist org migration — <source org_id> → <target org_id>
**Phase:** baseline | clone | move batch <n> | verify | activate
**Captured:** <ISO timestamp>

### Device manifest
| Device | MAC | Type | Cloud-native? | Source site_id | Target site_id | State |
|---|---|---|---|---|---|---|

### Objects created in target (revert manifest)
| Kind | Name | New id |
|---|---|---|

### Verification
- In target / correct site: <n>/<N>
- Removed from source: <n>/<N>
- Connected: <n>  Deferred to bring-up: <n>
- Alarms source: <before> → <after>; target: <before> → <after>

### Open items
- <subscriptions, out-of-band secrets, manual re-adopts>

### Revert
- <per-batch revert steps>
```

## Example

> "consolidate our old Mist org into the new one and move the APs and switches"
> "move these three switches to the other Mist org"
> "plan the decommission of the legacy Mist org"
