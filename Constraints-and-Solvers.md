Assembly3 supports multiple constraint solver backend. The user can choose
different solver for each `Assembly`. The type of constraints available may be
different for each solver. At the time of this writing, two backend are
supported, one based on the solver from [SolveSpace](http://solvespace.com/index.pl), 
while the other uses [SymPy](http://www.sympy.org/en/index.html) and 
[SciPy](https://www.scipy.org/), and is modeled after SolveSpace. In other
word, the current two available solvers supports practically the same set of
constraints and should have very similar behavior, except some difference in
performance. 

Assembly3 exposed most of SolveSpace's constraints. In addition, it also
provides several new constraints that are more useful for assembly purposes.
Most of these constraints are in fact composite of the original constraints
from SolveSpace. All of the constraints are available as toolbar buttons, but
only the more common ones are visible by default. To reveal all the constraints, click 
![More](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintMore.svg?sanitize=true).
See the [section](#list-of-constraints) below for links to details of each
individual constraint.

# Creating Constraint

Creating constraint is easy. Simply create an assembly by clicking 
![AddAssembly](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_New_Assembly.svg?sanitize=true),
drag in the parts, select some geometry elements of the parts in 3D view,
and click one of the constraint buttons to create the constraint. The constraining
elements will appear as child objects under the constraint object. By default,
auto element visibility function is active, which will hide the constraining
element, and auto reveal itself when being selected, either directly, or when
its parent constraint is selected. You can deactivate this function by clicking
![AutoVis](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_AutoElementVis.svg?sanitize=true),
and control the element visibility manually as shown in the following screen cast.

[[images/create-constraint.gif]]

When you create an assembly, it is advisable to first choose a _base part_,
which is supposed to have a fixed placement relative to its containing
assembly. You do this by creating a [[Lock]] constraint 
![Lock](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintLock.svg?sanitize=true),
as shown in the above screen cast. You can still move the _locked_ part, either using various
[[Mover]] tools (![Mover](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_Move.svg?sanitize=true),
![AxialMover](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_AxialMove.svg?sanitize=true),
![QuickMover](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_QuickMove.svg?sanitize=true)),
or directly modify the part's `Placement` property. The `Lock` constraint only
prevent the assembly solver from changing the part's location. If you moved
a locked part, every other part will gather around the new location of the
locked part, after solver recompute. If you don't want to accidentally activate
the mover for locked parts, click 
![LockMover](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_LockMover.svg?sanitize=true).

Each `Constraint` object has a `Type` property, which allows the user to change
the type of an existing constraint using the property editor. Not all
constraints require the same types of geometry elements, which means that
changing the type may invalidate a constraint. The tree view will mark those
invalid constraints with a red exclamation mark. Hover the mouse over those
invalid items to see the explanation. You can reorder the elements using 
![Up](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_TreeItemUp.svg?sanitize=true) and
![Down](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_TreeItemDown.svg?sanitize=true) 
buttons. You change an existing constraining element by selecting a new face or
edge in the 3D view and drag the corresponding tree item and drop it over the
element item.

# Constraining Geometry Element

`Assembly3` is very flexible about the interpretation of a constraining geometry
element. When a constraint is named as `PointOnLine`, it does not mean the
constraint only accepts vertex and linear edge. Here is a list of alternative
interpretations of an element,

* Point
    * Coordinate of a vertex;
    * Middle point of a linear edge;
    * Center of a circular edge;
    * Center of bounding box of any other edge;
    * Center of bounding box of a planar face;
    * Center of a circular face;
    * Center of the first edge (i.e. `Edge1`) of a cylindrical face.
* Line, (of which only the direction is relevant for constraining, not the end
  points)
    * Linear edge;
    * The normal vector of the surface of a circular edge;
    * The normal vector of the surface of a planar face;
    * The revolving axis of a cylindrical face.
* Plane, defined by an origin point interpreted the same way as the above
  _Point_, and a normal vector
    * The surface of a circular edge;
    * The surface of the first edge (i.e. `Edge1`) of a cylindrical face;
    * The surface of a planar face.

One thing that a serious user may find worth getting familiar with is the
[Element](Concepts#Eelment) concept in `Assembly3`. The elements used by
various constraints never reference the actual part object directly, but
instead, through an intermediate object stored in the `ElementGroup`. If the
geometry belongs to some model in a sub-assembly, then the `Element` object is
stored in the `ElementGroup` of that sub-assembly instead of the parent's. You
can consider the `Elements` as a declaration of geometry interfaces that this
part (as a sub-assembly) can be constrained in higher level assemblies. Many
advanced feature is made possible through this extra indirection, such as 
[part replacement](Replacing-Part). 

As shown in the previous section, you can create constraint by simply selecting
a face or edge of the part model. What is happening behind the scene is that
the program will find the sub-assembly that owns the selected part geometry
model, and create an `Element` object in its `ElementGroup` if there isn't one
existing, and finally create an `ElementLink` referencing the `Element`, and
put it under the new constraint. Instead of directly selecting geometry
elements of the model in 3D view, you can also select any existing
`ElementLink` or `Element` in the tree view to create new constraints, which is
in fact the preferable way to do, as illustrated in this
[tutorial](How-to-Handle-Large-Scale-Assembly).

If you are constraining a part object directly without using any sub-assembly,
then the `Element` will be stored directly in the current assembly. This is
allowed for quick testing of simple assemblies, and also to make it easy for
beginners to get started. For complex real world assemblies, it is strongly
recommended to wrap each part object inside an assembly before further
assembling, so that each geometry reference to the part model can be grouped
together for easy updating through drag and drop. See, again, 
[this](How-to-Handle-Large-Scale-Assembly) tutorial for more tips of handling
complex assemblies.

# Geometry Element Offset

Each `ElementLink` has an `Offset` property to allow you to apply some
transformation to the element before constraining. The offset is always in the
geometry element's coordinate space. For example, for a planar face, the z axis
of the offset placement is always pointing to the normal of the face. Because
`ElementLink` is used by one and only one `Constraint`, its offset does not
affect other constraints. The `Element` object also has an `Offset` property
with the same purpose. However, because the `Element` may be referenced by
multiple `ElementLink`, changing the offset may affect multiple constraints.
This is useful, for example, to introduce some allowance between parts when
assembling.

The following screen cast shows the usage of element `Offset`, and highlights
the difference between applying offset on `ElementLink` and `Element`.

[[images/element-offset.gif]]

<a name="element-actions"></a>Starting from version 0.11, there is a few new
context menu actions to simplify manipulating element offset. Simply right
click any `Element` or `ElementLink` item in the tree view to bring up the
context menu, available actions are,

* `Flip element`: Flip the Z normal of the element by rotation 180 degrees along
  its X axis. If `CTRL` key is pressed while activate this action, it will rotate
  along the element's Y axis. Note that `Flip element` is most effective when
  used in `Attachment` constraint, because the constraint follows the element's
  Z normal unambiguously. For other type of constraint, like `PlaneConcidence`
  or `PlaneAlignment`, you may want to try the `Flip part` menu action, because
  these types of constraint works the same regardless of the element Z normal
  orientation, so flipping the element will have no effect.

* `Offset element`, use the dragger (similar to `Mover` tool) to manipulate element
  offset directly in the 3D view.

* `Reset offset`, reset element to no offset.

# Constraints with More Than Two Elements

It is intuitive for one to select two geometry elements and create
a constraint. However, in `Assembly3`, there are many constraints that can
accept more than two elements.

On the constraint toolbar, when you see an icon with a red dashed border like 
![Angle](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintAngle.svg?sanitize=true),
it means the constraint accepts a third optional element as a projection plane
to project the first two elements into 2D space. Although 2D constraints are
more useful when creating [skeleton-sketch](Create-Skeleton-Sketch), you may
find it effective when dealing with the _redundant constraint_ problem, as
a way to reduce the number of [DOF](https://en.wikipedia.org/wiki/Degrees_of_freedom_(mechanics))
constrained.

For icons with a red border like
![PointsVertical](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintPointsVertical.svg?sanitize=true),
it means this is a 2D only constraint, and the third projection plane element
is mandatory.


For icon with a green border like 
![AxialAlingment](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintAxial.svg?sanitize=true),
it means the constraint accepts multiple elements of the same type (with a few
exceptions). The solver will expand the constraint as follow,

```
Constraint
    | -- ElementLink1
    | -- ElementLink2
    | -- ElementLink3
    | -- ElementLink4

will be expand to

Constraint1
    | -- ElementLink1
    | -- ElementLink2

Constraint2
    | -- ElementLink3
    | -- ElementLink2

Constraint3
    | -- ElementLink4
    | -- ElementLink2
```

For icons with a blue border like
![PlaneCoincident](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintCoincidence.svg?sanitize=true),
it supports the same multiple elements like those with green boarder.
In addition, the constraint has a `Cascade` property, and when set
to `True`, it will expand the constraints as follow,

```
Constraint
    | -- ElementLink1
    | -- ElementLink2
    | -- ElementLink3
    | -- ElementLink4

will be expand to

Constraint1
    | -- ElementLink1
    | -- ElementLink2

Constraint2
    | -- ElementLink2
    | -- ElementLink3

Constraint3
    | -- ElementLink3
    | -- ElementLink4
```

`Constraint2` and onwards will be automatically skipped if their elements
belong to the same part. See the following screen cast for an example, which
also shows the effect of applying other parameters of a constraint when
cascading is activated.

[[images/element-cascade.gif]]

# Constraint Multiplication

There are some multi-element capable constraints that support another advanced
mode, called _constraint multiplication_, which uses `Link Array` to multiply
the part of the first element to be constrained against the rest of the
elements. At the time of this writing, only `PlaneConicdent` and
`AxialAlignment` support this. You can activate this functionality by selecting
a supporting constraint object in the tree view and then click 
![Multiply](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_ConstraintMultiply.svg?sanitize=true).
There is one additional requirement. In order to activate that button. The
owner part of the first constraining element must be the first instance of
a `Link Array`. 

If the second constraining element and onwards is a circular edge, it will be
automatically expanded to include all coplanar circular edges of the same
radius. The following screen cast shows the default behavior of constraint
multiplication,

[[images/constraint-multiply.gif]]

Notice that the first step shown in the above screen cast is to replace
a directly added part object with a link, and then change it to an array by
setting the `ElementCount` property. If you drag an object from an external
document and drop it in an assembly container, a link will be created
automatically, so you don't need this _replace with a link_ step. The screen
cast also shows that you can add more constraining element by drag and drop.
And the new element will be expanded, too. You can disable element auto
expansion, by changing the `NoExpand` property as shown in the following screen
cast. If the last added element has `NoExpand` set to `True`, any new elements
added afterwards will not be expanded either. This allows you to choose exactly
which elements to multiply. You can also manually control the total number of
instances of the first part array, by turning off the `AutoCount` property of
the first element.

[[images/constraint-multiply2.gif]]

One disadvantage of multiplying a constraint this way is that you are forced to
use the same set of parameters for all part instances. If you want to have
different parameters, say, a different offset for each screw instance, you can
select the multiplied constraint, and click again
![Multiply](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_ConstraintMultiply.svg?sanitize=true).
This time, the constraint will be expanded into multiple independent constraint
objects equivalent to the multiplied constraint for easy customization.

[[images/constraint-multiply3.gif]]

# List of Constraints

* [[Lock]]
* [[Plane Alignment]]
* [[Plane Coincident]]
* [[Axial Alignment]]
* [[Same Orientation]]
* [[Multi Parallel]]
* [[Angle and Perpendicular]]
* [[Points Coincident]]
* [[Points in Plane]]
* [[Points on Circle]]
* [[Points Distance]]
* [[Points Plane Distance]]
* [[Point Line Distance]]
* [[Symmetric]]

# Implementation

Assembly3 exposed most of SolveSpace's constraints. If you want to know more
about the internals, please read 
[this document](https://github.com/realthunder/solvespace/blob/python/exposed/DOC.txt)
from SolveSpace first. The way Assembly3 uses the solver is that, for a given
`Assembly`,

* Create free parameters corresponding to the placement of each movable child
  feature, that is, three parameters for position, and four parameters for its
  orientation (quaternion).
* For each constraint, create SolveSpace `Entities` (i.e. points, normals,
  rotations, etc) for each geometry element as a transformation of its owner
  feature's placement parameters.
* Create SolveSpace `Constraints` with the `Entities` created in the previous
  step.
* Ask SolveSpace to solve the constraints. SolveSpace formulate the constraint
  problems as a non-linear least square minimization problem, generates
  equations symbolically, and then tries to numerically find a solution for the
  free parameters.
* The child features placements are then updated with the found solution.

For nested assemblies, Assembly3 will always solve all the child assemblies
first before the parent.

One thing to take note is that, SolveSpace is a numerical solver, which means
it is sensitive to initial conditions. In other word, you must first roughly
align two features according to their constraints, or else the solver may not
be able to find an answer. Assembly3 has extensive support of easy manual
placement of child feature. See the following section for more details.
