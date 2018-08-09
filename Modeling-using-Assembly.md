It is some time desirable to model parts while building the assembly. This
tutorial shows two approaches of doing this.

[[images/assembly-binder.png]]

We are going to assembly the above parts. Imaging the blueish part is a PCB,
the brown box is a USB socket, and the green board is a panel. In addition to
assembling these three parts together, we also want to cut a hole out of the
panel to expose the USB socket, at the assembled location.

# Using `Part` Workbench

The first approach uses the `Cut` feature in `Part` workbench. Both approaches
will use the new `SubShapeBinder` to link the child shape of some group with
synchronized placement update. Although `SubShapeBinder` is exposed in
`PartDesign` workbench, it can be used without a `Body`.

We first create a sketch at the opening facing of the socket. In real world
usage, you will certainly use some external geometry in sketch to draw the hole
shape, which is omitted here for simplicity. Once finished, go to `Part`
workbench and extrude the sketch. Then create an assembly to hold the socket and
the hole shape. Select the hole shape tree item in the assembly, go to
`PartDesign` workbench, and click
![SubShapeBinder](../../FreeCAD/raw/LinkStage3/src/Mod/PartDesign/Gui/Resources/icons/PartDesign_SubShapeBinder.svg?sanitize=true)
to create a binder to the shape. This binder will automatically update the shape
placement whenever either the hole or its parent assembly changes placement.

[[images/part-binder.gif]]

Next, create another assembly. Here comes the most critical step. You need to
add the original uncut panel shape to the assembly. When you drop to the
assembly you'll probably getting error message like "Box cannot be dragged out
of Cut", you can correct this by holding `CTRL` key before dropping. After that,
proceed to add other parts.

Finally, it is time to assemble the parts. First, lock the PCB part. Add
`PlaneAlignment` constraints to fix the socket to PCB. Then, another important
step, fix the original uncut panel shape to PCB, **instead of** the cut shape.
The cut shape does not need to be constrained, it will follow the original
panel shape. You can hide the uncut panel and reveal the panel with the hole in
the final assembly.

# Using `PartSesign` Workbench

The second approach is to use the `PartDesign` workbench, which seems to be the
natural choice when the panel itself is created using `PartDesign Body`. The
first step of creating the hole shape is exactly the same as the previous
approach. Because there exists a `PartDesign` body, when you create the
`SubShapeBinder`, it will be added to the body automatically. Hide the hole
shape, select a face of the binder shape, and create a `Pocket` to punch the
hole.

Once all parts are ready, create an assembly, drop in all three parts, and
assemble them as before. Notice that when the panel body changes position, the
hole is not updated accordingly. You need to manually update the binder by
double clicking the item in the tree view, and then recompute the document to
update the body. This limitation is due to the fact that the binder needs the
parent placement information to calculate the relative location of the binding
shape, but FreeCAD always recomputes child before the parent.

[[images/partdesign-binder.gif]]

