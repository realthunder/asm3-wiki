Starting from version 0.3, support of stable topological naming of geometry
element has been added to my FreeCAD Link branch. Because Assembly3 is the main
test environment, the document is put here.

Update: Version 0.4 now has full support of element mapping in all features
from the `Part` workbench. In addition, element mapping is fully supported in 
Python. Most python features automatically gain the benefit of element mapping
without any code modification.


# Overview

Current FreeCAD uses generic `type` + `index` as the topological names of
various geometry elements, such as `Face1`, `Edge2`, and `Vertex3`. The indexing
order of the elements is determined by the CAD kernel, 
[OCCT](https://www.opencascade.com/content/documentation), which is consistent
during document save and restore. However, when the model is edited, i.e.
when new geometry elements are added or removed, the element indices are
rearranged/reused, and there is no easy way of tracking which one is which after
the modification.

There has been other attempts trying to solve this problem, e.g.
[here](https://forum.freecadweb.org/viewtopic.php?f=10&t=23399), and
[here](https://docs.google.com/document/d/1-d2JD8RH13ar7QPh_SpX2H5eiNND4DiCPBmGwA2R9Ug/edit)
And various research papers posted in the forum, such as 
[this](https://www.sciencedirect.com/science/article/pii/S2288430015300117).
They all served as great inspirations leading to the solution presented here,
which is offered as a general framework that is

* __fully backward compatible__, meaning that you can save your model in new
  version of FreeCAD and still able to open it in older version of the software.
  You will loose all new style topological naming of the model, obviously. But
  your link properties continues to work; and,

* __automatic__, with very little code modification of existing workbench,
  because the framework resides in the core, and other workbench get the benefit
  through various property links which are enhanced. When you open models
  created in older version of FreeCAD, it automatically gains the benefit of the
  new topological naming feature after recompute.

The framework deals with abstract geometry element name mapping, and name
reference auto update. The core does not know or care how the topological names
are generated and mapped. It provides necessary ground works for other modules
to generate stable topological names. At the time of this writing,
`Sketcher.SketchObject` can now provide a complete stable topological naming,
and `Part.TopoShape` also has some partial implementation of the topological
name generation, which can already serve as a good demonstration of the
effectiveness of the overall framework.

# The Framework

In the current FreeCAD core, the interface class that describes a geometry shape
is called `ComplexGeoData`. It has been extended to include an optional
`ElementMap`, with the following new APIs in both C++ and Python,

* `setElementName(element,name, prefix=None, postfix=None, overwrite=False)`. It is used to assign a new 
  `name` to an `element` with optional prefix and/or postfix, e.g. `setElementName('Edge1','e1')`. An element
  can have multiple names, but a name can only be associated to one element.
  You can overwrite existing mapping by the `overwrite ` parameter. The purpose
  of prefix and postfix is to make it easy for the caller to form meaningful
  names, such as use the prefix to identify operation, and postfix to identify
  modification. More importantly, as the name gets longer, `ComplexGeoData`
  allows the caller to compress the name using string has, but it will ensure
  that the prefix and postfix remains the same, for more advanced applications
  like deducing modified elements, etc. More details about string hashing will
  be given in later section

* `getElementName(name,reverse=False)`. Returns the original element name
  by the given `name`, or, the other way around if `reverse` is true.

The entire element map is also exposed to C++ with `set/getElementMap()`, and 
as writable attribute called `ElementMap` in Python.

When exposed to `Gui.Selection`, `ComplexGeoData` uses a special prefix to mark
the mapped element. Currently, `;` is chosen as the prefix. When calling
`setElementName()`, the special prefix is optional, and will be stripped away before
storage, same for `getElementName()`. More details about the involvement of
`Gui.Selection` will be given later.

`ComplexGeoData` does not know or care the content stored in the map. It is up
to the inherited geometry classes to provide the semantic meaning of the 
content. These APIs also make it possible for other core classes to work
their magic to provide seamless upgrade. Here is how it works,

## GeoFeature

`GeoFeature`, the core geometry document object class, now provides a few
convenient method (`getElementName()`, `resolveElements()`) to let other query
the element map in its associated `PropertyComplexGeoData`. After `GeoFeature`
detects any element name changes in its `onChanged()` method, it will signal all
its dependent objects to update their geometry element references.

## Property Link

`PropertyLinkSub`, `PropertyLinkSubList`, and `PropertyXLink` are the only
properties in the core that can carry geometry element reference information.
Other objects that use those properties are free to choose which type of element
name to store as reference, either the original or the mapped one. Behind the
scene, they will keep a shadow copy of both the original and mapped element
reference (if there is one) each time they are assigned a new reference. When
updating, if and only if there exists a shadow copy of the mapped element
reference, will it auto updates its original index based element reference. 

Let's take an example, say, a compound with a mapped reference `F1 -> Face1`.
When the user assigns the reference `Face1` to a `PropertyLinkSub`, it will
immediately query the element map, and stores a shadow copy of both `F1` and
`Face1`. When you updates a compound, say rearrange the order of its children,
its element name will surely change. Assume the compound has updated the mapping
for `F1` to `Face10`. When `PropertyLinkSub` is asked to update its reference,
it will query the current element mapping for `F1`, and obtained the updated
reference `Face10`, and then update the user assigned value from `Face1` to
`Face10` automatically.

The reason why we keeps a shadow copy of the unmapped reference is because the
user is free to choose to store the new mapped name as well. Say, it stores `F1`
in the above example, then no user visible update will happen. But
`PropertyLinkSub` will still keep the shadow copy of the unmapped reference up
to date behind the scene. When saving, `PropertyLinkSub` will store this
up-to-date unmapped reference just like before, so that the old version FreeCAD
can still open the document, and every links inside works as expected.
`PropertyLinkSub` will also save the user assigned value, if and only if it is
a new mapped name, so that when restoring in the new FreeCAD, it stays the same.
The old version FreeCAD will simply skip it.

You may be wandering why can't we just hide the whole mapped element reference
name behind the scene, and only assign the original element name to
`PropertyLinkSub`, and let the core works its magic to keep it up-to-date. 
There are cases where you'll prefer a constant reference name. For example,
The new [sketch export](Release-Notes-0.3#user-content-export) feature allows you to
export any _private_ geometry element of a sketch, including the external
edges. Sketch will use the new mapped reference name for linking to external
edges, and then use the SHA1 hash of the link reference as the element name for
export. The stability of the mapped name ensures the stability of the export,
as well.

## GUI Selection

When you call `Gui.Selection.getSelectionEx()`, it will return the geometry
element selection in the `SubElementNames` attribute of its returned value. My
`Link` branch FreeCAD went through major upgrade to bring you full object
hierarchy information through this `SubElementName` field, as a `.` separated
string. For example, when you selects a face of a `PartDesign` `Pad` feature,
`SubElementNames` will give you something like, `Pad.Face1`. Now, it is further
extended to include any mapped element name found, as well, like, `Pad.;F1.Face`.
The `;` is used here to mark the following name as a mapped element name instead
of an object name. The mapped element name is not obtained by `Gui.Selection`
directly, but by the object's view object. Just like the current FreeCAD relying
on the view object's `getElement()` function to return the selected element name
`Face1`, it now returns `;F1.Face1` if the mapping is found. As a result, when
you hover your mouse over some geometry element in the 3D view, you will now see
the new mapped element reference in status bar.

By default, calling `Gui.Selection.getSelectionEx()` will automatically walk
down the selected object hierarchy, and resolve the final object, and its
selected geometry element index based reference, so that any existing workbench
sees exactly what they expect. New code can pass an additional parameter
`resolve`, set to `2` to see the mapped reference name, or `0` to see the
entire object hierarchy. That's how a `PropertyLinkSub` can be assigned
a mapped geometry reference.

# Topological Name Generation

It may seems that we have yet to enter the main topic, considering the fact that
the title of this article is called _Topological Naming_, and we haven't
actually talked about it. However, let me assure you the utter importance of the
ground works described above. Because, only when you actually tries to implement
a full working solution, will you realize that it is now difficult to implement
some seemingly straight forward operations, like copy and paste. Because,
ironically, in order to have a stable topological naming, it has to include
object identification. But copy and paste generates new objects, with new
identification, and thus, will completely change all existing topological
naming. In other word, it is useless to copy topological names. We have to rely
on old style `type`+`index` to re-generate all mappings, and re-establish all
property links. Without the framework above, it just won't work. 

Now, we are going to talk about the actual topological name generation, which
I consider to be fairly standard, most of which follows some of the research
papers out there.

## Object Identification

Each document object in FreeCAD will now be assigned an integer identification
number that is unique within its owner document. UUID is not really useful here,
when objects can be freely copied and pasted from other documents, which may be
a copy by themselves. There is no uniqueness guarantee outside of the object's
document. A simple integer is simpler, cheaper, much easier to manipulate.

`Part.TopoShape` has a new public member variable called `Tag` that is used to
link a shape to its owner object. When the shape is generated by code, the user
is free to assign any integer value to `Tag`. The rule is that, zero `Tag` disables
automatic element mapping. When mapping an element from another shape with a
different Tag, it will append the Tag into the mapped element name.

## Element Mapping

`TopoShape` now offers a helper method to map the same geometry element from
other `TopoShape` instance into itself.

`mapSubElement(type, shape, op=None, mapAll=False)`

`type` will be either `Face`, `Edge`, or `Vertex`. `shape` is the other shape to
map from. `op` is an optional string to insert before the mapped element name.
`mapAll` indicates whether you want to map all other lower level elements, e.g.
for `Edge`, the lower level element type will be `Vertex`. If `shape` has its
own mapping, then `mapSubElement` will use the mapped name.

In Python, `TopoShape`'s sub-shape attributes will all use `mapSubElement` to
transfer the element mapping from the owner shape to its sub-shapes. For
example, run the following code in FreeCAD Python console,

```python
import Part,pprint
cube = Part.makeBox(10,10,10)
cube.Tag = 1
cube.ElementMap
{}

pprint.pprint(cube.Face2.ElementMap)
{'Edge5': 'Edge1',
 'Edge6': 'Edge2',
 'Edge7': 'Edge3',
 'Edge8': 'Edge4',
 'Face2': 'Face1',
 'Vertex5': 'Vertex1',
 'Vertex6': 'Vertex2',
 'Vertex7': 'Vertex3',
 'Vertex8': 'Vertex4'}
```

As you can see, cube itself does not have any element mapping, because it is a
primitive shape, and thus, all its element can be considered as fixed. To
enable sub-shape element mapping, set the `Tag` to non-zero, and you can see how
sub-shape `cube.Face2` maps the original cube's element name into its own
elements. In the returned dictionary, the key is cube's original element name,
and the value is the sub-shape's element name. The element mapping will be
generated by all sub-shape accessors, like `Solids`, `Wires`, etc.

Although the `Part` primitive shape does not have element map by default, the
user (or Python code) can assign customized names that are persistent, so can
the generic `Part::Feature` object. Be careful though, if you change the element
name of a feature, all its dependent feature may change their element name, too,
which may invalidates the element reference inside some link properties. You
should choose a fixed rule to generate element names in your derived classes.

```python
import Part
doc = App.newDocument('test')
cube = doc.addObject('Part::Box','cube')
shape = cube.Shape.copy()
shape.setElementName('Face1','F1')
cube.Shape = shape
doc.recompute()
doc.save('test.fcstd')
App.closeDocument('test')
doc = App.openDocument('test.fcstd')
doc.getObject('cube').Shape.ElementMap
{'F1': 'Face1'}
```

## New Makers in `TopoShape`

`TopoShape` offers several new `maker` method that utilize this element
mapping. In C++, all of the new `maker` function name starts with `makE`, such
as `makECompound`, `makEWires`, `makEFace`, etc. All new maker code is
implemented in a new source file called `TopoShapeEx.cpp`. In Python, however,
the changes are made by change the C++ code of existing method, so that
existing Python feature can get the benefit without any python code
modification.

There two categories of maker functions. The first category of makers do not
modify any existing elements. Instead, it generates new elements by simply
combining input lower level elements. They are,

* `makECompound`, exposed to Python using both `Part.makeCompound` and 
  `Part.Compound` constructor.
* `makEWires`, exposed to Python as a new method in `Part.Shape` named 
  `makeWire`, and also as `Part.Wire` constructor. This function accepts 
  list or compound of unsorted edges. And will automatically connects
  the edges to form one or more wires (as compound)
* `makEFace`, exposed to Python through `Part.makeFace`, and as the
  `Part.Face` constructor.
* `makECompSolid`, exposed to Python as `Part.CompSolid` constructor

All of these method, except `makEFace`, do not really generate new element
names, but only map existing element names, although, it may append the tags of
the input shape to avoid name conflict. `makEFace` will name the new face, if
and only if there is more than one face generated, by combining the edge names
of the outer wire of the face.

The following code demonstrate how `makECompound` maps the input shape elements
into the new compound shape. The prefix `CMP` is the default op code of compound
making operation.

```python
import Part,pprint
cube = Part.makeBox(10,10,10)
cube.Tag = 1
cube2 = Part.makeBox(10,10,10,App.Vector(10,0,0))
cube2.Tag = 2
compound = Part.makeCompound([cube,cube2])

pprint.pprint(compound.ElementMap)
{'CMP1_Edge1': 'Edge1',
 'CMP1_Edge10': 'Edge10',
 'CMP1_Edge11': 'Edge11',

...

 'CMP2_Edge1': 'Edge13',
 'CMP2_Edge10': 'Edge22',
 'CMP2_Edge11': 'Edge23',
...

}
```

The following code demonstrates `makEFace` operation. We are making two faces at
the same time in order to show you how face names are generated. You can see
that the face names only contains the edge names of the outer wire.

```python
points = [
App.Vector(10,-10,0),
App.Vector(-10,-10,0),
App.Vector(-10,10,0),
App.Vector(10,10,0),
App.Vector(10,-10,0) ]

outer1 = Part.makePolygon(points)
outer1.Tag=1
inner1 = Part.makeCircle(5)
inner1.Tag=2

outer2 = outer1.copy()
outer2.Tag=3
outer2.Placement.translate(App.Vector(-40))
inner2 = inner1.copy()
inner2.Tag=4
inner2.Placement.translate(App.Vector(-40))

face = Part.makeFace([outer1,outer2,inner1,inner2])

pprint.pprint(face.ElementMap)

{'FAC(FAC1_Edge1,FAC1_Edge2,FAC1_Edge3,FAC1_Edge4)': 'Face2',
 'FAC(FAC3_Edge1,FAC3_Edge2,FAC3_Edge3,FAC3_Edge4)': 'Face1',
 'FAC1_Edge1': 'Edge6',
 'FAC1_Edge2': 'Edge7',
 'FAC1_Edge3': 'Edge8',

 ...

 'FAC3_Vertex4': 'Vertex4',
 'FAC4_Edge1': 'Edge5',
 'FAC4_Vertex1': 'Vertex5'}
}

```

The Sketcher now uses `TopoShape::makEWires` to create its public geometry
wires and map its internal geometry element names. Too see the effect, create
any sketch, and type the following command in the console.

```python
pprint.pprint(App.ActiveDocument.Sketch.Shape.ElementMap)

{'SKT_g1': 'Edge1',
 'SKT_g1v1': 'Vertex1',
 'SKT_g1v2': 'Vertex2',

 ...

}
```

The second category of maker relies on `OCCT`'s 
[BRepBuilderAPI_MakeShape](https://www.opencascade.com/doc/occt-7.0.0/refman/html/class_b_rep_builder_a_p_i___make_shape.html)
and its derivatives to construct the shape. And `TopoShape` relies on this
class to provide the shape history information, specifically, the mapping from
the input geometry elements to the newly generated or modified elements in the
resulting shape. The details of the mapping algorithm can be found [[here|Topological-Naming-Algorithm]]


## String Hasher

As shown in the section above, `makESHAPE()` is going to generate some very long
names. And as we add more histories to the model, the length of a new element name will
quickly go out of control. To deal with this problem, a new object is introduced,
`StringHasher`. It allows you to transform any string into an integer ID.
`SthringHasher.Threshold` controls how the strings are stored. If `Threshold` is
positive, and the string length goes beyond this value, the original string
content will be discarded, and only persists the crypto hash of the string,
currently SHA1. If `Threshold` is non-positive, then it will keep the string as
it is. The default `Threshold` is 40. The returned integer ID is guaranteed
to be unique within its own `StringHasher` object.

Every `Document` now has a default `StringHasher` object accessible through
`Document.Hasher`. `ComplexGeoData` also has an attribute named `Hasher` which
is initially `None`. Simply assign the `Document.Hasher` to it, or, if you want,
create a new one by `App.StringHasher()`. `ComplexGeoData.setElementName()`
will encode the mapped element name if there is a `Hasher` available.

`TopoShape` inherits the `Hasher` from `ComplexGeoData`. When you assign
a `TopoShape` containing a `Hasher` to a `PropertyPartShape`, the hasher will be
persisted to the document. The string inside the `Hasher` is reference counted.
By default, only used string will be persisted. You can change that behavior by
setting attribute `StringHasher.SaveAll` to `True`.

To see the effect of the string hasher,

```python
import Part,pprint
cube = Part.makeBox(10,10,10)
cube.Tag = 1
cube.Hasher = App.StringHasher()
cube.Hasher.Threshold=10

cube.setElementName('Edge1','A_VERY_LONG_ELEMENT_NAME')
'#1'

cube.setElementName('Edge2','ShortName')

pprint.pprint(cube.ElementMap)
{'#E1': 'Edge1', '#E2': 'Edge2'}

pprint.pprint(cube.Hasher.Table)
{1: '3627a73b7bd8c90ef07fefde34757e34dc14c1b9', 2: 'ShortName'}

compound = Part.makeCompound(cube)

pprint.pprint(compound.ElementMap)
{'CMP1_#E1': 'Edge1',
 'CMP1_#E2': 'Edge2',
 'CMP1_Edge10': 'Edge10',

...

}

```

As you can see, the first element name exceeds the threshold, so `Hasher.Table`
stores the SHA1 hash of the assigned element name. To use the `StringHasher`
more effectively, you should use `StringHasher.getID()` for both string lookup
or ID creation. The returned value is a `StringID` object that is reference
counted. 

```python
hasher = App.StringHasher()
id = hasher.getID("abcde")

print str(id)
'#1'

print id.Value
1

print id.Data
abcde

id2 = hasher.getID(1)

print id.isSame(id2)
True

```

Each document now has a new property named `UseHasher` to control be hashing
behavior. `Part` workbench features will check this property to decide
whether to hash the generated element names. It is by default on. When you
manually set this property to `False`, the document will touch every geometry
feature inside, and you'll need to recompute the entire document to get the
updated element map.

There is one more important details that is handled inside `ComplexGeoData`
and `TopoShape`. Most of shape building involves multiple steps. It is not
possible to only hash the element names at the very last step, i.e right
before the final shape is stored into a `PropertyPartShape`, because there
is no easy way to preserve prefix and/or postfix, which are important for
querying related geometry elements (see [[here|Topological-Naming-Algorithm#user-content-fillet]]).
However, if we hash element names in the intermediate steps, where the
shapes produced are not persisted, it is possible to have lost the string
ID when restoring the document, causing unwanted element name change. To
prevent this, the element map inside `ComplexGeoData` will keep an array
of very historically used string ID references for each hashed names.
And the arrays are persisted along with the element map.

## Element Map Versioning

As you can see from the description so far, the element name generation is
a very delicate process. It can be affected by many factors, such as whether
the user turns on the hasher, the hasher threshold, how the feature assigns the
primary element name, the `TopoShape` name generating algorithm, and of course,
`OCCT's` maker's mapping logic that `TopoShape` relies on to obtain the shape
history. Any change of these factors may result in significant changes in the
generated element names, and can potentially break existing element references.

To prepare for this possibility, we add a new API to `GeoFeature` to report its
element map version. It returns a string that is persistent. Derived features
can choose to override it the store its own version. In fact, it is __very important__
for any geometry feature to change the version number if there is any change in
its shape building logic. The default implementation will return something as
below,

```
1.4.70100.1.40
| |   |   | |
| |   |   | --If use hasher, then this is the hasher threshold
| |   |   |
| |   |   --ComplexGeoData algorithm version, now is 1
| |   | 
| |   --OCCT version
| |   
| --TopoShape algorithm version, now is 4
|
--1 means using the same hasher as the document's, or else 0
```

Any geometry feature can simply append their own version (if any) behind this.

When a document is restored, `PropertyPartShape` will compare the persisted
version string with the current one, and mark the feature for recompute
if it is changed. It will also inform any link property that has element 
reference to this feature to reset its shadow copies, similar to what 
happens when an geometry feature object is copied and pasted.

