FreeCAD has a powerful [Expression Engine](https://www.freecadweb.org/wiki/Expressions).
This tutorial shows how to create simple mechanism using constraints driven by
some expressions.

# Screw Mechanism

It is very easy to create a screw mechanism with `PlaneConicident` constraint.
Just follow the simple steps below

* Set the constraint property `LockAngle` to `True`;
* Enter an expression;
  * If you want to drive by rotating the screw, then express the `Offset`
    property in term of the `Angle` property, as shown in the screen cast
    below;
  * If you want to drive by translation, then express `Angle` in term of
    `Offset`.

[[images/screw.gif]]

# Gear Mechanism

A slightly more complex example is a [rack and pinion](https://en.wikipedia.org/wiki/Rack_and_pinion)
system shown below. All gears have module of 1mm. The big pinion gear has 18
teeth, while the small one has 15. You can download the parts 
[here](https://github.com/realthunder/files/raw/master/misc/gears.fcstd).
You need to install the [Gear Workbench](https://github.com/looooo/FCGear) if
you want to modify the gears.

[[images/gear.png]]


By the look of the picture above, one may intuitively think of using the
`PointOnLine` constraint for the gear in the slot. But in this case, it is much
easier to use, again, the all powerful `PlaneCoincident` constraint. We divide
this system into two assemblies for, well, the pinions and the rack. 

First, create a sketch to define the positions of the two pinions. In real
applications, you probably should derive the distance of the two circle using
expressions, too, as well as most of other constants shown in this tutorial, or
better, use a [Spreadsheet](https://www.freecadweb.org/wiki/Spreadsheet_Workbench).

Then, create an assembly, add the sketch and two gears, and position the two
using the sketch with `PlaneCoincident` constraint.

Toggle the `LockAngle` property of both constraints. And since we will be
driving the big gear, enter the expression for the `Angle` property of the
small gear constraint as below, which is basically the gear ration with some
offset,

```
-Constraint002.Angle*18/15-6
```

You can test the mechanism by changing the `Angle` of the big gear constraint.

[[images/gear1.gif]]

Now, we create the final assembly, add the pinion assembly and the base with
the rack gear, and fix them together with another `PlaneCoincident`. Notice
that we are using the `Sketch` for constraining. This is important, because
the gear is meant to be rotated, while the `Sketch` is fixed. We shall again
lock the rotation of this constraint. We want to translate the pinions in its
relative Y position, so enter an expression in the `OffsetY` property of the
constraint,

```
-Constraint001.Angle/360*15*pi
```

Done!

[[images/gear2.gif]]

