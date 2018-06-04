__New Feature__

* Support cylindrical surface in axial alignment and colinear constraint

[[images/cylinderical.gif]]


* New quick mover tool. Select any object inside an Assembly container, either
  from 3D viewer or the tree view, and click 
  ![QuickMove](../raw/master/Gui/Resources/icons/Assembly_QuickMove.svg?sanitize=true),
  the selected object will follow the mouse. Like the other mover tools, quick
  mover also support moving object within sub-assembly container. Simply add
  the sub-assembly as the second selection using CTRL select. As shown in the
  follow screen cast, by default the mover moves object in the top level
  assembly container, which is why the cube and cylinder in the sub-assembly
  moves together. To move the cube inside the sub-assembly, add it as the
  second selection. You may prefer to use keyboard shortcut to activate various
  mover, e.g. 'A, Q' for quick mover.


[[images/quick-move.gif]]

* Support yaw-pitch-roll angle locking, in composite constraint, including 
  PlaneCoincident, PlaneAlignment, AxialAlignment and MultiParallel. In addition,
  support X, Y and Z distance offset in PlaneCoincident constraint.

[[images/coincident-angle-and-offset.gif]]


* Support adding Origin structure into assembly. To add the origin, select the
  the assembly, or any object inside assembly, and click 
  ![AddOrigin](../raw/master/Gui/Resources/icons/Assembly_Add_Origin.svg?sanitize=true).
  To add origin to sub-assembly, you can add that sub-assembly as the second selection
  before clicking the button. Each assembly container can have only one origin. If there
  is already an existing origin, clicking the button will resize the origin to include 
  all objects inside the container. If the origin object is hidden, clicking the button
  will also reveal it. You may want to use keyboard shortcut 'A, O' for quick access.

[[images/origin.gif]]


* Add a checkable button ![LockMover](../raw/master/Gui/Resources/icons/Assembly_LockMover.svg?sanitize=true)
  to lock mover for fixed object. Once activated, the mover buttons will not activate
  when you select explicitly locked objects.

[[images/lock-mover.gif]]

* Support STEP export/import of assembly container.


__FreeCAD LinkStage3__

* Improve recomputation logic ([b2576c0a](/realthunder/FreeCAD/commit/b2576c0a9025d466881b7cf886bf46bbe00eec56))
  * If an error is reported by some object during recomputation, this object
    and all objects in its InListRecursive will be removed from the
    recomputation queue before resume recomputation.
  * If recompute() is called on some document marked as 'SkipRecompute', instead
    of doing nothing like previously, signal the Gui::Document to force recompute
    the editing object.

* Add 'Recompute object' action to tree view context menu. This will recompute
  the selected object and all its dependent objects.
  ([017da8a8](/realthunder/FreeCAD/commit/017da8a8621006037b3f4fafd30b5cc91d7e54af))

* Support context dependent color and visibility override in Link/LinkGroup (including
  the assembly container in asm3)
  ([a2f82b21](/realthunder/FreeCAD/commit/a2f82b2105d789f47c9124079ac14df7f9a66573),
  [2fcef036](/realthunder/FreeCAD/commit/2fcef036461cfde9efd1bcbe8fae7245f96f996c))


* Rework of STEP import/export to support geometric data sharing using App::Link, and
  color/visibility override
  ([66d95842](/realthunder/FreeCAD/commit/66d9584269384a0ff8e955ac182b7d74186a3a23),
  [99ae7ede](/realthunder/FreeCAD/commit/99ae7ede218a1eab6a4a09af204c3c2a5eb151d8))

[[images/step.gif]]
