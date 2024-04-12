## Simple Example

These are the steps I took to convert a very simple schematic into a working MAME netlist.

As an example I used

https://github.com/mamedev/discrete/tree/master/kicad/wildfire

Load the project in KiCAD and open the schematic editor.

Some of these files have been created in older version of KiCAD, in this case it will ask you to rescue the symbols, just click OK.

Once you can see the schematic, click on `File -> Export -> Netlist...`

In the Export Netlist window click on the `Spice` tab and then on `Export Netlist`.
Save the file next to the other project files.

If you had errors in the annotation, it will pop up a window at this point, in the case of this example it's fine to click on `Annotate` and continue.

Your wildfire.cir file will look somewhat like this:

```
.title KiCad schematic
C1 Net-_I_A12-P1_ VDD 4.7u
I_A12 __I_A12
O_AUDIO1 __O_AUDIO1
R2 Net-_I_A12-P1_ Net-_Q2-B_ 10k
I_F1 __I_F1
Q3 __Q3
Q2 __Q2
R1 Net-_Q2-B_ VDD 10k
.end
```

Next we'll need a build of nltool.
Open a terminal and navigate to your mame source folder and then into src/lib/netlist/build
If you don't have nltool yet, run `make` (It doesn't build automatically with MAME).

If the build succeeded run `./nltool --help` to get a short overview of the features.
Next we want to convert the Spice netlist into a MAME one.

For this the command is `./nltool -c convert /path/to/discrete/kicad/wildfire/wildfire.cir`

Here we hit our first roadblock, the converter thinks the part labeled I_A12 is a power source.
Go back into the schematic editor, click the in- and output terminals and add a 'J' at the start of the reference.
You can find a list of which reference letters will convert to which parts in nl_convert.cpp

After changing them export the netlist to retry converting it.

This time, nltool will throw a segmentation fault.
The problem is that it expects the transistors to have more than 1 parameter each.

Click the transistor in the schematic, press E (or right click and choose Edit) and in the window that pops up
at the bottom there's a button `Simulation Model...`
In the Simulation Model Editor change the radio box to `Built-in SPICE model` and for Device pick "PNP BJT", because
the transistor in this schematic is a PNP one.
Exit out with OK and pay attention to the Sim.Pins field in the Symbol Properties.
Most likely the Pin assignment is wrong here, this transistor uses ECB as the pin order but the default spice symbol assigned CBE.
Change it to `1=E 2=C 3=B` and exit with OK.

Export the netlist again and run nltool with the convert command again and it will work!

To save this netlist run the command again but add ` > nl_wildfire.cpp` to pipe the output into that file.

You can now try to run this file with `./nltool -c run nl_wildfire.cpp`
It will fail and complain that the Base of Q3 isn't connected to anything else.

The base of Q3 is actually one of the inputs of this circuit so we'll do that next.
Open nl_wildfire.cpp in a text editor.

Remove the line it complained about `NET_C(Q3.B)`

Between the last NET_C line and the } bracket add a few newlines.

Here we'll add some lines for our in- and outputs
```
ALIAS(I_F, Q3.B)
ALIAS(I_A12, R2.1)
ALIAS(O_AUDIO, Q3.E)
```

If we run the netlist now it complains about CAP_U missing.
This is an easy fix, add `#include "netlist/devices/net_lib.h"` as the first line before `NETLIST_START(dummy)` and on that occasion rename dummy to wildfire.

Next error is that xxPNP is missing. MAMEs PNP transistor has a different name from KiCADs
Remove the 2 NET_MODEL lines and at the 2 lines for `QBJT_EB(Qx, "__Qx")`, replace the `__Qx` with `PNP`.

If you run it again it will not complain about our netlist anymore but that the solver is missing.
Create a newline after `// .END` and add this line.
```
SOLVER(Solver, 48000)
```

48000 is the default value for audio circuits.
If you're recreating a different circuit you'll want to change this later.

After this change the netlist will finally run in nltool
We can check on the output pin with the following command:
`./nltool -c run -t 3 -l O_AUDIO ./nl_wildfire.cpp`

If you open `log_O_AUDIO.log` you'll see the time on the left side and the voltage on the right side.
The voltage is slowly rising and the circuit isn't working quite right.
This probably has to do with the fact we're missing input power.

After the SOLVER line add
```
ANALOG_INPUT(V5, 5)
ALIAS(VDD, V5)
```

Go to line `NET_C(C1.2, R1.2, Q2.E)` and add `VDD, ` before C1 to connect our new power source up.
If we run the log again it's now at a steady 5V.
This is expected as the circuit isn't getting any input right now.

## Notes
This will not work for multi-part ICs (such as almost all 74 series parts)
The reason for this is that KiCAD ignores all parts after the first one.
The bug is being tracked here:
https://gitlab.com/kicad/code/kicad/-/issues/1779