__New Feature__

* Add co-linear constraint
* Allow creating XZ and ZY working plane. Or select any face and create a work
  plane on it. The work plane is not attached to the source. If you want an
  attached plane, use the new `SubShapeBinder` in PartDesign. It can be used
  without a body.

[[images/workplane.gif]]

__Bug Fix__

* Fix zero angle locking in various composite constraints (`PlaneCoincident`, 
  `PlaneAlignment`, `AxialAlignment`, etc)
* Fix negative offset in various composite constraints
* Fix line plane parallel/perpendicular constraints.

__FreeCAD LinkStage3__

* New [[Topological Naming]] framework
  ([e057cc17](/realthunder/FreeCAD/commit/e057cc1728a28b60b96bd5aafca00536924eda9d),
  [ba4a6e6c](/realthunder/FreeCAD/commit/ba4a6e6c1d8af75b0aac04f03aaec1cec16663c6),
  [c38ef04d](/realthunder/FreeCAD/commit/c38ef04d7a84633450e956e672d840b77f88201b),
  [579dee27](/realthunder/FreeCAD/commit/579dee27ceddcf566436bbbc61793f147bc8bd70)).

* Stabilized `Sketcher` geometry element naming.

* <a name="export"></a>New `SketchExport` to export any _private_ geometry of a sketch, including
  the construction lines, and external edges. The `Export` can have
  synchronized placement with the parent sketch, or independent placement, or
  even with its own plane attachment
  ([5fc101b3](/realthunder/FreeCAD/commit/5fc101b3290555faeec8adb5081e92e38cc9c387)).

[[images/sketch-export.gif]]

* Sketcher Geo-history reuses deleted geometry names for new elements that
  have the same topology
  ([2ddd4b44](/realthunder/FreeCAD/commit/2ddd4b44b900ff718b856ac54d9f99f31e091087)).

[[images/sketch-history.gif]]

* Two-pass recompute to allow some cases of dependency inversion, which is used on `SketchExport`.
  ([70c3d859](/realthunder/FreeCAD/commit/70c3d8596b36b411268b4e743146b4073999e7e2)).

* New `SubShapeBinder` in PDN supports relative linking that respects the
  parent group placement.
  ([73035a66](/realthunder/FreeCAD/commit/73035a6608921cb91bb0852e0b042e1f99469ec2)).

[[images/subshapebinder.gif]]

* Tree view supports multiple sub-elements drag and drop
  ([a2f057af](/realthunder/FreeCAD/commit/a2f057afb2dd9b45965cee717fd18ae54013410f)).


