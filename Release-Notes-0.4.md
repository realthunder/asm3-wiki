__New Feature__

* Constraint/Element auto visible feature now supports highlight on mouse over
  tree view item.
* Turn on SolveSpace solver backend redundant constraint tolerance.
* Auto relax [certain combinations](../commit/80bceefcc69f80ae6812628a5a2368c0bbdd0a77) of constraints.

__Bug Fix__

* Fix (again) negative offset in various composite constraints
* Tolerate empty lock constraint
* Fix mover on whole object selection

__FreeCAD LinkStage3__

* Complete [[Topological Naming|Topological-Naming-Algorithm]] support in
  `Part` workbench and other Python features.
  ([1c7b973b](/realthunder/FreeCAD/commit/1c7b973b3ebd1721268d07c40afa8cf613cc0917),
  [765bb3b8](/realthunder/FreeCAD/commit/765bb3b83f909fde3657a5b23eff344d323ae5d4),
  [83a58cd9](/realthunder/FreeCAD/commit/83a58cd9b74ecb765fa1cc7decea0ba2d6e44582),
  [8156ff23](/realthunder/FreeCAD/commit/8156ff23ba001d21c73a7519f4f7f0ad39f5b9cb),
  and many following bug fixes)

* 3D view highlight when mouse over tree view item. Once highlighted, the 3D
  view will add the object to the always-on-top display group, and make it
  transparent. This mouse over highlight feature can be turned on/off by tree
  view context menu, `Tree view options -> Pre-select`.
  ([381e79ba](/realthunder/FreeCAD/commit/381e79ba9ec7310db9acf22816bf034fae80971b))

  [[images/mouse-over-highlight.gif]]

* Refactor `PartDesign::SubShapeBinder` to support design-in-context. The new
  shape binder now works in `Relative` linking mode by default, meaning that
  when you drag and drop a selection to the binder object, it will link to the
  first non-common ancestor of the target, and use sub-name to refer to the
  linked child object(s). If the binder belongs to some parent object (e.g.
  a body) and is moved, you can re-synchronize the placement by double clicking
  the binder object in the tree view. The binder also has a new `BindMode`
  property, that is default to `Synchronized`, meaning that it will auto update
  itself whenever the bound shape is changed. If set to `Fronzon`, then the
  shape will not be updated unless you double click the binder item in the tree
  view. When set to `Detached`, the shape is obtained at the time the link is
  set (e.g. through drag and drop), after which the link will be immediately
  reset to empty.
  ([76744223](/realthunder/FreeCAD/commit/76744223521dd1f32490731459e588dd269d4d0b),
   [c2c2c2aa](/realthunder/FreeCAD/commit/c2c2c2aa1c83f927be0178052d858264d312676b))

  [[images/shapebinder-context.gif]]

