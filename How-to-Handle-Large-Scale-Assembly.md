This tutorial shows some of the features offered by `Assembly3` to work with
complex assemblies. The sample assembly used in this tutorial is not that
complex by itself, but by following the steps illustrated here, smaller
assembly can be easily scaled up to build much larger and complex ones.

The model was taken from FreeCAD forum member `jpg87` in 
[this](https://forum.freecadweb.org/viewtopic.php?f=20&t=25712&start=590#p241223)
post. It is the rotor head of a wind turbine as shown below,

[[images/wind-turbine.png]]
[[images/wind-turbine-detail.png]]

The finished rotor assembly looks like this,

[[images/wind-turbine-rotor.png]]

The assembly consists of the following parts.

[[images/wind-turbine-gear.png]]
[[images/wind-turbine-ring.png]]
[[images/wind-turbine-dome.png]]
[[images/wind-turbine-socket.png]]
[[images/wind-turbine-pin.png]]

The original parts file can be downloaded 
[here](https://github.com/realthunder/files/raw/master/misc/wind-turbine-parts.zip)

Now, let's begin.

# Wrap Each Part With an Assembly

Although optional, it is highly recommended to wrap each part object using an
`Assembly` container. In `Assembly3`, the assembly container serves a double
purpose of both constructing a _Product_, and representing a _Part_. For the
later, the container often does not contain any constraint. It is used to
manage the geometry elements in this _Part_ for use by other _Products_. No
matter how many hierarchies the _Product_ has, any constraining elements that
belong to this _Part_ will be referencing the `Element` object in the
`ElementGroup` of the assembly container. As shown in the following screen
cast, it is better to plan ahead which elements of the part will be used for
constraining, and create them explicitly, with proper names. You don't have to
do these at this stage. You can create new elements of any assembly at any
hierarchy with simple drag and drop. But it is a good habit to do it manually
and explicitly. 

[[images/wind-turbine1.gif]]

Also, notice that we are creating element from Sketch element as much as
possible. And when that is not possible, use feature early in the modeling
history, or datum feature, or even the origin plane if possible, in order to
avoid lost of element reference due to model changes.

# Assemble the Axis

Next, we are going to assemble the axis part, which consists of two pins and
a dual holes socket type of thing. Create a new file, and remember to **SAVE**
the file first in order for external linking to work. Create a new assembly,
and then simply drag the parts into it. This automatically creates a new `Link`
object to externally link to the dropped object. Because of some technical
reason, right now FreeCAD does not allow you to drag more than one objects from
different documents (you can if all dragging objects are from the same
document).

The axis needs two pins. You can either drag and drop the pin object twice to
create two separate links, or drop it once, and change the `ElementCount`
property of the link to make it an array. Before creating any constraints,
remember to choose a fixed base part first, by creating a `Lock` constraint
using one of its element. This simple assembly can be formed by two
`PlaneCoincient` constraints,
![PlaneCoincident](../raw/master/Gui/Resources/icons/constraints/Assembly_ConstraintCoincidence.svg?sanitize=true).

[[images/wind-turbine2.gif]]

As shown in the above screen cast, after the assembly is done, we create two
new `Element`, so that this assembly can be used as a _Part_ by other higher
level _Product_. These new `Element` link back to the lower level `Element`
inside the pin _Part_, and defines the interface of this axis _Part_.

# Multiply Axis Part

Now, we start to create the main assembly of this tutorial. First, we are going
to assemble the axis parts to the dome. We need three axis parts. The screen
cast below demonstrate the use of [Constraint Multiplication](Constraints-and-Solvers#constraint-multiplication)
feature. In order to use it, we must use a link array. Simply drag and drop the
axis part into the main assembly once, and then change the `ElementCount` to one,
which is one of the requirement of activating `Constraint Multiplication` button
![Multiply](../raw/master/Gui/Resources/icons/Assembly_ConstraintMultiply.svg?sanitize=true).

[[images/wind-turbine3.gif]]

Normally, we you click any geometry element in the 3D view, the tree view will
auto expand to the deepest object that actually owns the selected geometry. 
When you are working on complex assemblies with multiple levels of hierarchy,
you may find it annoying to try find the top level assembly you are working on.
You may find the new [navigation feature](Navigation) helpful. The screen cast
above shows another solution, that is, to freeze a sub-assembly. Once frozen,
the sub-assembly's `PartGroup` object will make a copy of all geometry shapes
of its parts. Any subsequent changes to the parts will not be reflected. A side
effect is that when selected in 3D view, the tree view will only expand until
the `PartGroup`, because `PartGroup` now owns all the geometry.

Let's continue and select the corresponding elements. A `PlaneCoincent`
constraint requires two elements. In order to activate the `Multiply` button,
you must select the element from the multiplying component **FIRST**, i.e. the
first array element of the axis part array. The newly created constraint will
be automatically selected, which will activate the `Multiple` toolbar button.
Proceed to click it. As shown in the screen cast, an error message shows up,
saying "Cannot create element in frozen assembly". This is because the
`multplication` command will try to auto expand the second element by creating
a new element containing all coplanar circular edges of the same radius, but
a frozen assembly does not allow creation of new elements. Simply unfreeze it,
select the constraint, and click the `Multiply` button again. The axis part
array will be multiplied to fit into all three holes.

# Assemble the Ring

Next, we'll assemble the ring to fix all three axis parts above. Note in the
screen cast above that we forgot to include all three corner elements, and we
can add them easily by drag and drop. 

[[images/wind-turbine4.gif]]

The first constraint is a simple `PlaneConstraint`. The following two
constraints, though, is not every intuitive. You need to know a little bit of
the notion of [DOF](https://en.wikipedia.org/wiki/Degrees_of_freedom_(mechanics)). 
Each free part has six DOF, three positional and three rotational.
A `PlaneCoincident` constraint takes away five of them, and leaving only one
free rotational DOF. In other word, after constrained, the second part can only
rotate along the normal axis of the constraining plane. The axis part is
constrained to the fixed dome with a `PlaneCoicedent`, leaving one DOF for this
two-part system, which is constrained again to the ring, which leaves two DOF
for this three-part system. And now we are going to connect another axis part,
which in fact is another two-part system. In other word, we are trying to
connect,

* a three-part two-DOF system, dome-axis-ring, to another
* two-part one-DOF system, dome-axis.

There are totally three DOF left, not enough for another `PlaneCoincedent`. You
can try with the `AxialAlignment` constraint 
![AxialAlignment](../raw/master/Gui/Resources/icons/constraints/Assembly_ConstraintAxial.svg?sanitize=true).
that takes away four DOF, meaning that the second part can still rotate and
move along the normal axis of the constraining plane. If you use
a `AxialConstraint`, it will work, but you'll get some complaining message at
the status bar when recomputing saying "redundant constraints". The default
constraint solver can handle some level of redundancy, but if left uncheck and
you keeps adding more redundancy, the solver sometimes may fail with no obvious
reason. The best solution is to use a two DOF constraint here, `PointOnLine`,
![PointOnLine](../raw/master/Gui/Resources/icons/constraints/Assembly_PointOnLine.svg?sanitize=true),
which restricts only two positional parameter. We select the first circular edge
in the axis part, which is used as the _point_ at its center. And the second
element, the corner arc of the ring, will be used as the _line_, which is the
normal vector of the arc plane. See
[here](Constraints-and-Solvers#constraining-geometry-element) for all possible
interpretations of various geometry elements.

When combining the above three-part system and two-part system, we have
a four-part system, dome-axis-axis-ring, with three DOF. After the
`PointOnLine` constraint, there is one DOF left. Finally, we combine the last
one DOF two-part dome-axis system with another `PointOnLine`, and there is no
DOF left. The assembly is completely fixed.

# Assemble the Gear

We are coming to the last part. The first constraint is another simple
`PlaneCoincident`. But the second one is again not very intuitive.

[[images/wind-turbine5.gif]]

As mentioned in the previous section, the dome-axis-ring part is completely
fixed with no DOF left, so after constraining the gear with the
`PlaneCoincident`, the whole system has only one DOF left, which means we need
to find an one-DOF-only constraint. With the slot shape in the gear, it is
natural to first think of, again, the `PointOnLine` constraint, but, it takes
away two DOF. Fortunately, this constraint works in 2D, which you can
[tell](Constraints-and-Solvers#constraints-with-more-than-two-elements) by the
look of its icon. A 2D point has only two DOF, and `PointOnLine` takes away one
of them, and thus completely fixes the whole system. Notice how the constraint is
created in the screen cast. You'll need to first select the element in the axis
part to used as the _point_, and then the corner element in the ring to be used
as the _line_, and finally the third element in the gear as the _projection plane_.

# Derived Assembly

Once an assembly is done, it is a good habit to freeze it. Besides the benefits
mentioned above, it also allows to be partially loaded when linked externally.
In other words, when you open a document containing an externally linked frozen
assembly, the external document containing the frozen assembly will only be
partially loaded. None of the assembly's children part will be fully loaded,
and if those children part are externally linked, none of their documents need
to be loaded, either.

In addition, there is another feature to help scale up even further. It's
called a _derived assembly_, which lets you create a new assembly inheriting the
parts from another one. The parts are added as locked parts. You can add new
parts to be constrained against them. The way it helps scaling up is that it
allows you to customize visibility of derived parts, as shown in the following
screen cast

[[images/wind-turbine6.gif]]


Imaging an assembly with complex internal mechanisms, you normally don't need
to show its internals when used as a _Part_ in higher level assembly. And this
is where the derived assembly can be of great help here. By hiding the internal
parts before freezing, we only save shapes of visible parts. And this makes
it load much quicker then the full version.

# Summary

Here is a summary of the tips mentioned in this tutorial to help dealing
with complex assemblies,

* Put each discrete part in its own file, and wrap it inside an assembly
  container;

* Explicitly create and manage the constraining element of each assembly,
  and give them proper names, while preferring sketch element, datum feature,
  or features in early model history;

* Explicitly use those elements in each assembly's `ElementGroup` to create
  the constraints;

* Freeze sub-assembly to reduce assembly hierarchy for easy navigation and
  quick loading;

* Use _derived assembly_ to customize part visibility and simplify assembly
  geometry.
