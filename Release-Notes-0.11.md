__New Feature__

* Add ![AutoElementFix](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_AutoFixElement.svg?sanitize=true)
  button for automatic fix of broken element references. It utilizes the new core feature to search
  the linked object for sub-shape that are geometrically similar to the one stored in the element object.
  Enabling this option will activate `Auto Fix` for all elements. You can also disable the option here,
  and opt to manually fix the element using the new `Fix` tree view menu action of the element object.

* New element tree view menu actions to offset/flip the element or the owner part. See
  [here](Constraints-and-Solvers#user-content-element-actions) for more details.

__FreeCAD LinkStage3__

* New UI feature [docked window overlay mode](https://forum.freecadweb.org/viewtopic.php?f=34&t=45349)

* Various 3D picking and visualization [enhancement](https://forum.freecadweb.org/viewtopic.php?f=17&t=41103)
    * Add object editing menu in 3D context menu
    * Add 'Pick geometries' action in 3D context menu, which allows for picking
      hidden and high level (wire, solid, compound, etc.) geometries.

* Refactor [expression completer](https://forum.freecadweb.org/viewtopic.php?f=17&t=43412)

* Parametric modeling using [spreadsheet configuration table](https://forum.freecadweb.org/viewtopic.php?f=17&t=42183)

* Various PartDesign enhancement
    * Support Link or SubShapeBinder as base feature
    * Use SubShapeBinder for boolean operation
    * Feature based visualization
    * Editing enhancement for dress up features (fillet/chamfer/draft) and pattern transform features
    * Support pattern transform of dress up features
    * Add feature Split

* Topological naming auto fix using geometry search

* Improve document recomputation efficiency by doing property comparison, i.e.
  skip unnecessary recomputation if property content is unchanged.

* Add view option for auto transparency on all objects except those in editing.
  Currently only applies to editing sketches.

* Allow saving document to directory without archive, for better support with
  external version control software.
