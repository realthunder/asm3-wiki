The design of Assembly3 (and the fork of FreeCAD) partially follows the
unfinished FreeCAD Assembly [project plan](https://www.freecadweb.org/wiki/Assembly_project), 
in particularly, the section [Infrustracture](https://www.freecadweb.org/wiki/Assembly_project#Infrastructure)
and [Object model](https://www.freecadweb.org/wiki/Assembly_project#Object_model), 
which are summarized below,

### Multi model

The forked FreeCAD core supports external object linking (with a new type of
property, `PropertyXLink`, as a drop-in replacement of PropertyLink),
displaying, editing, importing/exporting, and cross-document undo/redo.

### Part-tree

Assembly3 provides the `Assembly` container for holding its child features (or
sub-assembly), and their constraints. It also introduce a new concept of
`Elements` for declaring geometry elements used by constraints inside parent
assembly. The purpose of the `Element` is to minimize the problem caused by
geometry topological name changing, and make the assembly easier to maintain.
See the following section for more details. A single object (e.g. Part object,
sketch, or another assembly), can be added to multiple parent assemblies,
either within the same or located outside of the current document. Each of its
appearance inside the parent assembly has independent visibility control, but
shares the same placement, meaning that if you move one instance of the object,
all other instances moves relative to their parent assembly container. You can
have independent placement by converting a child object into a link type object
(See the following section for details). Simply right click the child object in
the tree view and select `Link actions -> Replace with link`.

### Unified Drag/Drop/Copy/Paste interface

The drag and drop API has been extended to let the target object know where the
dropped object located in the object hierarchy, which is taken full advantage
by Assembly3. The copy and paste is extended as well to be aware of external
objects, and let the user decide whether to do a shallow or deep copy.

### Object model

The `SubName` field in `Gui::SelectionObject` is for holding the selected
geometry sub-element, such as face, edge or vertex. The forked FreeCAD extended
usage of `SubName` to hold the path of selected object within the object
hierarchy, e.g. a selection object with `Object = Assembly1` and `SubName
= Parts.Assembly2.Constraints002.Constraint.` means the user selected the
`Constraint` object of `Assembly2`, which is a child feature of the part group
(`Parts`) in `Assembly1`. Notice the ending `.` in `SubName`. This is for
backward compatibility purpose, so that the `SubName` can still be used to
refer to a geometry sub-element of some sub-object without any ambiguity. The
rule is that, any sub-object references must end with a `.`, and those names
without an ending `.` are sub-element references. The aforementioned
`PropertyXLink` has an optional `subname` field (assign/return as a `tuple(obj,
subname)` in Python) for linking into a sub-object/element.

`Gui.Selection` is extended with backward compatibility to provide full object
path information on each selection, which makes it possible for the same object
to be included in more than one group like objects without ambiguity on
selection. Several new APIs have been added to FreeCAD core to provide nested
child object placement and geometry information. 
