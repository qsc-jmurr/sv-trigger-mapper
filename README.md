# Seervision Trigger Mapper (Q-SYS plugin)

Maps Seervision / VisionSuite zone-triggered events to control changes on any other named component in a Q-SYS design. When a camera zone trigger goes true, every rule pointing at it applies its configured value to its target control.

| | |
|---|---|
| Plugin file | `vsmanager control.qplug` |
| Plugin name in Designer | `QSC~Seervision Trigger Mapper v<version>` |
| Plugin ID | `3e5605bd-9c7f-4614-84ec-e9ccb36b4370` |
| Current version | **0.0.0.14** (development, not production-ready) |

This README is the source of truth for how the plugin behaves and how it's installed. Update it in the same change as the code.

## Contents

- [What it does](#what-it-does)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Using the plugin](#using-the-plugin)
- [Worked examples](#worked-examples)
- [Control reference](#control-reference)
- [How it works (for maintainers)](#how-it-works-for-maintainers)
- [Customising the plugin](#customising-the-plugin)
- [Known limitations](#known-limitations)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Versioning](#versioning)
- [Changelog](#changelog)

---

## What it does

Seervision exposes zone triggers for each camera (conductor cameras included) as Boolean controls on its Q-SYS plugin. On its own, using those triggers means writing a Control Script or wiring control pins for every action. The Trigger Mapper replaces that with a table of rules:

> **When** *Seervision trigger X* goes true, **set** *property P* of *control C* on *component K* **to** *value V*.

Key features:

- **No scripting or wiring.** Everything is picked from dropdowns that are filled from the live design.
- **Any target.** Any named component with Script Access and any of its writable controls can be a target: mixers, gains, snapshots, routers, camera presets, UCI controls, other plugins.
- **Type-aware.** The Property and Value fields change to suit the target control (Boolean, number, text or trigger).
- **Stacking.** Several rules can share one trigger. All of them fire, in rule order.
- **Per-rule arming and status.** Each rule has its own On button and status LED. A broken rule faults only its own LED.
- **Self-healing discovery.** The design is re-scanned every 10 seconds, so components that appear later (or a Seervision plugin that starts late) are picked up without a restart.
- **Built-in Inspector.** Browse any component's control names, types and directions without leaving the plugin.

---

## Requirements

- Q-SYS Designer (the plugin uses the standard `Component`, `Timer` and `Controls` runtime APIs).
- The **Seervision / VisionSuite plugin** in the same design. The mapper finds it by its plugin Type string:
  `%PLUGIN%_b26588a3-524c-4809-a7c2-9c31d9cda097_%FP%_a2fd6c6c6e0650e986935f1af43ea354`
  If a new Seervision plugin release changes that string, its instances won't appear in the dropdown. See [Troubleshooting](#troubleshooting).
- **Script Access** enabled on every component the mapper needs to reach (the Seervision component and every target component). Without it, `Component.New` and `Component.GetControls` can't see them.

## Installation

1. Close Q-SYS Designer.
2. Copy `vsmanager control.qplug` into the Designer plugins folder:
   `%USERPROFILE%\Documents\QSC\Q-Sys Designer\Plugins\`
   If you're upgrading, overwrite the existing file. The plugin ID is unchanged, so instances already in a design upgrade in place.
3. Open Designer. The plugin appears in the **Schematic Elements → Plugins** pane as *Seervision Trigger Mapper*.
4. Drag it into the design.
5. In the plugin's **Properties**, set **Rule Count** (1–32, default 4) to the number of rule rows you need. Changing it rebuilds the controls.

### Prepare the design

1. Select the Seervision plugin. In its Properties, set **Script Access** to `Script` or `All`. Give it a clear name, which is the name you'll pick in the mapper.
2. Do the same for every component you want rules to control (mixers, snapshots, routers, UCI controls and so on).
3. Save to the Core and run, or run in Emulate.

> The mapper only does anything while the design is running (on a Core or in Emulate). In design mode the dropdowns are empty.

---

## Quick start

1. Install the plugin and enable Script Access on the Seervision plugin and your targets (see above).
2. Run the design and open the mapper.
3. On the **Rules** page, pick your Seervision instance in **Seervision Component**.
4. In Rule 1:
   - **Trigger**: a zone trigger, e.g. one ending in `Triggered`.
   - **Target Component** and **Target Control**: what you want to change.
   - **Property** and **Value**: what to set it to.
   - Turn **On** on. The LED goes green.
5. Walk into the zone. The target changes and the LED stays green. If it turns red, read the rule's status text (the `rule_status_N` control's string) and see [Status LED](#status-led).

---

## Using the plugin

### Properties

| Property | Type | Range | Default | Effect |
|---|---|---|---|---|
| **Rule Count** | Integer | 1–32 | 4 | Number of rule rows on the Rules page. The Rules page grows by 24 px per row. |

### Rules page

1. **Seervision Component** – choose the Seervision instance. Only components of the Seervision plugin type are listed.
2. For each rule row:

   | Column | Purpose |
   |---|---|
   | **On** | Arms or disarms the rule. Turning it on doesn't fire the action. |
   | **Trigger** | A Seervision control whose name contains `Triggered`. This covers every camera's zone triggers, conductor cameras included. `none` unhooks the row. |
   | **Target Component** | Any named component in the design. |
   | **Target Control** | Any control on that component that isn't read-only. |
   | **Property** | What gets set. The options depend on the control's type: Boolean → `Boolean`; Float/Integer → `Value` or `Position`; Text → `String`; Trigger → `Trigger`. |
   | **Value** | The value to apply. The input shown matches the property: true/false, a number, or text. `Trigger` needs no value. |
   | **Status** | Rule state. See [Status LED](#status-led). |
   | **Clear** | Resets the row: disabled, trigger `none`, target `none`, default values. Its action is removed from the trigger at the same time. |

Changing **Target Component** reloads the **Target Control** list, which in turn reloads **Property** and the visible **Value** field. Existing selections are kept wherever they're still valid.

### Properties and values

| Property | Offered for | Value field | What happens when the rule fires |
|---|---|---|---|
| `Boolean` | Boolean controls (buttons, mutes, toggles) | Dropdown: `true` / `false` | `control.Boolean = <value>` |
| `Value` | Float and Integer controls | Number | `control.Value = <number>`, in the control's own units (e.g. dB for a gain) |
| `Position` | Float and Integer controls | Number | `control.Position = <number>`, normalised 0–1 across the control's range |
| `String` | Text controls | Text | `control.String = <text>` |
| `Trigger` | Trigger buttons (snapshot loads, preset recalls) | None | `control:Trigger()` |

Notes:

- If the target control's type can't be read (or isn't one of the above), the Property list falls back to `Value` and `String`.
- A number field that isn't a valid number is applied as `0`.
- Values outside the control's range are handled by Q-SYS: some are clamped, others are rejected and show `Apply failed: …` on the LED.
- `Boolean` set to `true` on a momentary button acts as a press. If you want a one-shot action, prefer a `Trigger` control when the target has one.

### Stacking triggers

Several rules can use the **same trigger**. When it goes true, **all** of them run, in rule-number order (Rule 1 before Rule 3). For example, one zone trigger can recall a camera preset, unmute a mic and fire a snapshot.

To remove an action from a stack, set that row's Trigger to `none`, point it at a different trigger, or press **Clear**. The other rules on that trigger keep working. When no rule references a trigger, the mapper unhooks that control entirely.

Rules only act when the trigger goes **true**. Nothing happens when it goes false. To do something when a zone is left, use the matching zone control if Seervision provides one, or add a second rule on a trigger that goes true on exit.

Disabled rules in a stack are skipped: their LED shows `Disabled` and the remaining rules still run.

### Status LED

| State | Status text | Meaning |
|---|---|---|
| OK (green) | *(blank)* | Rule is armed, or its last apply succeeded. |
| Fault | `No target selected` | Trigger fired but no target component or control is set. |
| Fault | `Target component/control not found` | Target was renamed or removed, or has no Script Access. |
| Fault | `Apply failed: …` | Q-SYS rejected the value (e.g. out of range). |
| Missing (off) | `Disabled` | Rule's **On** button is off. |

A fault is only reported when the trigger fires. A rule with a bad target shows green while armed until its first fire. Toggling **On** off and on resets a faulted LED to green.

### Inspector page

Pick any named component to list each of its controls as:

```
<control name>  [<Type>, <Direction>]
```

Use it to confirm Seervision's real trigger control names, or to check a target's control names and types before building a rule. If the component has no Script Access, the output reads `Could not read controls for <name>`.

### Background refresh

Every 10 seconds the plugin re-scans the design. It:

- refreshes the **Seervision Component** list and binds to the selected instance if it wasn't bound yet (for example, if Seervision started after the mapper);
- refreshes every row's **Trigger** list and hooks up any newly referenced triggers;
- refreshes every row's **Target Component** list and the Inspector's component list.

Saved selections are kept while their entries still exist. While the Seervision component can't be reached, saved trigger selections are left as they are rather than being reset to `none`.

---

## Worked examples

### 1. Follow the presenter with audio

A presenter zone should unmute the lectern mic and mute the audience mics.

| Rule | Trigger | Target Component | Target Control | Property | Value |
|---|---|---|---|---|---|
| 1 | `Cam1 Lectern Triggered` | `Mixer` | `input.1.mute` | Boolean | false |
| 2 | `Cam1 Lectern Triggered` | `Mixer` | `input.5.mute` | Boolean | true |

Both rules share the trigger, so both run when the zone goes true.

### 2. Recall a snapshot and set a level

| Rule | Trigger | Target Component | Target Control | Property | Value |
|---|---|---|---|---|---|
| 1 | `Cam2 Panel Triggered` | `Room Snapshots` | `load.3` | Trigger | — |
| 2 | `Cam2 Panel Triggered` | `Program Gain` | `gain` | Value | -12 |

### 3. Drive a UCI or text display

| Rule | Trigger | Target Component | Target Control | Property | Value |
|---|---|---|---|---|---|
| 1 | `Cam1 Whiteboard Triggered` | `UCI Vars` | `mode_label` | String | Whiteboard |

The trigger and control names above are illustrative. Use the Inspector to find the real names in your design.

---

## Control reference

These are the plugin's own controls. They can be read or set by other scripts (`Component.New("<mapper name>")["<control>"]`), exposed on UCIs, or pinned out like any other plugin control. `N` is the rule number, 1 to Rule Count.

| Control | Type | Default | Description |
|---|---|---|---|
| `sv_component` | Text (combo) | `none` | Selected Seervision component name. |
| `inspect_component` | Text (combo) | `none` | Component shown on the Inspector page. |
| `inspect_output` | Text | *(blank)* | Inspector listing. |
| `rule_enable_N` | Toggle button | off | Arms rule N. |
| `rule_trigger_N` | Text (combo) | `none` | Seervision trigger control name. |
| `rule_target_component_N` | Text (combo) | `none` | Target component name. |
| `rule_target_control_N` | Text (combo) | `none` | Target control name. |
| `rule_target_property_N` | Text (combo) | `Value` | `Boolean`, `Value`, `Position`, `String` or `Trigger`. |
| `rule_target_value_bool_N` | Text (combo) | `false` | Value used when Property is `Boolean`. |
| `rule_target_value_num_N` | Text | `0` | Value used when Property is `Value` or `Position`. |
| `rule_target_value_str_N` | Text | *(blank)* | Value used when Property is `String`. |
| `rule_status_N` | Status indicator | 4 (Missing) | 0 = OK, 2 = Fault, 4 = Disabled. Status text in `.String`. |
| `rule_clear_N` | Trigger button | — | Resets rule N. |

All three value controls exist on every row; only the one matching the Property is visible. Setting a control from a script runs the same handlers as the UI, so a scripted change to `rule_trigger_N` re-wires triggers immediately.

---

## How it works (for maintainers)

### Runtime flow

```
Start-up
  EventHandlers()            attach handlers to every plugin control
  Initialize()
    RefreshSVComponentChoices()   list Seervision instances by Type
    BindSeervision()              Component.New on the selected one
    RefreshTriggerChoices()       list "*Triggered*" controls → RewireTriggers()
    RefreshNamedComponents()      list every named component
    per rule: OnTargetComponentChanged() → OnTargetControlChanged()
              → UpdateValueControlVisibility(); RefreshRuleStatus()
  RefreshTimer:Start(10)     repeat the discovery steps every 10 s

Trigger fires (Seervision control goes true)
  dispatcher → for each rule index in TriggerMap[name]: ApplyRule(i)
```

### Design points

- **One dispatcher per trigger control.** `RewireTriggers()` builds `TriggerMap` (trigger name → list of rule indices) from the rule rows. It attaches a single EventHandler to each referenced Seervision control. When the handler fires, it reads `TriggerMap` and runs `ApplyRule(i)` for every listed rule.
- **No push/pop bookkeeping.** `TriggerMap` is rebuilt from the UI every time a trigger selection changes, so adding or removing a reference can't leave a stale action behind. Controls that drop out of the map are unhooked (`EventHandler = nil`). `HookedTriggers` records which Seervision controls currently carry the dispatcher.
- **Dispatcher reads the map at fire time.** Changing which rules share a trigger doesn't need re-hooking; the existing dispatcher picks up the new list on its next fire.
- **Single Seervision binding.** `BindSeervision()` only creates a new `Component.New` proxy when the selected component changes. Switching components unhooks everything on the old one first.
- **Arming is separate from firing.** `rule_enable_N` only calls `RefreshRuleStatus()`. `ApplyRule()` is only ever called by a trigger dispatcher, and checks the enable state itself.
- **Targets are resolved on every fire.** `ApplyRule()` calls `Component.New(target)[control]` each time, so a target that is renamed back, or gains Script Access, starts working without a restart.
- **Failures stay local.** Target lookups and value writes run inside `pcall`. A missing target only faults its own rule's LED.
- **Selections survive reloads.** `PreserveOrDefault()` keeps a dropdown's current value whenever its choices are rebuilt, as long as that value is still in the list.

### Key functions

| Function | Role |
|---|---|
| `RefreshSVComponentChoices()` | Fills `sv_component` with components whose Type is `SV_PLUGIN_TYPE`. |
| `BindSeervision()` | Creates the Seervision proxy for the selected component, if not already bound. |
| `RefreshTriggerChoices()` | Fills every Trigger dropdown from the bound Seervision controls, then calls `RewireTriggers()`. |
| `RefreshNamedComponents()` | Fills every Target Component dropdown and the Inspector list. |
| `OnTargetComponentChanged(i)` | Lists the writable controls of rule i's target component. |
| `OnTargetControlChanged(i)` | Sets rule i's Property choices from the target control's Type. |
| `UpdateValueControlVisibility(i)` | Shows the value field matching rule i's Property. |
| `BuildTriggerMap()` / `RewireTriggers()` | Rebuild the trigger → rules map and hook/unhook Seervision controls to match. |
| `ApplyRule(i)` | Writes rule i's value to its target and sets its status LED. |
| `ClearRule(i)` | Resets rule i and re-wires triggers. |
| `DumpInspector()` | Writes the Inspector listing. |

---

## Customising the plugin

All tunables are near the top of the runtime block in `vsmanager control.qplug`. Any change to the file needs a version bump and changelog entry (see [Versioning](#versioning)).

| Constant / location | Purpose | When to change it |
|---|---|---|
| `SV_PLUGIN_TYPE` | Type string used to find Seervision instances. | A Seervision plugin update changes its Type. Read the new value from `Component.GetComponents()`. |
| `TRIGGER_CONTROL_PATTERN` | Substring a Seervision control name must contain to be offered as a trigger. It's used with `string.find`, so Lua pattern characters (`-`, `.`, `%` and so on) need escaping. | Seervision renames its zone controls, or you want to offer other controls as triggers. |
| `PROPERTY_CHOICES_BY_TYPE` | Control Type → Property options. | Supporting another control type. |
| `DEFAULT_PROPERTY_CHOICES` | Fallback when the target's Type is unknown. | Rarely. |
| `RefreshTimer:Start(10)` | Re-scan interval in seconds. | Very large designs where a 10 s scan is too costly, or where faster pick-up is needed. |
| `Rule Count` `Max` in `GetProperties()` | Maximum number of rule rows. | You need more than 32 rows. |

To add a new Property type, add it to `PROPERTY_CHOICES_BY_TYPE`, give it a branch in `ApplyRule()`, and decide which value field it uses in `UpdateValueControlVisibility()`.

### Testing changes

1. Copy the updated `.qplug` into the plugins folder and restart Designer.
2. Use a test design with the Seervision plugin (or any component whose controls contain `Triggered`; temporarily changing `SV_PLUGIN_TYPE` lets you test without Seervision hardware) and a few targets with Script Access.
3. Run in Emulate and check at least: single rule fires; two rules stacked on one trigger both fire in order; setting a row to `none` or pressing Clear stops it firing while the other rule still does; a renamed target faults only its own LED; disabling a rule shows `Disabled`; switching Seervision component unhooks the old triggers.

---

## Known limitations

- **One Seervision instance per mapper.** To map triggers from several Seervision plugins, add one mapper per instance.
- **Rising edge only.** Rules run when a trigger reports true. There's no "on false" action or delay/hold.
- **Target Control lists aren't refreshed by the timer.** They're rebuilt when the row's Target Component changes or the plugin restarts. If a target gains controls while running, re-select its component.
- **Faults surface on fire, not on arm.** A misconfigured rule shows green until its trigger fires.
- **Seervision is found by exact Type string.** A Seervision plugin update that changes its Type hides it from the dropdown until `SV_PLUGIN_TYPE` is updated.
- **Changing Rule Count rebuilds the controls.** Check your rules after changing it, especially when reducing the count.
- **Development build.** Versions below 1.0.0 aren't signed off for production.

## Troubleshooting

| Symptom | Check |
|---|---|
| Dropdowns are empty | The design is running (Core or Emulate). The plugin only populates at runtime. |
| Seervision component not in the dropdown | Script Access is enabled on it. If it still doesn't appear, its Type string may have changed: open the **Inspector**, or run `Component.GetComponents()` in a script, and update `SV_PLUGIN_TYPE` in the plugin. |
| Trigger dropdown only shows `none` | Seervision component is bound and running. Use the Inspector to confirm its zone controls contain `Triggered`. If they don't, update `TRIGGER_CONTROL_PATTERN`. |
| Target component missing | Script Access on the target. Wait up to 10 s for the next refresh. |
| Target control missing | The control is read-only (these are filtered out), or the component's controls changed after it was selected: re-select the component. |
| Rule shows OK but nothing happens | Rule **On** is enabled, and the trigger is actually going true (watch it on the Seervision plugin; the Inspector lists controls but not their live values). |
| `Target component/control not found` | The target was renamed, deleted, or lost Script Access. |
| `Apply failed: …` | The value doesn't suit the control (e.g. out of range, wrong property). Check the control's Type in the Inspector. |
| Action fires twice | Two rules in the stack point at the same target, or another script is also reacting to the same trigger. |

## FAQ

**Does turning a rule On fire it?**
No. On only arms the rule. It fires the next time its trigger goes true.

**What order do stacked rules run in?**
Rule-number order, lowest first.

**Can a rule's target be another mapper or the Seervision plugin itself?**
Yes, any named component with Script Access and a writable control can be a target.

**Will my rules survive a reload or redeploy?**
Yes. Selections are stored in the plugin's controls and restored when the design starts, as long as the trigger and target names still exist.

**Can I configure rules from a UCI?**
Yes. All rule controls can be placed on a UCI, and the dropdowns behave the same there.

---

## Versioning

- Development builds use `0.0.0.N`. Every change bumps `N` (`BuildVersion` at the top of the `.qplug`) and gets a changelog entry below.
- Version **1.0.0** is reserved for the release signed off as production-ready.

## Changelog

### 0.0.0.14
- **Trigger stacking:** multiple rules can share one Seervision trigger, and all of them fire in rule order. Previously only the last-wired rule fired.
- Removing or changing a rule's trigger now removes its action. Previously the old trigger kept firing the rule. Unreferenced trigger controls are unhooked.
- Added `none` as a permanent choice in the Trigger dropdown so rows can be unhooked.
- Added a per-rule **Clear** button. The Rules page widened to 750 px.
- The Seervision component is no longer re-bound with a fresh `Component.New` proxy every 10 s, only when the selection changes.
- The target control lookup in `ApplyRule` is now fully inside `pcall`.
- Removed the redundant timer re-start; `Timer:Start()` already repeats.
- Saved trigger selections are no longer reset to `none` while the Seervision component is unavailable.
- Header graphics use `HTextAlign` (was `HTextAlignment`). Stale comments fixed.

### 0.0.0.13
- Baseline: rule rows, Seervision type filter, `Triggered` control discovery, Inspector page.
