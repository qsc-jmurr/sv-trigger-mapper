# Seervision Trigger Mapper

The Seervision Trigger Mapper connects zone triggers from a Seervision / VisionSuite plugin to writable controls on other components in a Q-SYS design. Configure the mappings in Q-SYS Designer—no Control Script or control-pin wiring is needed.

## Prepare your design

1. Add the Seervision plugin and the components you want to control to the design.
2. Give the Seervision component and target components clear, unique names. These names appear in the mapper's dropdowns.
3. Select the Seervision component and set **Script Access** to **Script** or **All** in its Properties.

   ![Enable Script Access on the Seervision component](<MD Images/Seervision Script Access.png>)

4. Enable **Script Access** on each target component as well. The example below shows a target component in the schematic.

   ![Example target component in the schematic](<MD Images/Component Script Access.png>)

5. In the mapper's Properties, set **Rule Count** to the number of mappings you need (1–32; default is 4). Changing this later rebuilds the rule controls, so check your configuration afterward.
6. Save and run the design on a Core, or start **Emulate**.

## Configure trigger mappings

Open the mapper and use the **Rules** page. Start by selecting the Seervision component, then configure a row for each action:

1. **Trigger** — Select a Seervision control for the zone event. Zone trigger names typically contain `Triggered`.
2. **Target Component** — Select the component whose control should change.
3. **Target Control** — Select a writable control on that component.
4. **Property** — Select how the target is to be set. Available options depend on the target control's type.
5. **Value** — Enter or select the value to apply. This field changes to match the selected property.
6. **On** — Enable the rule. Enabling a rule arms it; it does not fire the action immediately.

![Rules page with example trigger mappings](<MD Images/Configuration.png>)

When the selected trigger goes true, the mapper applies the configured value. The row's status indicator is green when enabled and working; if an action fails, check the status text and the target's Script Access and control settings.

### Properties and values

| Property | Use with | Value applied when the trigger goes true |
|---|---|---|
| **Boolean** | Boolean controls such as mute or bypass | Select `true` or `false`. |
| **Value** | Float or Integer controls | Enter a value in the control's units, such as dB for a gain. |
| **Position** | Float or Integer controls | Enter a normalized position from `0` to `1` across the control's range. |
| **String** | Text controls | Enter the text to set. |
| **Trigger** | Trigger controls such as snapshot loads or preset recalls | No value is needed; the control is triggered. |

### Multiple actions from one zone

You can select the same trigger in multiple rows. When it goes true, all enabled rows run in rule-number order. For example, one zone can unmute a lectern microphone and mute audience microphones.

Rules run only when a trigger goes true; the mapper does not apply an action when the trigger goes false. To remove an action, select `none` as its trigger or use that row's **Clear** button. Clearing a row does not affect other rules that use the same trigger.

## Find component and control names

Use the mapper's **Inspector** page to select a component and see its control names, types, and directions. This is useful for confirming a Seervision trigger name or finding a target control. Controls marked **Read** are not writable targets.

![Inspector page showing a component's controls](<MD Images/Inspector.png>)

## Test the configuration

1. Confirm the mapper lists the intended Seervision component and that the trigger dropdown contains the expected zone trigger.
2. Confirm the target component and writable target control are selected.
3. Turn the rule **On** and verify its status indicator is green.
4. Make the zone trigger go true in your running design and check that the target changes as expected.

## Notes

- Each mapper instance selects one Seervision component. Add another mapper if you need to map triggers from another instance.
- The mapper refreshes component and trigger lists every 10 seconds. If a target component gains controls while running, select it again to refresh its target-control list.
