# Sovol Eddy on a mainlined SV08 with native Klipper tap and scan correction

## Prereq

- SV08 mainlined with the Sovol Eddy kit installed as outlined in their docs, but do not flash their software as that would ruin mainline
- Ensure you are on the latest version of Klipper
- Remove any previous Eddy config (as great as Eddy-NG is, you will need to uninstall it for this to work, if doing so, don't forget to reflash all MCUs)
- Ensure the full section `[probe]` is commented out in your printer.cfg (if it was not, run `SAVE_CONFIG ` afer you do it)
- Tap requires scipy, so install it: `~/klippy-env/bin/pip install scipy`

---

## 1. Edit `ldc1612.py`

First step is editing `ldc1612.py` to edit an else statement that helps Klipper use the right values for the frequency of the coil. In future updates of Klipper this will be done automatically.

For now:

```bash
cd klipper/klippy/extras/
edit ldc1612.py with your fav editor (vi/vim/nano etc)
go to line 94 (or if that changes, the line with: self.sensor_div = 1 if self.clock_freq != DEFAULT_LDC1612_FREQ else 2)
change else 2 to else 4
save the file
sudo systemctl restart klipper.service
```

If this is not done, the calibration step will not work correctly.

**NOTE:** If you automatically update Klipper and this file changes, you will need to redo this step until it is added in via an official commit eventually.

---

## 2. Update config

Copy my sovol-eddy-cfg from `config/options/probe/sovol-eddy.cfg` here and reference it in your printer.cfg, however, it's a good idea to change the following for now (we will update/change it later):

### In the `[probe_eddy_current my_eddy_probe]` section, comment out the following lines:

```text
tap_threshold: 140
samples: 5
samples_result: average #median
samples_tolerance: 0.025
samples_tolerance_retries: 3
```

### In the `[homing_override]` section

Change the line:

```
SET_Z_FROM_PROBE METHOD=tap
```

to:

```
SET_Z_FROM_PROBE
```

---

## 3. Home and set kinematic position

Home the printer, since we have not run the next calibration steps, it should not home Z and give you an error (but it homes X and Y ok):

```
Must home axis first: 172.000 172.000 9.000 [0.000]
```

We can overcome this my tricking the printer into thinking Z is known:

```
SET_KINEMATIC_POSITION X=177.50 Y=177.5 Z=10
```

The value of Z does not matter here, since we're doing the "paper test" next and Z will be overwritten. If you used my homing_override the values of X and Y should match.

---

## 4. Run calibration

Run:

```
PROBE_EDDY_CURRENT_CALIBRATE CHIP=my_eddy_probe
```

You should see something similar (but not the same):

```
10:35 p.m.
The SAVE_CONFIG command will update the printer config file
and restart the printer.
10:35 p.m.
z_offset: 3.010 # noise 0.001115mm, MAD_Hz=13.904
10:35 p.m.
z_offset: 2.010 # noise 0.001014mm, MAD_Hz=19.484
10:35 p.m.
z_offset: 1.010 # noise 0.000772mm, MAD_Hz=23.503
10:35 p.m.
z_offset: 0.530 # noise 0.000158mm, MAD_Hz=6.995
10:35 p.m.
z_offset: 0.290 # noise 0.001456mm, MAD_Hz=69.981
10:35 p.m.
Total frequency range: 87791.762 Hz
10:35 p.m.
probe_eddy_current: noise 0.001387mm, MAD_Hz=28.688 in 2525 queries
```

Issue:

```
SAVE_CONFIG
```

---

## 5. Tap calibration

Now, for your own tap calibration you can follow https://www.klipper3d.org/Eddy_Probe.html#tap-calibration , or, here is a condensed version using an estimation:

In Step 4's output rules of the calibration, you'll see lines like:

```
z_offset: 0.290 # noise 0.001456mm, MAD_Hz=69.981
```

What you do is take one of the higher (or higest values) to start, round it to nearest number, and multiply by 2 to give you your `tap_threshold` value which is updated in your `[probe_eddy_current my_eddy_probe]` section of config.

Example using my results:

```
z_offset: 0.290 # noise 0.001456mm, MAD_Hz=69.981
```

So `69.981` rounded up is `70` multiplied by 2 is `140`.

Do the same for your results, and put that value into the `[probe_eddy_current my_eddy_probe]` section of your config, uncommeting everything again:

```text
tap_threshold: 140
samples: 5
samples_result: average #median
samples_tolerance: 0.025
samples_tolerance_retries: 3
```

You can also change the `[homing_override]` section back: change the line:

```
SET_Z_FROM_PROBE
```

to:

```
SET_Z_FROM_PROBE METHOD=tap
```

Run:

```
SAVE_CONFIG
```

after your changes.

---

## 6. Test

Test it all out, if you run:

```
PROBE METHOD=tap
```

nothing should blow up, and as the official guide suggests, running:

```
PROBE_ACCURACY METHOD=tap
```

should give you a `range:` output of around ~0.02mm

If not, update your `tap_threshold` value using the tests outlined in the guide. I personally just took the value I showed you in my example and it was great out of the box.

## 7. How to use it

The included macros in my `config/options/probe/sovol-eddy.cfg` can be used like this, for example (see my `macros.cfg` file for how I use them):

### START_PRINT
1. Preheat bed
2. QGL, re-home Z
3. Heat hotend to 150C, clean nozzle
4. Go somewhere (center or SMART_PARK from KAMP)
5. Tap
6. Bed mesh
7. Start print!

**Notes / updates on your QGL macro:**

I learned from eddy-ng it is a good idea to run two QGLs when you have a nozzle tap. One a bit higher in case of a racked gantry, then next one lower.

So perhaps you'll want to update your QGL to use do something like:
1. First pass with `horizontal_move_z=5 retry_tolerance=1`
2. Second pass with `horizontal_move_z=2`

To use step 7.5's tap, use the macro `SET_Z_FROM_PROBE METHOD=tap`
   
To use step 7.6's bed mesh macro use:
```
SET_SCAN_FROM_TAP
# For best results should match the previous command "scan" height
{% set scan_height = printer.configfile.settings["probe_eddy_current my_eddy_probe"].z_offset  %}
BED_MESH_CALIBRATE ADAPTIVE=1 METHOD=scan HORIZONTAL_MOVE_Z={scan_height}
```

**NOTE:** You can change `METHOD=scan` to `METHOD=rapid_scan` if you want

This bed mesh macro creates another offset from the scan that is unique from your Z offset tap.

## 8. First layer corrections

If you notice your first layer needs to be a bit higher or lower, edit this value in this line of the `[gcode_macro _RELOAD_Z_OFFSET_FROM_PROBE]`

`    {% set tap_correction = -0.05 %} # manual, can adjust up or down as needed`

Negative values bring nozzle down, postive values bring the nozzle up

## 9. Credit and sources

Thanks especially to Timofey Titavets (https://klipper.discourse.group/u/nefelim4ag/summary) for their help setting this up and for implementing this code into Klipper!

All Z offset and bed mesh calibration macros credited to Timofey as well

Relevant links:
**Z offset macros:** https://www.klipper3d.org/Eddy_Probe.html#homing-correction-macros

**Bed mesh calibration macros:** https://github.com/Klipper3d/klipper/pull/7179#issuecomment-3832267138

## 10. Questions or comments?

Happy to help here for any questions related directly to my config, but probably better for everyone if the greater discussion for mainline Klipper using Sovol's Eddy probe is had on:

https://klipper.discourse.group/t/sovols-eddy-hardware-info/25610
