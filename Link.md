This article explains the motivation behind the creation of the `Link` object,
along with the design and implementation. All necessary FreeCAD core changes
are described [here](Core-Changes)

# Motivation

`Link` is primarily designed for working with assembly structure. There are two
major requirements. 

Firstly, we need a container object to hold all the child parts of an assembly.
And more importantly, the container must provide a coordinate system for
placing its child parts, which allows the assembly to be moved as a whole, or
be nested in other parent assemblies. 

Secondly, we need an efficient way of handling multiple instances of an object,
such as a screw, a pin, or more importantly, a sub-assembly. The instance
should be able to move freely without interfering with each other, and be
selected without ambiguity.

There are also minor requirements, such as be able to override the colors of
some specific instance of an object, or even a sub-object of some sub-assembly.

This document describes the design and implementation of `Link` that fulfills
all the requirements mentioned above.

# Design

`Link` is a FreeCAD `DocumentObject`, not to be confused with `PropertyLink`
which is a property belonged to some object. `Link` does not contain any
geometry data, nor any visual representations. If you think of FreeCAD tree
view as a file structure, then `Link` is like `symlink` on Linux, and
`shortcut` on Windows. It merely references the object being linked. `Link`
provide its own `Placement` and `Visibility` property that overrides those from
the linked object, which gives the linked object a seemingly new identity,
which can be moved and hid/shown independently. `Link` is also capable of
overriding either the whole shape color of the linked object, or the color of
individual geometry element, such as face, edge or vertex.

The greatest power of `Link` comes when it is used to link to groups, which may
contain other links in itself. This greatly improves modeling efficiency, and
allows FreeCAD to handle massive nested assembly structures that are not
previously possible. 

`Link` has several working mode, as listed below

* When used alone, it exists in FreeCAD as `App::Link`. It is not only capable
  of linking to individual object or group, but also sub object (e.g. a child
  object of a group), and sub geometry element (e.g. a face).

* `App::Link` has a special _array mode_ that makes easy to multiply the linked
  object with minimum overhead. The array can either be expanded with each
  individual instance exposed in FreeCAD as objects shown in the tree view,
  which can be moved and hid/shown individually, or collapsed to offer
  maximum efficiency by not exposing each instance as FreeCAD objects.

* `Link` can also operate in container mode, which exists in FreeCAD as
  `App::LinkGroup`. `LinkGroup` creates internal `Links` to added objects. The
  added objects has independent visibility settings. The user can choose to
  either override the object's placement, by asking `LinkGroup` to
  automatically create an `App::Link` for each object added to it, Or opt to
  keep using the object's placement, and make the `LinkGroup` act like a normal
  container. 

* The programmer can create their own `Link` type object with great flexibility
  using `App::LinkBaseExtension`, and `Gui::ViewProviderLink`.

<a name="selection-context"></a>The biggest challenge of making `Link` work is
to solve the selection ambiguity problem. Because with `Link`, the  same object
can now appear in many different hierarchy, e.g. with a `Link` that links to
a group, the child object now appears in two different coordinate system.
Saying some object is selected may now be ambiguous. We must include the
hierarchical information of the selected object. We shall call the hierarchy of
a specific selection its _Selection Context_.

<a name="suname"></a>To introduce the _Selection Context_ concept in a backward
compatible way, we choose to extend the meaning of an existing attribute in
FreeCAD selection. The `SubName` is originally used to carry non-object
sub-element reference of a selection, such as a geometry element reference like
face, edge or vertex. It is now extended to be able to carry object hierarchy
information as well. The selected object recorded by FreeCAD is now changed to
the top level parent object. And the hierarchical path of the actual selected
object is stored in `SubName` as a dot separated names of intermediate objects.
`SubName` can still carry geometry element reference. To distinguish the
element name from object name, we demand that the object name reference in each
hierarchy must always end with a dot, and the geometry element reference, if
there is one, should be the last name without the ending dot. So, for example,
to reference an object named _Face_ inside some group, the `SubName` will be
`'group.Face.'`, and to reference a geometry face of that object,
`'group.Face.Face1'`. There are various other extensions to `SubName` reference
which gives special meaning to the name referenced in a hierarchy depending on
the first character of the name,

* `$` marks the following name as the label of the referenced object. In other
  words, the object of this hierarchy is referenced by its label. The label
  matching scope is limited to the child objects of the upper hierarchy.

* `;` marks the name as the mapped geometry element name generated
  by the new [Topological Naming](Topological-Naming#the-framework).


# Implementation

Like most core objects in FreeCAD, the implementation of `Link` is divided into
two parts, one exists in the `App` name space for handling non-visual logic,
and the other in the `Gui` space for visual representation. 

## `App` Namespace

In `App` namespace, the core API that makes `Link` work is
[DocumentObject::getSubObject()](Core-Changes#user-content-getsubobject), which
is used to obtain the object referenced by a given `SubName`, along with the
accumulated transformation matrix and the Python binding object, without
knowing the exact type of the object. To make it easy for other object to use
link to obtain shared data, it is recommended to implement helper function in
respective modules, such as the [getShap()](Core-Changes#user-content-getshape)
function from the `Part` module.

The core logic of `Link` in `App` namespace is implemented as an `Extension`
called `App::LinkBaseExtension`, which can be dynamically installed into other
object to add the `Link` related logic. The best example will be the
`Assembly3` workbench, with most of its document object implemented that way.

`App::LinkBaseExtension` does not contain any property of its own, but relies 
on the host object to provide the properties. This allow the property to have
custom names. `App::LinkBaseExtension` can behave differently depending on which
properties are provided, and also on the type of the provided properties. For
example, object supplies a [PropertyXLink](Core-Changes#user-content-xlink) is
capable of linking to a sub-object or an external object, while those with
`PropertyLink` cannot. Object can choose to supply a `Placement` to make the
`Link` movable, or choose not to, and make the object always follow the linked
object's placement, as the case of the constraining element link in
`Assembly3`.

C++ code calls the `LinkBaseExtension::setProperty()` function to supply
the properties, while Python code calls `configLinkProperty(key=val...)`, where
`key` is predefined text names of the property, and `val` is the actual
property name provided by the object. `configLinkProperty()` accepts multiple
properties. If `val` is omitted, then the property name is assumed to be the
same as `key`. Python code can also call `getLinkExtProperty(name)` function to
obtain the value of a configured property, where `name` is the predefined `key`
name of the property. Each property supplied must be of a pre-defined type,
or its derived type. Currently supported properties are,

| Key name | Property type | Description |
| -------- | ------------- | ----------- |
| LinkPlacement | App::PropertyPlacement | The placement of the `Link` object |
| Placement | App::PropertyPlacement | Alias to LinkPlacement to make the `Link` object compatible with other objects |
| LinkedObject | App::PropertyLink | Property to hold the linked object |
| SubElements | App::PropertyStringList | To hold a list of non-object geometry element references of the linked object, e.g. Face1. See [here](#user-content-partial-render) |
| <a name="link-transform"></a>LinkTransform | App::PropertyBool | Default false. Set to true to follow the linked object's placement. If this property is not supplied, and neither does any placement properties, then the link will also follow the linked object's placement. |
| Scale | App::PropertyFloat | Scale factor |
| ScaleVector | App::PropertyVector | Scale factors, in case non-uniform scaling is desired. |
| PlacementList | App::PropertyPlacementList | The placement for each link element in array mode |
| ScaleList | App::PropertyVectorList | The scale factors for each link element in array mode |
| VisibilityList | App::PropertyBoolList | The visibility state of each link element in array or group mode |
| ElementCount | App::PropertyInteger | Link element count. Set to non-zero to activate array mode |
| ElementList | App::PropertyLinkList | The property for holding child object list in array or group mode |
| ShowElement | App::PropertyBool | Set to true to show element as objects, and false to collpase the array and claim the linked object a the single child. |
| LinkMode | App::PropertyEnumeration | Link group mode, `AutoDelete`: auto delete the child object when removed. `AutoLink`: auto create a link for the dropped object. `AutoUnlink`: like `AutoLink` and also auto delete it when removed. |
| ColoredElements | App::PropertyLinkSubHidden | The property for holding color overridding elements |

`App::Link` and `App::LinkGroup` are implemented by supplying a different set
of properties to activate relevant behaviors in `App::LinkBaseExtension`. There
is another `App::LinkExtension` provided for convenience, which supplies part
of the required properties, including `Scale`, `ScaleList`, `ElementList`,
`VisibilityList` and `PlacementList`. Object using this extension can provide
other property to further customize its behavior. For example, if the object
provides a `LinkedObject` property, then it will behave like a `Link`,
otherwise, it behaves like `LinkGroup`.

<a name="python-link"></a>The following code shows how to make a Python link
that also supports link array,

```python
class MyLink(object):
    def __init__(self):
        self.Object = None

    def __getstate__(self):
        return

    def __setstate__(self,_state):
        return

    def attach(self,obj):
        'new Python API called when the object is newly created'

        # the actual link property with a customized name
        obj.addProperty("App::PropertyLink","MyLink"," Link",'')

        # the placement with the default name
        obj.addProperty("App::PropertyPlacement","Placement"," Link",'')

        # the following two properties are required to support link array
        obj.addProperty("App::PropertyBool","ShowElement"," Link",'')
        obj.addProperty("App::PropertyInteger","ElementCount"," Link",'')

        # install the actual extension
        obj.addExtension('App::LinkExtensionPython', None)

        # initial the extension
        self.linkSetup(obj)

    def linkSetup(self,obj):
        'helper function for both initialization (attach()) and restore (onDocumentRestored())'

        assert getattr(obj,'Proxy',None)==self
        self.Object = obj

        # Tell LinkExtension which additional properties are available.
        # This information is not persistent, so the following function must 
        # be called by at both attach(), and restore()
        obj.configLinkProperty('ShowElement','ElementCount','Placement', LinkedObject='MyLink')

    def getViewProviderName(self,_obj):
        'new Python API for overriding C++ view provider of the binding object'

        return 'Gui::ViewProviderLinkPython'

    def onDocumentRestored(self, obj):
        'Python API called on document restore'

        self.linkSetup(obj)

class ViewProviderMyLink(object):
    def __init__(self,vobj):
        vobj.Proxy = self
        self.attach(vobj)

    def attach(self,vobj):
        self.ViewObject = vobj

    def __getstate__(self):
        return None

    def __setstate__(self, _state):
        return None


def makeMyLink(obj):
    'make a link to the given object'

    # addObject() API is extended to accpet extra parameters in order to 
    # let the python object override the type of C++ view provider
    link = obj.Document.addObject("App::FeaturePython",'link',MyLink(),None,True)

    ViewProviderMyLink(link.ViewObject)

    link.setLink(obj)
    return link
```

Use the following code to create a link array of a box, and customize the color
of each instance. Note that by default `ShowElement` is `False`, which means it
is a collapsed link array, showing only one single child. Change it to `True`
to see each element as an object. Once _expanded_, you can change each
element's placement and color by directly modifying the corresponding child
object in the property view.

```python
doc = App.newDocument('test')
box = doc.addObject('Part::Box','box')
box.ViewObject.ShapeColor = (0.8,0.8,0.8)

# create the link object
link = makeMyLink(box)

# Set element count to switch to collpased link array.
# If you want to see the element as object, set ShowElement=True
link.ElementCount=4

# Set each element placement
link.PlacementList = (
    App.Placement(App.Vector(15,0,0),App.Rotation()),
    App.Placement(App.Vector(-15,0,0),App.Rotation()),
    App.Placement(App.Vector(0,15,0),App.Rotation()),
    App.Placement(App.Vector(0,-15,0),App.Rotation()))

# Set each element color
materials = [App.Material(),App.Material(),App.Material(),App.Material()]
materials[0].DiffuseColor = (1.0,0.0,0.0)
materials[1].DiffuseColor = (0.0,1.0,0.0)
materials[2].DiffuseColor = (0.0,0.0,1.0)
materials[3].DiffuseColor = (0.0,1.0,1.0)
link.ViewObject.MaterialList = materials

# Turn on material override
link.ViewObject.OverrideMaterialList = [True]*4

doc.recompute()
```

## `Gui` Namespace

FreeCAD uses `Coin3D` library for 3D visual representation and rendering.
`Coin3D` represents geometry information using tree of nodes, such as
coordinate transformation node, material node, shape node, etc. See
[here](https://www.freecadweb.org/wiki/Scenegraph) for a brief introduction.
`Coin3D` supports node sharing, meaning that the same node can be added to
different trees to be rendered at a different location and/or with a different
material. And this is the foundation of how `Link` can share the visual
representation of the linked object. The upstream FreeCAD cannot handle node
sharing, because its selection framework cannot distinguish among the same
nodes in different context. The aforementioned 
[Selection Context](#user-content-selection-context) is specifically designed
to resolve this problem.

`Gui::ViewProviderLink` is designed as a view provider to work with any type of
object that is installed with `App::LinkBaseExtension`. However, this class
does not manipulate `Coin3D` nodes directly, but by a member of class type
`Gui::LinkView`, exposed to Python by `LinkViewPy`. The separation of logic
is done in the hope that one day, `LinkView` may find its usage in other
non-document-object related application, for example, more efficient animation
of an object by directly manipulating a snapshot of the object's node tree,
without affecting any of the object's own nodes or property. 

<a name="link-node"></a>To share visual representation, we can not simply
insert the node tree of the linked object as it is, because we may want to have
separated visibility and coordinate transformation. Depending on the type of
linked object and property setting of `App::LinkBaseExtension`, there are three
variations of linking (corresponding to `LinkView::SnapshotType`), each
requiring a slightly different way of cloning the node tree of the linked
object. 

* `SnapshotTransform`, support independent visibility control by replacing the
  switch node, and removing the transformation node of the linked object in the
  snapshot;

* `SnapshotVisible`, with independent visibility control, but linked
  transformation, which is turned on by setting the 
  [LinkTransform](#user-content-link-transform) property;

* `SnapshotChild`, link both visibility and transformation, by keeping the node
  tree as it, and only replacing the root node. This type of linking is only
  used to indirectly link to the children of an `App::Part` group.

A private class `Gui::LinkInfo` is used to maintain those modified snapshots of
the node tree, and update them when they change. `LinkInfo` monitors the linked
object for changes by installing an extension, `Gui::ViewProviderObserver` into
the linked view provider. `LinkInfo` ensures that each object has at most three
snapshots (one for each linking type) no matter how many times it is linked, in
order to save memory resources and improve update efficiency.

A new class `Gui::SoFCSelectionRoot`, derived from `Coin3D SoSeparator`, is added
both as the storage, and also as the key(s) for looking up selection context.
When an object is linked, `LinkView` will request a snapshot of the node tree
from `LinkInfo`, and use a `SoFCSelectionRoot` to hold the snapshot tree.

<a name="rendering"></a>Various shape nodes are extended to support selection
context, including `PartGui::SoBrepPointSet`, `PartGui::SoBrepEdgeSet`, and
`PartGui::SoBrepFaceSet`, as well as non-shape node `Gui::SoFCSelection` (for
rendering selection highlight of non shape object). When selection action is
traversed down the node tree hierarchy, `SoFCSelectionRoot::doAction()` will
maintain a stack containing every encountered instance of `SoFCSelectionRoot`.
When the action reaches a shape node, the shape node will call
`SoFCSelectionRoot::getActionContext()` to obtain a context for storing the
selected shape element index. The context itself is stored in a map owned by
the top-level `SoFCSelectionRoot` node, and keyed using the current stack of
(i.e. all upper-level) `SoFCSelectionRoot` nodes plus the shape node itself.
When rendering, `SoFCSelectionRoot::GLRender()` will build the same stack while
traversing, so that the shape node can call
`SoFCSelectionRoot::getRenderContext()` to obtain the correct selection context
and render the relevant selection highlight.

Each context is captured by an abstract class `Gui::SoFCSelectionContextBase`,
which is overridden to store customized information for each shape nodes, such
as index of the selected geometry element.

## Secondary Context

<a name="partial-render"></a>In addition to the selection context described
above, `SoFCSelectionRoot` also supports a secondary context, which is used for
customizing visibility, material, and even partial rendering, that is,
rendering only some element (e.g. face) of a shape. The visual style override
function is similar to the _Assembly Component Instance Styling_ described in
this STEP [recommended practices](https://www.cax-if.org/documents/rec_prac_styling_org_v15.pdf),
section 5.

Unlike selection context, which stores the context in the first visited
`SoFCSelectionRoot` node during traversing, the secondary context is stored
in the last one. The secondary context is established using the extended
`SoSelectionElementAction`. But instead of always traversing down from the top
level hierarchy of the scene graph as in normal selection, the action for
secondary context starts traversing from the root node of a `Link` object, so
that the overridden style only applies to (sub)object(s) below this `Link`, and
any higher level `Link`. Other unrelated `Link` to the same (sub)object(s) will
not be affected. See the following code and screenshot for an illustration.

```python
# create a grey box

doc = App.newDocument('test')
box = doc.addObject('Part::Box','box')
box.ViewObject.ShapeColor = (0.8,0.8,0.8)
box.recompute()

# create a link to box, override color to blue

link1 = doc.addObject('App::Link','link1')
link1.setLink(box)
link1.ViewObject.ShapeMaterial.DiffuseColor = (0.0,0.0,1.0)
link1.ViewObject.OverrideMaterial = True
link1.Placement.Base.x += 15

# create a higher level link to the previous link1, setup partial rendering of
# only face1 and face2, and override face1 color with transparent red. You can
# see that for face2 the higher level link inherits the lower level link color
# override.

link2 = doc.addObject('App::Link','link2')
link2.setLink(link1, '', ['Face1','Face2'])
link2.ViewObject.setElementColors({'Face1':(1.0,0.0,0.0,0.5)})
link2.Placement.Base.x -= 15

# create an even higher level link to the previous link2, override its face1
# color again to transparent green

link3 = doc.addObject('App::Link','link3')
link3.setLink(link2)
link3.ViewObject.setElementColors({'Face1':(0.0,1.0,0.0,0.5)})
link3.Placement.Base.y -= 15

# create another link to box, and you can see the color override in other
# unrelated link has no effect here.

link4 = doc.addObject('App::Link','link4')
link4.setLink(box)
link4.Placement.Base.y += 15
```

[[images/secondary-context.png]]

Note that `link3` above demonstrate `Link's` capability of overriding the color
or an already overridden element by the lower level `link2`. This is because
`SoFCSelectionContext::getRenderContext()` will enumerate all secondary context
along the rendering path, and merge them together before returning to shape
node caller. 

## Selection Logic Flow

The core stores the selection in class `Gui::SelectionSingleton` (exposed as
`Gui.Selection` in Python), which is also responsible for notifying of 
selection change to other part of the program. The class is
[modified](Core-Changes#selectionsingleton) to be aware of the `SubName`
reference, and provides API that either can auto resolve the actual selected
object referred by the `SubName` for backward compatibility, or provider full
object hierarchy information for newer code that requires it.

`SoFCUnifiedSelection` is the class responsible of handling `Coin3D` selection
events. There are two sources of selection,

### Selection in 3D View

When the user selects an object in the 3D view, the logic flows like this,

* `SoFCUnifiedSelection` receives `Coin3D` mouse event in its `handleEvent()`
  function.

* For each picked point returned by `Coin3D`, `SoFCUnifiedSelection` will
  lookup the top level view provider using the node path of the picked point,
  and call `ViewProvider::getElementPicked()` to translate the selected path
  to a `SubName` reference. 

* `SoFCUnifiedSelection` will add the first picked object, along with its
  `SubName` reference, to `Gui.Selection`. If enabled, it will also populate
  `Gui.Selection.PickedList` with the obtained `SubName` references from all
  picked points.

* Once a new selection is added (by the above step), `Gui.Selection` will
  broadcast a message of selection change.

* `View3DInventorViewer` receives the message in its `onSelectionChanged()`,
  encapsulate the selection change message in `SoFCSelectionAction` and
  applies the action to its scene graph.

* `SoUnifiedSelection`, being the root node of the scene graph, handles the
  action, and looks up the view provider of the top parent of the selected
  object, and calls its `ViewProvider::getDetailPath()` to translate the
  `SubName` reference to `Coin3D` path, and obtain the shape element `SoDetail`

* `SoUnfifiedSelection` sends a `SoSelectionElementAction` down the path
  obtained above,

  * For element selection, such as a face, `getDetailPath`() will return a path
    all the way to the shape node that owns the element. Therefore, the
    selection action message is handled by that shape node's `doAction()`
    function, which will call `SoFCSelectionRoot::getActionContext()` to obtain
    the selection context, and stores the selected element index inside.

  * For whole object selection, the path returned by `getDetailPath()`
    terminates at the root node of the view provider of the selected object.
    Normally, if an action is applied to a path, it is traversed by following
    the path with `SoAction::PathCode::IN_PATH` first. Once reached the end of
    the path, it will change to `SoAction::PathCode::BELOW_PATH`, which means
    broadcasting the action to reach all (grand)children nodes of the node at
    the end of the path. This is very ineffective for selection of large
    hierarchies. An optimization is implemented such that the first encountered
    `SoFCSelectionRoot` node when traversing `BELOW_PATH` will stop further
    downward traversing by calling `SoAction::setHandled()`. And that
    `SoFCSelectionRoot` node will obtain a selection context of its own to
    record the state of whole object (and all its sub-objects) selection. Each
    shape node's `GLRender()` function will check for this state by calling
    `SoFCSelectionRoot::checkGlobal()`.

### Selection in Tree View

When the user selects an object in the tree view, the logic flows in
a slightly different path.

* Tree view receives user selection signal through Qt.

* Tree view finds the object corresponding to the selected tree item, walks
  upwards the tree hierarchy to find the top parent object, and then translates
  the hierarchy into a `SubName` reference.

* Tree view adds the top parent object and the `SubName` reference to
  `Gui.Selection`

* `Gui.Selection` broadcasts a message of selection change.

* The reset of the logic flows just like selecting in 3D view.

## Render Caching

`Coin3D` implements several ways for rendering acceleration. One is using OpenGL
_Vertex Object Buffer_ (VBO) for shape rendering. The other is using OpenGL
display list for render caching. Display list essentially records the OpenGL
rendering calls including the involved vertex data for faster replay.
`SoSeparator` node is used to manager the render cache. Once a render cache is
established, the render action no longer needs to traverse below the caching
`SoSeparator` node. For scenegraph with large hierarchy, the most critical
factor that affects rendering speed is often the traversing time. 

`Coin3D` render cache tracks the traversing state for correct rendering and to
auto refresh in case of changes. The state is recorded inside cache using
class derived from `SoElement`, holding information such as, transformation
matrix, material, texture, etc. Each `SoSeparator` node can keep a number of
caches (by default two). During rendering, it will search the cache list
for a cache that match the current traversing state. If none is found, it will
decide on some heuristics (roughly based on cache hit versus miss frequencies),
to either discard some old cache and create a new one, or proceed normal
rendering traversal without cache. (See
[here](http://developer98.openinventor.com/content/303-optimizing-rendering)
and [here](http://developer98.openinventor.com/content/59-special-considerations-caching),
for more details).

Upstream FreeCAD has just begun supporting hierarchical group (`App::Part`).
The node tree of shape object (`PartGui::ViewProviderPartExt`) is not
optimized for hierarchical rendering. A typical node tree looks like this,

[[images/node-tree.png]]

As you can see, there are several `SoSeparator` nodes placed directly above the
shape node, plus another intermediate `SoSeparator` node for the current
display mode, `Flat Lines`, and finally the root node. By default, each
`SoSeparator` has auto cache turned on. `Coin3D` uses some heuristics to
determine whether to cache or not at each instance of `SoSeparator` node. It is
not easy to determine for sure exactly which node will hold the cache. If the
cache is at the `SoSeparator` above the shape object, then the caching barely
offers any acceleration for large hierarchical groups, because the render
action has to traverse all the way down, in which case, render using cache
offers just minimum acceleration comparing to render with `VBO` without cache.

The render caching becomes even more complex when `Link` is involved, because
the same shape node is reused at many different places. The auto cache inside
the `SoSeparator` nodes of the linked object may be turned off by the
heuristics, because it sees frequent state changes caused by rendering at
different location (i.e. changes in the accumulated transformation element),
and this negatively affects auto caching in upper hierarchy as well (which
may arguably be considered as a implementation flaw)

To fix the above problem, we turn off caching in all intermediate `SoSeparator`
nodes, and auto cache only at the root node of the view provider. As described
[above](#user-content-link-node), the snapshot used for linking requires to
replace some node of the original node tree, which means we need to replace the
root node as well. And this also means `Link` will no longer affect cache of
the linked object. We also need to turn off cache in the replaced root
`SoSeparator` in the snapshot, because it is expected to be rendered in
multiple locations. In `ViewProviderLink`, we turn on auto cache in those
`SoFCSelectionRoot` nodes that are the parent node of the snapshot tree, which
may itself contain other `SoFCSelectionRoot` with cache for lower hierarchy. We
leave `Coin3D` auto caching logic to decide exactly where to cache inside the
hierarchy, which shows satisfying result in practice. 

<a name="cache-setting"></a>A new option is added to give user some control of
the caching behavior. It is called _Render Cache_ in _Display_ preference page,
with the following options,

* Auto (default), let `Coin3D` decide as described above,

* Distributed, manually turn on cache for all view provider root node, and
  intermediate `SoFCSelectionRoot` node in `ViewProviderLink`.

* Centralized, manually turn off cache in all nodes of all view provider,
  and only cache at the scene graph root node. This offers the fastest 
  rendering speed, but slower response to any scene changes.

In order to track this user setting without traversing the whole scene
graph, a new class `SoFCSeparator` is added and used as the root node of all
view provider.

There is still room for improvement for render caching. Currently, `Link` will
disable render caching when it has any element color override or element
selection (which is essential the same as color override). This cache disable
only affects all instances of the linked object. It is necessary because
element color override is handled inside the shape node, rather than the cache
keeping `SoSeparator` nodes, which means that the recorded cache is unable to
track color override changes. Whole object selection and color override does
not have this negative effect, because they are handled inside
`SoFCSelectionRoot` by its overridden render function.
