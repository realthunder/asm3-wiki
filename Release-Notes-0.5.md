__New Feature__

* Add support for MacOS

__Bug Fix__

* Fix axial alignment redundancy handling

__FreeCAD LinkStage3__

* Introduce new `LinkScrop::Hidden` and the associated `PropertyLinkHidden`,
  `PropertyLinkSubHidden`, etc. The hidden links are not included in dependency
  calculation, but still have all the benefits of auto link tear down when
  linked object are deleted, and more importantly auto geometry reference
  update. One example use case is the new `ColoredElement` property in
  `Part::Feature` that stores the geometry reference for color override. This
  `PropertyLinkSubHidden` property links to the object itself.

* Enhanced [[Topological Naming|Topological-Naming-Algorithm]] support for tracing
  element evolving [[history|Topological-Naming-Algorithm#trace-model-history]].

* Complete support of new topological naming in `PartDesign` workbench

* Support [element color mapping](Topological-Naming-Algorithm#element-coloring)
  in `PartGui::ViewProviderPartExt` derived view providers, including `PartDesign` objects.

  [[images/partdesign-color.gif]]

