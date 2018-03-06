Starting from version 0.3, support of stable topological naming of geometry
element has been added to my FreeCAD Link branch. Because Assembly3 is the main
test environment, the document is put here.

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

* `setElementName(element,name, overwrite=False)`. It is used to assign a new 
  `name` to an `element`, e.g. `setElementName('Edge1','e1')`. An element
  can have multiple names, but a name can only be associated to one element.
  You can overwrite existing mapping by the `overwrite ` parameter.

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

## New Maker in `TopoShape`

`TopoShape` offers two new `maker` method that utilize this element mapping,

* `makECompound([shape],appendTag=True,op=None)` 

```python
import Part,pprint
cube = Part.makeBox(10,10,10)
cube.Tag = 1
cube2 = Part.makeBox(10,10,10,App.Vector(10,0,0))
cube2.Tag = 2
compound = Part.Shape()
compound.Tag = 3
compound.makECompound([cube,cube2])

pprint.pprint(compound.ElementMap)
{'C1_Edge1': 'Edge1',
 'C1_Edge10': 'Edge10',
 'C1_Edge11': 'Edge11',

...

 'C2_Edge1': 'Edge13',
 'C2_Edge10': 'Edge22',
 'C2_Edge11': 'Edge23',
...
```

  As you can see, `makECompound` append the input shape's `Tag` into the
  generated element name. The `C` is its default `op` code. You can choose your
  own code. When you create a compound of compound, expect to see some thing like
  `C3_C1_Edge1`.

* `makEWires(shape,op=None)`. This method builds wires using the edges inside
  the input shape. The edges do not need to be sorted. The method will sort them
  and connect them into wires. If more than one wire is found, it will create
  a compound to hold them. To create wires from multiple shapes, you can call
  `makEWires(Part.Shape().makECompound([shapes]))`. The default `op` code is
  `W`. `Sketcher` now use this function to map its own internal geometry element
  names, with `op` code `S`. Create any sketch, and type the following command
  in the console.

```python
pprint.pprint(App.ActiveDocument.Sketch.Shape.ElementMap)

{'S_g1': 'Edge1',
 'S_g1v1': 'Vertex1',
 'S_g1v2': 'Vertex2',
 ...
```

In the future, expect to see more `maker` function like these to be added, such
as `makEFuse`, `makECut`, `makEBoolean`, etc.

## String Hasher

For method like `makEFuse`, instead of just map existing elements, it needs
a way to name new elements. The existing FreeCAD `ShapeHistory` class provides
an implementation for this. The naming scheme can be simply concatenating all
input shapes element name together. So, if `Face1` is generated by shape one's
`Face2` and shape two's `Face3`, then we shall name it as `F1_Face2_F2_Face3`.
If shape one and two also have its own mapping, say they are a compound like
above, then the new name may be `F1_C3_Face2_F2_C4_Face3`.

If it goes like that, then the length of a new element name will quickly go out
of control. To deal with this problem, a new object is introduced,
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
'H1'

pprint.pprint(cube.ElementMap)
{'H1': 'Edge1'}

pprint.pprint(cube.Hasher.Table)
{1: '3627a73b7bd8c90ef07fefde34757e34dc14c1b9'}

compound = Part.Shape()
compound.Tag = 3
compound.Hasher = cube.Hasher
compound.makECompound([cube])

pprint.pprint(compound.ElementMap)
{'H10': 'Edge3',
 'H11': 'Edge4',
 'H12': 'Edge5',
...

pprint.pprint(compound.Hasher.Table)
{1: '3627a73b7bd8c90ef07fefde34757e34dc14c1b9',
 2: 'C1_Face1',
 3: 'C1_Face2',
...

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
'H1'

print id.Value
1

print id.Data
abcde

id2 = hasher.getID(1)

print id.isSame(id2)
True

```



