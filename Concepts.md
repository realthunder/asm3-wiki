# Coordinate System

The user is encouraged to first read 
[this tutorial](https://www.freecadweb.org/wiki/Assembly_Basic_Tutorial) to get some
idea about the new concept of _local coordinate systems_. The tutorial is for
the original unfinished Assembly workbench, but gives a pretty comprehensive
overview of what Assembly3 is providing as well. The `Part` or `Product`
container mentioned in the tutorial are equivalent to the `Assembly` container
in Assembly3, which of course can be treated just as a _part_ and added to
other assemblies. There is one thing I disagree with this tutorial. The concept
of _global coordinate system_ is still useful, and necessary to interoperate
with objects from other legacy (i.e. non-local-CS-aware) workbench. Let's just
define the _global coordinate system_ as the 3D view coordinate system, in
other word, the location where you actually see the object in the 3D view, or,
the coordinates displayed in the status bar when you move your mouse over some
object.

There is an existing container, `App::Part`, in upstream FreeCAD, which is
a group type object that defines a local coordinate system. The difference,
comparing to Assembly3 container, is that one object is allowed to be added to
one and only one `App::Part` container. The owner container can be added to
other `App::Part` container, but must still obey the one direct parent
container rule. The reason behind this is that when any object is added to
`App::Part`, it is physically removed from its original parent coordinate
system, and added to the owner `App::Part's` coordinate system, so the object
cannot appear in more than one coordinate system. By _physically removed_,
I mean the 3D visual representation data is physically moved to a different
coordinate system inside the 3D scene graph (See
[here](https://www.freecadweb.org/wiki/Scenegraph) for more details). 

Assembly3 container has no such restriction. When added to a Assembly3
container, the object's visual data is simply reused and inserted multiple
times into the scene graph, meaning that the object actually exists
simultaneously in multiple coordinate systems. This has a somewhat unexpected
side effect. When an object is added to an assembly with some placement, the
object is seemingly jumping into a new place (Update: there is a new feature
to auto adjust placement when dropping object across different coordinate
system. To activate it, right click anywhere in tree view, and select
`Tree options -> Sync placement`). This is expected, because the
object enters a new coordinate system, and it seems to have the same behavior
as `App::Part`. But what actually happened is that the original object inside
the global coordinate system is simply made invisible before adding to the
assembly container. You can verify this by manually toggle the `Visibility`
property to reveal the object in its original placement. Every object's
`Visibility` property controls its own visibility in the global coordinate
system only. Each assembly container has the `VisibilityList` property to
control the visibilities of its children.

# Link

The forked FreeCAD core introduced a new type of object, called _Link_.
A _Link_ type object (not to be confused with a _link property_) often does not
have geometry data of its own, but instead, link to other objects (using link
property) for geometry data sharing. Its companion view provider,
`Gui::ViewProviderLink`, links to the linked object's view provider for visual
data sharing. It is the most efficient way of duplicating the same object in
different places, with optional scale/mirror and material override. The core
provides an extension, `App::LinkBaseExtension`, as a flexible way to help
users extend their own object into a link type object. The extension utilize
a so called _property design pattern_, meaning that the extension itself does
not define any property, but has a bunch of pre-defined property place holders.
The extension activates part of its function depending on what properties are
defined in the object. This design pattern allows the object to choose their
own property names and types. 

The core provides two ready-to-use link type objects, `App::Link` and
`App::LinkGroup`, which expose different parts of `LinkBaseExtension's`
functionality. `App::Link` supports linking to an object, either in the same or
external document, and has built-in support of array (through property
`ElementCount`) for efficient duplicating of the same object. `LinkGroup` acts
like a group type object with local coordinate system. It relies on
`LinkBaseExtension` and `ViewProviderLink` to provide advanced features like,
adding external child object, adding the same object multiple times, etc. All
of the Assembly3 containers are in fact customized `LinkGroup`. 

# Element

`Element` is a brand new concept introduced by Assembly3. It is used to
minimize the dreadful consequences of geometry topological name changing, and
also brings the object-oriented concept in the programming world into CAD
assembling. `Element` can be considered as a declaration of connection
interface of the owner assembly, so that other parent assembly can know which
part of this assembly can be joined with others. 

For a geometry constraint based system, each constraint defines some
relationship among geometry elements of some features. Conventionally, the
constraint refers to those geometry elements by their topological names, such
as `Fusion001.Face1`, `Cut002.Edge2`, etc. The problem with this simple
approach is that the topological name is volatile. Faces or edges may be
added/removed after the geometry model is modified. More sophisticated
algorithm can be applied to reduce the topological name changing, but there
will never be guarantee of fixed topological names. Imagine a simple but yet
extreme case where the user simply wants to replace an entire child feature,
say, changing the type of some screw. The two features are totally different
geometry objects with different topological naming. The user has to manually
find and amend geometry element references to the original child feature in
multiple constraints, which may exists in multiple assembly hierarchies, across
multiple documents.

The solution, presented by Assembly3, is to use abstraction by adding multiple
levels of indirections to geometry references. Each `Assembly` container has an
element group that contains a list of `Elements`, which are a link type of
object that links to some geometry element of some child feature of this
assembly. In case the feature is also an `Assembly`, then the `Element` in
upper hierarchy will instead point to the `Element` inside lower hierarchy
assembly. In this way, each `Element` acts as an abstraction to which geometry
element can be used by other parent assemblies. Any constraint involving some
assembly will only indirectly link to the geometry element through an `Element`
of some child assembly. If the geometry element's topological name changes due
to whatever reason, the user only needs to change the deepest nested (i.e.
nearest to the actual geometry object) `Element`'s link reference, and all
upper hierarchy `Elements` and related constraints stays the same. 

The `Element` is a specialized `App::Link` that links into a sub-object, using
a `PropertyXLink` that accepts a `tuple(object, subname)` reference. In
addition, `Element` allows to be linked by its label, instead of the immutable
internal FreeCAD object name. `Element` specifically allows its label to be
duplicated (but still enforces uniqueness among its siblings). This enables the
user to define inter-changeable parts with the same set of elements as
interface. 

Let's take a look at the following assembly hierarchy for an example,

```
Assembly001
    |--Constraints001
    |       |--Constraint001
    |               |--ElementLink -> (Elements001, "$Element.")
    |               |--ElementLink001 -> (Parts001, "Assembly002.Elements002.$Element001.")
    |--Elements001
    |     |--Element -> (Parts001, "Cut.Face3")
    |--Parts001
          |--Cut
          |--Assembly002
                 |--Constraints002
                 |--Elements002
                 |      |--Element001 -> (Parts002, "Assembly003.Elements003.$Element002.")
                 |--Parts002
                       |--Assembly003
                                |--Constraints003
                                |--Elements003
                                |       |--Element002 -> (Parts003, "Fusion.Face1")
                                |--Parts003
                                       |--Fusion
```

The `Assembly001` has two child features, a `Cut` object and a child
`Assembly002`, which in turn has its own child `Assembly003`. `Assembly001`
contains a constraint `Constraint001` that defines the relationship of its two
child features. `Constraint001` refers to two geometry element through two
links, `ElementLink`, which point to a second level link, `Element`.
`ElementLink001` points to `Element001`, And, because the first child feature
`Cut` is not defined as an assembly, its geometry element reference is directly
stored inside the parent assembly element group. `Element001`, however, links
to the lower hierarchy `Element002` in its child assembly, which again links to
`Element003` in its child `Assembly003`. Notice the `$` inside the subname
references. It marks the followed text to be a label instead of an object name
reference. If you re-label the object, all `PropertyXLink` of all opened
documents containing that label reference will be automatically updated. 

The grand idea is that, after the author modified an assembly, whether its
a modification to the geometry model, or replacing some child feature. He needs
to check all element references inside that and only that assembly, and make
proper adjustment to correct any undesired changes. Other assemblies with
elements or constraints referring to this assembly will stay the same (although
recomputation is still required), even if those assemblies reside in different
documents, or come from different authors.

Let's say, we have modified `Fusion`, and the original `Fusion.Face1` is now
changed to `Face10`. All we need to do is to simply modify `Element002` inside
the same owner assembly of `Fusion`. Everything else stays the same. 

Again, say, we want to replace `Assembly003` with some other assembly. Now this
is a bit involving, because, we added `Aseembly003` directly to `Assembly002`,
instead of using a link, which can be changed dynamically. The FreeCAD core has
a general command to simplify this task. Right click `Assembly003` in the tree
view, and select `Link actions -> Replace with link`. `Assembly003` inside
`Parts002` will now be replaced with a link that links to `Assembly003`. Every
relative link that involving `Parts002.Assembly003` will be updated to
`Parts002.Link_Assembly003` automatically. In our case, that will be
`Element001`. You can then simply change the link to point to another assembly
containing an element object with the same label `Element001` (remember element
object allows duplicated labels). If you still insist on adding the new
assembly directly and get rid of the link, you can use `Link actions ->
unlink`, and delete the link object afterward.

It may seem intimidating to maintain all these complex hierarchies of
`Elements`, but the truth is that it is not mandatory for the user to manually
create any element, at all. Simply select any two geometry elements in the 3D
view, and you can create a constraint, regardless how many levels of
hierarchies in-between. All intermediate `Elements` and `ElementLinks` will be
created automatically. Although, for the sake of re-usability, it is best for
the user as an assembly author to explicitly create `Element` as interfaces,
and give them proper names for easy (re)assembling. Check out [[this|Replacing-Part]]
tutorial for a demonstration of part replacement.

Last but not the least, `Element`, as well as the `ElementLink` inside
a constraint, make use of a new core feature, `OnTopWhenSelected`, to
forcefully show highlight of its referring geometry sub-element (Face, Edge,
Vertex) when selected, regardless of any obscuring objects. The property
`OnTopWhenSelected` is available to all view object, but default to `False`,
while `Element` and `ElementLink` make it active by default. The on-top feature
makes it even easier for the user to check any anomaly due to topological name
changing.

# Selection

There are two types of selection in FreeCAD, geometry element selection by
clicking in the 3D view, and whole object selection by clicking in the tree
view. When using Assembly3, it is important to distinguish between these two
types of selection, because there are now lots of objects with just one
geometry element. While you are getting used to these, it is helpful to bring
out the selection view (FreeCAD menu bar, `View -> Panels -> Selection view`).
You select a geometry element by clicking any unselected element (Face, Edge or
Vertex) in the 3D view. If you click an already selected element, the selection
will go one hierarchy up. For example, for a `LinkGroup` shown below,

```
LinkGroup
    |--LinkGroup001
    |       |--Fusion
    |       |--Cut
    |--Cut001 
```

Suppose you have already selected `Fusion.Face1`. If you click that face again,
the selection will go one hierarchy up, and select the whole `Fusion` object.
If you click any where inside `Fusion` object again, the selection goes to
`LinkGroup001`, and you'll see both `Fusion` and `Cut` being highlighted. If
you again click anywhere inside `LinkGroup001`, `Cut001` will be highlighted,
too, because the entire `LinkGroup` is selected. Click again in `LinkGroup`,
the selection goes back to the geometry element you just clicked. 

There is a new feature in the forked FreeCAD selection view. Check the `Enable
pick list` option in selection view. You can now pick any overlapping geometry
elements that intersect with your mouse click in the selection view. 

You may find it helpful to turn on tree view selection synchronization (right
click in tree view, select `Sync selection`), so that the tree view will
automatically scroll to the object you just selected in the 3D view. When you
select an originally unselected object in the tree view, the whole object will
be selected. And if you start dragging the object item in the tree view, you
are dragging the whole object. If you select a geometry element in the 3D view,
its owner object will also be selected in tree view. But if you then initiate
dragging of that particular object item, you are in fact dragging the selected
geometry element. This is an important distinction, because some containers,
such as the `Constraint` object, only accept dropping of geometry element, and
refuse whole object dropping.

* 2D/3D Sketch

The usage of sketch in assembly is a technique called _Skeleton Modeling_. It
is a type of top-down design approach, where you draw skeleton (lines, arcs,
etc) sketches to specify design criteria, and then add individual components
that references those criteria. Some type of mechanical systems are naturally
modeled in 2D, e.g. a pin joint that can only rotate in a plane, while others
must be modeled in 3D, e.g. a ball joint.

FreeCAD already has a powerful _Sketcher_ workbench, but it is limited to
creating 2D sketches. In addition, the sketcher is more geared toward geometry
modeling, i.e. creating bases for extrusion, pocketing, etc. The sketch object
stores all elements and constraints inside the object itself. It has its own
editor and solver, which makes it probably the most complex single object in
the entire FreeCAD. Instead of modifying the sketch object to suit for assembly
needs, Assembly3 opt to repurpose some of the objects from the _Draft_
workbench without modification. The draft workbench, I believe, was originally
used for sketching purposes before the birth of the more specialized sketcher. 

At the time of this writing, Assembly3 supports using draft wire and circle/arc
for sketching purpose. For normal objects added to an assembly container as
a part, the placement of the object is used as constraining parameters (see
[[here|Constraints-and-Solvers]] for more details). Draft wire is treated
specially. Instead of constraining on the object placement, it is constrained
on the coordinates of each individual points. Draft circle is still constrained
on its placement like other objects, but with an additional parameter for its
radius, and draft arc, two more parameters for the first and last angle that
defines its two end points. Draft wires has many use cases, not all of which
are acceptable by sketching constraint (e.g. `LineLength`). Only non-subdivided
wires without a base or tool object attached are accepted.

The draft objects added are free to move in 3D space. To create a 2D sketch,
you can add a `SketchPlane` constraint. It accepts any planar edge or face as
the first element for defining the current sketching plane. Any draft
object elements involved in the following constraints will be confined to
this plane implicitly. You can also explicitly add elements into the 
`SketchPlane` constraint. To reset the current sketch plane, i.e. make the
following draft elements free in 3D, add an empty `SketchPlane` constraint.
You can have more than one sketch plane defined in the same assembly container.

You can checkout [[this|Create-Skeleton-Sketch]] tutorial to find out more
details


