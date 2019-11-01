When dealing with complex assemblies, it become increasingly difficult to
navigate among various parts and hierarchies. `Assembly3` offers a few tools
to help with this.

# Selection Stack

The _Selection Stack_ is a new core functionality that records the user
selection before and after navigational jump. `Assembly3` exposed all current
supported navigation command in the new navigation toolbar, which are introduced
in the sections below. The usage of back and forward command
![Back](../../FreeCAD/raw/LinkStage3/src/Gui/Icons/sel-back.svg?sanitize=true)
![Forward](../../FreeCAD/raw/LinkStage3/src/Gui/Icons/sel-forward.svg?sanitize=true) 
is basically the same as those in a web browser.

# Relation Group

There are always three group objects in each `Assembly` container, the first of
which is the constraint group. This group contains the constraints that hold
all the parts of this assembly together. Sometimes, it is not obvious which
part objects are involved in which constraints. There is a hidden forth
_relation group_ containing the _relation_ objects. There is one relation object
for each part object of the assembly, and each relation contains a list of
constraints that are related in its representing part. In case the part is an
array, then its corresponding relation will contain child relations each
corresponding to an array element.

To reveal the relation group, select any part object and click
![Relation](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_GotoRelation.svg?sanitize=true).
It is not necessary to select the root part object. You can select any child
geometry in the 3D view, and click that button to go to the top level
assembly's relation object corresponding to the second level part object.

Alternatively, you can select any constraint object, and click the same button
to locate the same constraint object under the relation.

Although relation objects take only small amount of resources, they do take
extra recomputation time. You can safely delete the entire relation group if
they are not needed, and bring them back at any time using the above methods.

[[images/relation-group.gif]]

# Toggle Part Visibility

When working with multi-hierarchy assemblies, you may want to toggle the
visibility of some part object, but find it tedious to have to scroll the tree
up and down to locate the root object of the part. This task can be simplified
by clicking ![Vis](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_TogglePartVisibility.svg?sanitize=true).
Select any geometry in the 3D view that belongs to the part object you want to
hide, then click this button. It will find the root part object, collapse the
tree item, and then hide the object.

[[images/toggle-part-vis.gif]]

# Link Selection

There are a few toolbar buttons for easy link navigation.

When select a link, you can click
![LinkSelect](../../FreeCAD/raw/LinkStage3/src/Gui/Icons/LinkSelect.svg?sanitize=true) to
jump to the linked object. You can spot a link by the small arrow in the left
bottom of its icon, like [[images/link-object.png]].

For this to work better on external links, you may want to turn on tree sync
view function, by right click any where in the tree view, and select 
_Tree view options -> Sync view_. This option is not on by default, because it
may disturb existing user's work flow. Once you get used to it, you'll find it
to be very useful, especially working on multi-document projects.

Even if the object itself is not a link, but from an external document, and is
brought in by some link, you can use the same button to jump into that external
document. You can spot an external object by the small arrow in the right bottom
of its icon, like [[images/external-object.png]].

In case the link points to another link, you can use
![LinkSelectFinal](../../FreeCAD/raw/LinkStage3/src/Gui/Icons/LinkSelectFinal.svg?sanitize=true)
to directly jump to the final object. The `ElementLink` and `Element` in
`Assembly3` are multi-level links to link the constraining element through any
intermediate sub-assembly all the way till the deepest part model object. You
can use this button to easily find the actual object owning the geometry
element.

You can use 
![LinkSelectAll](../../FreeCAD/raw/LinkStage3/src/Gui/Icons/LinkSelectAll.svg?sanitize=true)
to find out all links that are linked to the current selected object.

In FreeCAD, the same object can be claimed by multiple features. You can use
![SelectInstance](../../FreeCAD/raw/LinkStage3/src/Gui/Icons/sel-instance.svg?sanitize=true)
to expand the tree to show all items corresponding to the current selected object.
