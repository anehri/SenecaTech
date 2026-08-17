# SenecaTech
ServiceNow ITSM  Drone AI custom workflows, scripts, and automations.
demonstration of many skillset of system adminsration by Anas Nehri 

# SenecaTech — AI Device Ordering Platform (ServiceNow Custom Application)

## Objective

> This project is a custom ServiceNow scoped application (`x_1860165_seneca_0`)
> built for **SenecaTech**, a fictional manufacturer of AI-powered devices for
> construction workers (safety drones, AR glasses, smart hats, and environmental
> sensors). It digitizes the full device ordering lifecycle — Service Catalog
> ordering, Flow Designer automation, form-driven auto-population, ACL/RBAC-secured
> data access, and a data import pipeline for historical order records.
>
> Built in a ServiceNow Personal Developer Instance (PDI) and synced to GitHub via
> ServiceNow's native Source Control integration at
> **[github.com/anehri/SenecaTech](https://github.com/anehri/SenecaTech)**.

## Key Skills Demonstrated

- Custom table design (including extending `cmdb_ci` and `task`)
- Service Catalog item design with ordered variables and conditional mandatory fields
- Catalog Client Scripts (onChange logic)
- Record-level Client Scripts with section visibility and reference auto-population (`getReference`)
- Flow Designer automation triggered from a Service Catalog request
- Table-level and field-level Access Control Lists (ACLs), including a scripted/conditional ACL
- Role-Based Access Control (RBAC) with custom scoped application roles
- UI Policies for conditional mandatory fields and field visibility
- Data import pipeline: Excel data source → Import Set staging table → Transform Map → target table
- Event-driven email notifications
- Scoped application development and GitHub source control integration

  ![imgage alt](https://github.com/anehri/SenecaTech/blob/a1201de9e8ebdaa4d0379cf58658bb310abe3d7c/Screenshot%202026-08-13%20181934.png)


## Tools & Technologies Used

- ServiceNow (Personal Developer Instance — PDI)
- ServiceNow Studio / Table Designer
- ServiceNow Service Catalog (Catalog Items, Variables, Catalog Client Scripts)
- ServiceNow Flow Designer
- ServiceNow ACL Editor
- ServiceNow Role Management
- ServiceNow UI Policies & Client Scripts
- ServiceNow Import Sets, Data Sources, and Transform Maps
- ServiceNow Notifications (Email) & Script Events
- ServiceNow Source Control (native GitHub integration)
- Excel (source file for historical order data import)


## Real-World Relevance

This project mirrors how enterprise ServiceNow teams actually build and ship
scoped applications: a Service Catalog front door for end users, Flow Designer
handling the approval/fulfillment logic behind the scenes, tightly scoped ACLs
so each team only touches the data relevant to their job, and a one-time data
migration path (Import Set + Transform Map) for bringing in historical records
from a legacy spreadsheet — a near-universal requirement whenever a manual
process gets replaced by a ServiceNow application.

---

## Application Identity

| | |
| --- | --- |
| **Application Name** | SenecaTech |
| **Scope / Prefix** | `x_1860165_seneca_0` |
| **GitHub Repository** | [github.com/anehri/SenecaTech](https://github.com/anehri/SenecaTech) |
| **Source Control** | Native ServiceNow → GitHub integration (auto-committed from Studio) |

---

## Current vs. Desired State

### Current Process (Manual & Inefficient)

- **Order Placement:** Clients reach out via calls, emails, or other channels
- **Order Processing:** Orders manually entered into spreadsheets by the Device Management team
- **Approval:** Device Management emails the Release Management group for approval
- **Dispatch:** An outsourced team handles device dispatch based on approval

**Challenges with the Manual Process:**
- Manual spreadsheet entry is inefficient and has a high risk of errors
- Lacking a central system makes it hard to track device location and maintain real-time client communication
- Email-based approval delays device delivery
- Difficulty collecting and analyzing client feedback slows product improvement

### Desired Process (Built in This Application)

1. **Client places an order** through the **Order AI Devices** Service Catalog item
2. **Flow Designer** (flow: *Order AI Devices*) picks up the request and drives it through fulfillment
3. **Device Management** owns and reviews device data; **Release Management** and **Dispatch Management** have scoped read/write access appropriate to their role
4. **Client receives status updates**, including an automated email if delivery fails
5. **Historical orders** are migrated in via Excel import rather than re-keyed by hand

---

## Who Uses This Application?

**External:** Clients — place orders through the catalog item; limited, self-service access.

**Internal (custom application roles):**

| Role | Purpose |
| --- | --- |
| `x_1860165_seneca_0.device_management` | Owns the AI Device catalog and Device Request data — the only role with full CRUD across both tables |
| `x_1860165_seneca_0.release_management` | Reviews/approves requests; read access to AI Device and Device Request (Device Request read is governed by a scripted ACL condition) |
| `x_1860165_seneca_0.dispatch_management` | Fulfills approved orders; read access to AI Device, read/write on Device Request to record delivery outcomes |
| `x_1860165_seneca_0.admin` | Full administrative access to the application |

**Note:** Two additional roles (`x_1860165_seneca_0.user` and `x_1860165_seneca_0.ai_devices_user`) exist in the instance from an earlier draft of the ACL design but are not part of the current, cleaned-up ACL matrix documented below — the three operational roles above are what's actually enforced.

---


## Part 1 — Data Model

### Overview

| Table | Extends | Purpose |
| --- | --- | --- |
| **AI Device** (`x_1860165_seneca_0_ai_device`) | `cmdb_ci` | Master catalog of orderable AI devices |
| **Device Request** (`x_1860165_seneca_0_device_request`) | `task` | The actual order record — created from the catalog item, tracked through fulfillment |
| **device data** (`x_1860165_seneca_0_device_data`) | `sys_import_set_row` | Staging table for importing historical/legacy order data from Excel |

### Table 1: AI Device

Extends `cmdb_ci`, so it inherits standard CI fields (model number, short description,
warranty expiration, etc.) in addition to two custom fields:
![imgage alt](https://github.com/anehri/SenecaTech/blob/ac12c7926273270272bdb648742689085678ba61/Screenshot%202026-08-06%20082118.png)

| Field (Label) | Type | Details |
| --- | --- | --- |
| Device Category (`u_choice_1`) | Choice | Safety Drone, AR Glasses, Smart Hats, Environmental Sensor |
| New Price (`u_price_2`) | Price | Unit price used for cost calculations elsewhere in the app |

### Table 2: Device Request

Extends `task`. This is the working order record.

| Field (Label) | Type | Details |
| --- | --- | --- |
| Requester Name (`u_reference_2`) | Reference → `sys_user` | Mandatory |
| Company (`core_company`) | Reference → `core_company` | |
| Device Name (`u_reference_1`) | Reference → AI Device | Mandatory — selecting a device auto-populates Model, Device Description, New Price, and Warranty Expiration via client script |
| Quantity (`u_string_3`) | Integer | Mandatory |
| Business Justification (`u_string_2`) | String | Conditionally mandatory (see UI Policy below) |
| Delivery Date (`u_glide_date_4`) | Date | |
| **Reminder for pick up** (`reminder_for_pickup`) | Date | Added to support the delivery-failure follow-up (see Part 5) |
| **State** (`u_choice_1`) | Choice | Delivered, Delivery failed, In Progress, Canceled |
| Total Cost (`total_cost`) | Currency (read-only) | |
| Model (`u_model`) | String (read-only) | Auto-populated from AI Device |
| Device Description (`u_string_full_utf8_8`) | String (read-only) | Auto-populated from AI Device |
| New Price (`u_price_10`) | Price (read-only) | Auto-populated from AI Device |
| Warranty Expiration (`u_glide_date_11`) | Date (read-only) | Auto-populated from AI Device |
| Price (`u_string_full_utf8_9`) | String (read-only) | |
| Demo (`demo`) | String | |

![imgage alt](https://github.com/anehri/SenecaTech/blob/5132a20e60cc7554b9208ba78aa6b8b3e9650a8d/Screenshot%202026-08-10%20153148.png)
### Table 3: device data (Import Staging Table)

Extends `sys_import_set_row`. Used purely as the landing table for the Excel data
source described in Part 6 — not used operationally once the transform has run.

| Field | Import Attribute |
| --- | --- |
| Device Name | Device Name |
| Ordered By | Ordered By |
| Company | Company |
| Quantity | Quantity |
| Justification | Justification |
| State | State |
| Delivery Date | Delivery Date |

---


## Part 2 — Access Control Lists (ACLs)

Access is enforced with a consistent pattern: **Device Management owns the data**
(full CRUD), while **Release Management and Dispatch Management get the specific
read/write access their role in the workflow actually requires**.


### AI Device — Table & Field ACLs

| Operation | Roles Granted Access |
| --- | --- |
| create | `device_management` |
| read | `device_management`, `release_management`, `dispatch_management` |
| write | `device_management` |
| delete | `device_management` |

**Field ACL — `owned_by`:** read access limited to `release_management`, `device_management`
![image alt](https://github.com/anehri/SenecaTech/blob/0bcaffb7d3fcb40761674324548bf7be382da785/Screenshot%202026-08-10%20163812.png)
### Device Request — Table ACLs

| Operation | Roles Granted Access | Notes |
| --- | --- | --- |
| create | `device_management` | |
| read | `device_management`, `dispatch_management`, `release_management` | The `release_management` read ACL is **scripted** — access additionally depends on a script condition returning `true`, not role membership alone |
| write | `device_management`, `dispatch_management` | Dispatch gets write access specifically so delivery outcome (State, Reminder for pick up) can be recorded |
| delete | `device_management` | |

**Design note:** early in development, ACLs also existed for a generic `user` role
and an `ai_devices_user` role (open create/read/write/delete on AI Device). These
were superseded by the current, tighter three-role model above — the final ACL
set only grants the three operational roles the access shown here.

---



## Part 3 — Module & Application Access

| Module | Type | Target | Roles Granted Access |
| --- | --- | --- | --- |
| Device Details | List | AI Device | `device_management`, `release_management`, `dispatch_management` |
| Upload a Device | New Record | AI Device | `device_management` |
| All Device Requests | List | Device Request | `device_management`, `release_management`, `dispatch_management` |
| Create New Device Request | New Record | Device Request | `device_management` |
| Active Device Request | List (filtered) | Device Request | `device_management` |
| device data | List | Import staging table | Restricted to data-import/admin use |

Module-level access controls what each role sees in the **SenecaTech** application
menu — table ACLs stop unauthorized data access underneath, while module scoping
keeps the navigation itself clean and role-appropriate (Dispatch and Release users
aren't shown "Create" options they aren't permitted to use).

---
![image alt](https://github.com/anehri/SenecaTech/blob/0bcaffb7d3fcb40761674324548bf7be382da785/Screenshot%202026-08-10%20162831.png)


## Part 4 — Order AI Devices (Service Catalog Item)

**Short description:** *"You can order your AI devices from catalog item"*

### Catalog Variables (in display order)

| # | Question | Variable Name | Type | Mandatory |
| --- | --- | --- | --- | --- |
| 100 | Requester Name | `requester_name` | Reference → `sys_user` | No (auto-populated, effectively read-only) |
| 200 | Requester Company | `requester_company` | Reference → `core_company` | No (auto-populated) |
| 300 | Choose your AI Device | `choose_your_ai_device` | Reference → AI Device | **Yes** |
| 400 | Choose Quantity | `choose_quantity` | Numeric | **Yes** |
| 500 | Business Justification | `business_justification` | Multi-line text | Conditional — becomes mandatory if quantity > 3 |
| 600 | Delivery Details | `delivery_details` | Multi-line text | **Yes** |

### Catalog Client Script — "Field validation( biz just)"

An `onChange` script on **Choose Quantity** enforces the conditional requirement:

```javascript
function onChange(control, oldValue, newValue, isLoading) {
    if (isLoading || newValue == '') {
        return;
    }
    var quantity = g_form.getValue('choose_quantity');
    if (quantity > 3) {
        g_form.setMandatory('business_justification', true);
    } else {
        g_form.setMandatory('business_justification', false);
    }
}
```

This is the real, committed implementation of the "Business Justification required
if quantity is more than 3" requirement from the original design spec.

---


## Part 5 — Flow Automation & Form Behavior

### Flow: "Order AI Devices"

A **Flow Designer** flow (internal name `order_ai_devices`) is triggered when the
Order AI Devices catalog item is submitted, and drives the request through the
approval/fulfillment lifecycle described in the original process design (group
approval → Dispatch task → Device Request record creation on completion).

### Device Request Form — Client Scripts

Two client scripts on the Device Request table auto-populate device details the
moment a device is selected, rather than requiring the requester or Device
Management to look up specs manually:

**"Show the section(device name selected)"** — `onChange` of Device Name:
```javascript
function onChange(control, oldValue, newValue, isLoading, isTemplate) {
    if (isLoading || newValue === '') {
        return;
    }
    if (newValue !== '') {
        g_form.setSectionDisplay('device_details', true);
        g_form.setSectionDisplay('delivery_details', true);
        g_form.getReference('u_reference_1', getDetails);
    }
    function getDetails(details) {
        g_form.setValue('u_model', details.model_number);
        g_form.setValue('u_string_full_utf8_8', details.short_description);
        g_form.setValue('u_price_10', details.u_price_2);
        g_form.setValue('u_glide_date_11', details.warranty_expiration);
    }
}
```
![imgage alt](https://github.com/anehri/SenecaTech/blob/0bcaffb7d3fcb40761674324548bf7be382da785/Screenshot%202026-08-13%20131619.png)

**"Hide the section(details delivery)"** — `onLoad`, mirrors the same logic so
the Device Details and Delivery Details sections start hidden on a blank form
and populate correctly when reopening a record that already has a device selected.

### UI Policies on Device Request

| Policy | Purpose |
| --- | --- |
| Mandatory Business Justification if quantity is more than 2 | Form-level equivalent of the catalog client script's conditional requirement |
| reminder to pick field visibility | Controls when the **Reminder for pick up** field is shown, tied to the Delivery Failed state |
| Hide/Mandatory Delivery date field | Conditional handling of the Delivery Date field |

---
![imgage alt](https://github.com/anehri/SenecaTech/blob/0bcaffb7d3fcb40761674324548bf7be382da785/Screenshot%202026-08-10%20115326.png)

## Part 6 — Delivery-Failure Handling (Enhancement)

### Original Problem

The initial design always marked a completed dispatch as **Delivered**, with no
way to record or follow up on failed deliveries (wrong address, unreachable
client, no one available to receive the device).

### What Was Built

- **State** choice list expanded to include `Delivered`, `Delivery failed`,
  `In Progress`, and `Canceled`
- **Reminder for pick up** date field added to Device Request, with its own UI
  Policy controlling visibility
- A dedicated **script event** — `x_1860165_seneca_0.pickup` — registered against
  the Device Request table
- A **Notification** ("Reminder to the user") tied to that event, with the exact
  subject and message committed in the app:

  **Subject:** `Unable to deliver ${u_reference_1} / ${number}`

  **Body:**
  > *"Hey ${u_reference_2.first_name}, This is regarding your order number
  > Number: ${number}, We have tried to deliver your Device Name: ${u_reference_1}
  > but we could not deliver. as per our company policy we will deliver this to
  > your company location and would request you collect the same."*

### Status of the Recurring Check

The original design called for a **Scheduled Job running every 3 hours** to scan
for `Delivery failed` records with an empty Reminder for pick up field and fire
the `x_1860165_seneca_0.pickup` event automatically. The event registration and
notification template are committed and ready — the Scheduled Job itself was not
found in the current source export, so it's the clear next build step to close
the loop on this enhancement (see Part 9).

---
![imgage alt](https://github.com/anehri/SenecaTech/blob/0bcaffb7d3fcb40761674324548bf7be382da785/Screenshot%202026-08-13%20181928.png)

## Part 7 — Historical Data Import

Legacy order records (originally tracked in a spreadsheet) are brought into the
application through a standard ServiceNow import pipeline rather than manual
re-entry:

1. **Data Source:** `device_orders.xlsx (Uploaded)` — Excel file type, targeting the `device data` import set table
2. **Import Set Table:** `x_1860165_seneca_0_device_data` (extends `sys_import_set_row`)
3. **Transform Map:** "Device transform" — maps `device data` → `Device Request`

### Sample Source Data

| Device Name | Ordered By | Company | Quantity | Justification | State | Delivery Date |
| --- | --- | --- | --- | --- | --- | --- |
| ES Sensor | Allan Schwanted | Amazon | 3 | Required for building construction | Delivered | 20/12/2022 |
| SD Power Drone | Amos Linan | APC | 3 | We need to deploy 3 power drones to have control on the construction area | Delivered | 12/06/2022 |
| SH Super Cap | Angelo Ferentz | Apple | 2 | | Delivered | 11/02/2021 |
| ARG Smart Wear | Brant Darnel | AWS | 1 | | Delivered | 10/4/2021 |

**Real-world relevance:** date format handling was a specific challenge during
this import — ServiceNow Transform Maps have no pure configuration-based way to
convert `DD/MM/YYYY` to `MM/DD/YYYY`, so dates were normalized in Excel prior to
import as the most reliable no-code workaround.

---


## Part 8 — Security Testing & Validation

### Test Scenarios

#### Scenario 1: `device_management` — Full CRUD on AI Device

Log in as a user with `x_1860165_seneca_0.device_management`, create a new AI
Device via **Upload a Device**, edit it, then delete it. All four operations
should succeed.

#### Scenario 2: `release_management` — Scripted Read Access on Device Request

Log in as a user with `x_1860165_seneca_0.release_management` and open **All
Device Requests**. Because Device Request read access for this role is governed
by a scripted ACL condition (not plain role membership), confirm the visible
record set matches what the script is intended to allow — this is worth testing
explicitly since scripted ACLs can behave differently than a flat role check.

#### Scenario 3: `dispatch_management` — Write on Device Request, No Write on AI Device

Log in as a user with `x_1860165_seneca_0.dispatch_management`. Confirm the
**State** and **Reminder for pick up** fields on a Device Request record can be
updated, but any attempt to create or edit an AI Device record is blocked.

#### Scenario 4: Catalog Item — Conditional Business Justification

Submit **Order AI Devices** with Quantity ≤ 3 — Business Justification should
remain optional. Change Quantity to 4+ — Business Justification should become
mandatory before the form can be submitted, per the committed catalog client
script.

#### Scenario 5: Device Selection Auto-Population

On a Device Request record (or during catalog submission), select a Device Name
and confirm Model, Device Description, New Price, and Warranty Expiration
populate automatically from the referenced AI Device record, and that the
Device Details / Delivery Details sections become visible.

#### Scenario 6: Delivery Failure Notification

Manually fire the `x_1860165_seneca_0.pickup` event on a Device Request record
(or set State to Delivery failed once the follow-up automation is completed) and
confirm the requester receives the "Reminder to the user" email with the correct
order number and device name substituted into the subject and body.

---
![imgage alt](https://github.com/anehri/SenecaTech/blob/0bcaffb7d3fcb40761674324548bf7be382da785/Screenshot%202026-08-13%20190754.png)

## Part 9 — Next Steps / Known Gaps

- **Scheduled Job for delivery-failure follow-up:** build the recurring
  (every-3-hours) job that scans Device Request for `Delivery failed` records
  with an empty **Reminder for pick up** field, sets the timestamp, and fires the
  `x_1860165_seneca_0.pickup` event — the event and notification are ready, this
  is the missing trigger.
- **Legacy roles cleanup:** `x_1860165_seneca_0.user` and
  `x_1860165_seneca_0.ai_devices_user` remain in the instance from an earlier
  ACL draft and should be retired or documented if intentionally kept.
- **UI Policy threshold consistency:** the Device Request form's UI Policy label
  references "more than 2" while the catalog client script enforces "more than
  3" — worth reconciling to a single documented threshold.
- **Reporting/dashboard:** no reports were found in the current export; adding
  order-volume and cycle-time reporting would round out the Leadership-facing
  value described in this README.

---


## Summary of Skills Demonstrated

| Skill | Where It Shows Up |
| --- | --- |
| Custom table design, extending base tables | AI Device extends `cmdb_ci`; Device Request extends `task` |
| Service Catalog design | Order AI Devices item, 6 ordered variables |
| Catalog Client Scripting | Conditional mandatory field (`onChange`) |
| Client Scripting with reference lookups | `getReference()` auto-population of device specs |
| Flow Designer | "Order AI Devices" flow driving the fulfillment lifecycle |
| Table & field-level ACLs | Role-scoped CRUD across two tables, including a scripted ACL |
| RBAC / custom application roles | Three operational roles with distinct, minimal permissions |
| UI Policies | Conditional mandatory/visible fields on the Device Request form |
| Import Sets & Transform Maps | Excel → staging table → target table pipeline |
| Event-driven notifications | Script event + email notification for delivery failures |
| Source control integration | App synced live to GitHub via ServiceNow Studio |

---


## Notes

This application was developed using a ServiceNow Personal Developer Instance
(PDI), available free through the ServiceNow Developer Program at
developer.servicenow.com, and is version-controlled directly from ServiceNow
Studio to GitHub. All data, users, and test records are fictional and created
for demonstration purposes. No production data was used.

---

**GitHub Repository:** [github.com/anehri/SenecaTech](https://github.com/anehri/SenecaTech)

**GitHub Username:** anehri

**Application Scope:** `x_1860165_seneca_0`

**Last Updated:** 08/17/2026
