__New Feature__

* Add ![QuickSolve](../raw/master/Gui/Resources/icons/Assembly_QuickSolve.svg?sanitize=true)
  button for quick solve. A normal solve can report any redundant constraints,
  but can also be very slow because of that. A quick solve (which is also used
  when auto solve is enabled) is faster as it does not perform redundancy
  checking.

* Add ![SmartRecompute](../raw/master/Gui/Resources/icons/Assembly_SmartRecompute.svg?sanitize=true)
  option to reduce number of recomputation required.

* Various changes to `Assembly` objects as listed below. See demo [here](https://youtu.be/uwPXDx-D4nY)
    * Change _Add origin_ ![AddOrigin](../raw/master/Gui/Resources/icons/Assembly_Add_Origin.svg?sanitize=true)
      to a checkable action, such that when enabled, the origin features will be
      automatically added to any newly created assemblies.

    * Constraint group object is auto hidden if empty.

    * Support `Element` and `ElementLink` in link navigation command.

    * Constraint element link now always link through a local element object, which
      retains a local copy of the linked geometry, and will continue to synchronize
      with the owner part's placement even if the lower element is missing. This
      can serve as a reminder for user to restore the missing element. See demo

    * Synchronize label change of `ElementLink` inside constraint with its linked `Element` 

    * Add context menu in `ElementGroup` to sort `Element` alphabetically

    * Add context menu in `Constraint` for disable and enable.

    * Add context menu in `Assembly` for freeze and unfreeze.

    * Introduce various measurement pseudo constraints,
        * _Points distance_ ![PointsDistance](../raw/master/Gui/Resources/icons/constraints/Assembly_MeasurePointDistance.svg?sanitize=true)
        * _Point plane distance_ ![PointPlane](../raw/master/Gui/Resources/icons/constraints/Assembly_MeasurePointPlaneDistance.svg?sanitize=true)
        * _Poin line distance_ ![Angle](../raw/master/Gui/Resources/icons/constraints/Assembly_MeasurePointLineDistance.svg?sanitize=true)
        * _Plane/Line angle_ ![Angle](../raw/master/Gui/Resources/icons/constraints/Assembly_MeasureAngle.svg?sanitize=true)

__FreeCAD LinkStage3__

* [Enhanced expression engine and spreadsheet](Expression-and-Spreadsheet)

* [Enhanced sketch external geometry](https://youtu.be/VB0guJ8kI70), which can be
    * Defining, i.e. used for shape building;
    * Frozen, i.e. no auto update when source geometry changes, but can be
      manually synchronized;
    * Detached from source geometry;
    * Re-attached.

* [Enhanced PartDesign SubShapeBinder](https://youtu.be/PY3PU4wDEwk), featuring
    * Support of binding to multiple external objects,
    * Support of partial document loading. Once enabled, the external document
      of the bound object does not need to be loaded.

* [TreeView object search], enhanced from upstream to support searching
  sub-objects, and objects from any document. Simply click anyway in the tree view
  and press `CTRL+F`.

* [New treeview options](https://youtu.be/8T_3tHOHsDw), including
    * _Sync view_, single click view switching, that supports switching to 3D
      view of external objects, TechDraw editing view, and spreadsheet;
    * _Sync selection_, auto expand and scroll to tree view item on 3D view;
      selection;
    * _Sync placement_, auto adjust object's placement when drag and drop so that
      the object does not jump inside 3D view;
    * _Pre-selection_, tree item mouse over highlight in 3D view;
    * _Record selection_, record tree view selection history so that you can go
      back and force with [navigation buttons](Navigation#selection-stack)
    * _Single/Multi/Collapse/Expand document_, inherited from upstream to enhance
      multiple document working experience.
    * _Initiate dragging_, allowing to initiate tree view item dragging by
      clicking this button (or better with keyboard shortcut) without holding
      the mouse button. This greatly enhanced drag and drop experience for
      large and complex hierarchies. Try it and you'll love it.
    * _Go to selection_, manually scroll to the tree item corresponding to the
      current 3D view selection, in case you disabled _Sync selection_ option.

* Introduce a second column in tree view as the user changeable description of
  any object.

* Introduce a core-built-in Python [logger](realthunder/FreeCAD/blob/adc477ef1ef50ce43463d72bbdc81766e992b7ae/src/App/FreeCADInit.py#L240), that supports dynamic log level changing.

* Enhanced Python console output. Log all previously non python console
  logging command, such as

```python
Gui.runCommand('Std_ToggleVisibility',0)
```

  For logging command with multiple outputs, insert comments to mark the start
  and end of the command, such as

```python
### Begin command Part_Box
App.ActiveDocument.addObject("Part::Box","Box")
App.ActiveDocument.ActiveObject.Label = "Cube"
App.ActiveDocument.recompute()
Gui.SendMsgToActiveView("ViewFit")
### End command Part_Box
```

  Log all GUI selection commands, such as

```python
Gui.clearSelection('Unnamed')
Gui.Selection.addSelection('Unnamed','Box001','Edge9',3.76698,0,0)
```

* Various bug fixes
