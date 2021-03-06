# `hal_fixed_atc_touchoff` for LinuxCNC 

HAL component containing ATC (Automatic Tool Changer) and tool touchoff routines for LinuxCNC. Assumes a fixed tool rack and touchoff within the machine boundary.

Written in Python 2 (LinuxCNC does not yet support Python 3; see LinuxCNC/linuxcnc#403). Uses `transitions` for finite state machine and `PySimpleGUI` for the GUI.

Made for the Gerber CNC wood router at Makehaven (makerspace in New Haven, CT).

## Credits

- Andrew Suzuki ([andrewsuzuki.com](https://andrewsuzuki.com))

## Pins

- `hal_fixed_atc_touchoff.number` IN FLOAT
- `hal_fixed_atc_touchoff.tool_prepare` IN BIT (internally looped to `prepared`)
- `hal_fixed_atc_touchoff.tool_change` IN BIT
- `hal_fixed_atc_touchoff.touchoff` IN BIT
- `hal_fixed_atc_touchoff.prepared` OUT BIT (internally looped to `tool_prepare`)
- `hal_fixed_atc_touchoff.changed` OUT BIT
- `hal_fixed_atc_touchoff.chuck` OUT BIT

## Loading into LinuxCNC

```
loadusr -W hal_fixed_atc_touchoff
net tool-change-request     iocontrol.0.tool-change       =>  hal_fixed_atc_touchoff.change
net tool-change-confirmed   iocontrol.0.tool-changed      <=  hal_fixed_atc_touchoff.changed
net tool-number             iocontrol.0.tool-prep-number  =>  hal_fixed_atc_touchoff.number
# net tool-prepare-loopback   iocontrol.0.tool-prepare      =>  iocontrol.0.tool-prepared


```

## Notes

- All moves must avoid the tool rack. Strategy?
- Tool loading and return can be dangerous if there is already a tool in the
  pocket.
- Whenever the machine will be picking up a new tool (atc tool retrieval or
  operator tool unloading), the chuck should briefly open to drop any tool
  already in the chuck (*flush*).

## TODO

- Wait for spindle off
- Tweak feeds
- Cancel current program (using command api?) if toolchange fails (for example,
  tool doesn't exist in table)
- Update procedure notes below

## Configuration

See `config.json` as an example.

- `touchoff_length_compensation_offset`: When setting tool length compensation
  (G43), this is the amount to subtract from the MACHINE coordinate when the
  touchoff hits to calculate tool length compensation. Accepts positive/negative
  numbers.
- `touchoff_retract_offset_z`: Relative z offset after touchoff touchoff.
- `pocket_side_offset`: Relative x/y offset (map) for pocket sides (where it
  will slow down for insertion/removal), calculated from individual pockets.
- `pocket_above_close_offset_z`: Relative z offset from individual pockets
  where the chuck will open/close (depending on if it's retrieving/returning).
- `pocket_above_clearance_offset_z`: Relative z offset from individual pockets
  where the machine is free to move in the xy plane (towards/away from the tool
  rack). Must account for height of the tool holder and pull stud above the
  pocket.
- `loiter_seconds`: Time in seconds where the chuck will remain open at the
  pocket after reaching it.
- `flush_seconds`: Time to open the chuck as a safety procedure before inserting
  a tool into the rack
- `loading`: absolute x/y/z coordinate (map) of the loading position. x/y SHOULD
  be equal to the x/y of `touchoff_limit` (below)
- `touchoff_limit`: absolute x/y/z coordinate (map) of the touchoff position (the
  farthest down it should go). x/y SHOULD be equal to the x/y of `loading`
  (above)
- `safe`: absolute x/y/z coordinate (map) of an arbitrary position near the tool
  rack (should be "in front" of it)
- `pocket_default`: absolute x/y/z coordinate (map) of the default pocket. This
  will be merged into the actual pocket coordinates (below), allowing, say a
  default Y/Z but variable X.
- `pockets`: map of tool/pocket number strings to absolute [x]/[y]/[z]
  coordinates (maps). Will use `pocket_default` as default x/y/z.

## Procedure

The operator will repeat this process for every tool in the upcoming program.

1. Operator enters tool # in GUI and clicks "Load / touchoff Tool".
2. The machine moves to `loading` (first z, then xy).
3. GUI displays temporary window with "Please load tool # now". The window also
   has a "Continue" and "Cancel" button.
4. Operator loads a tool (either by pressing button on solenoid manually, or
   through GUI, though manually is probably more convenient).
5. Operator clicks "Continue"
6. Optionally, operator jogs the machine down closer to the touchoff.
7. GUI displays temporary window with "Please attach the touchoff wire to the
   tool", and "Begin touchoff" button.
8. Operator attaches touchoff wire
9. Operator clicks "Begin touchoff"
10. The machine begins a loop where it moves down in small increments, checking
    each time if it has contacted the touchoff. If it has, it calculates the tool
    offset from the current machine z position.
11. The machine retracts (a relative amount upwards).
12. The machine places the tool back in the rack (see M6 Tool Change)

### Operator Unloads Tool

1. Operator enters tool # in GUI and clicks "Unload tool".
2. The machine moves to `loading` (first z, then xy).
3. GUI displays temporary window with "Please unload tool # now". The window also
   has a "Done" button.
4. Operator unloads the tool (either by pressing button on solenoid manually, or
   through GUI, though manually is probably more convenient).
5. Operator clicks "Done"

### M6 Tool Change

If there's a tool in the spindle, return it:

1. Machine moves to `safe` (first z, then xy)
2. Machine moves to target pocket *side*.
3. Machine moves to target pocket (slow).
4. Chuck opens and stays at target pocket for some amount of time (~0.5s).
5. Machine moves a bit above the pocket (chuck still open).
6. Machine closes chuck.
7. Machine moves further upwards to clearance position.
8. Machine moves to `safe`.
9. Continues below...

If there isn't a tool in the spindle:

1. Machine moves to `safe` (first z, then xy)
2. Machine moves to target pocket *above-clearance*
3. Machine moves to target pocket *above-chuck*.
4. Chuck opens.
5. Machine moves to pocket (slow).
6. Machine stays at target pocket for some amount of time (~0.5s).
7. Chuck closes.
8. Machine moves to target pocket *side* (slow).
9. Machine moves to `safe`.

### Load/Unload common setup logic

Decisions on begin load/unload:
- if spindle unloaded
  1. [attempt to] retrieve target tool
  2. go to loading, continue...
- if spindle loaded, it's the target tool
  1. go to loading, continue...
- if spindle loaded, it's not the target tool
  1. return current tool
  2. [attempt to] retrieve target tool
  3. go to loading, continue...
