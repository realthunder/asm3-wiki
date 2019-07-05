Here is a list of core changes sorted by C++ namespace, and classes. With only
very few exception, most of the core changes are exposed to Python as well.

# `App` Namespace

## `Property`

The property status bits are expanded for dynamic access control, just like
file attributes in file system. It will be persisted when the document is
saved. This allows for more flexible control of the properties of an object.
For example, the programmer can now set a `Trasient` bit on the `Shape` of
a `Part::Feature` derived object to make it not persistent, but dynamically
generated when restore, which is the more efficient way of handling OCCT
compound. Python code can use the following code to manipulate status bits of
a property dynamically. This function are implemented in `PropertyContainerPy`
class.

```python
# make some property immutable, i.e. read-only in python
obj.setPropertyStatus(property_name, "Immutable")

# remove immutable from some property
obj.setPropertyStatus(property_name, "-Immutable")

# add multiple status bit
obj.setPropertyStatus(property_name, ["Immutable", "Hidden"])

# get property status of some proeprty
obj.getPropertyStatus(property_name)

# get a list of available property status name
obj.getPropertyStatus()
```

Currently supported property status are,

* `Touched`: this property status cannot be set using `setPropertyStatus()` through
  python, but can be returned by `getPropertyStatus()`. Use `touch()` instead.
* `Immutable`: mark the property as read-only in python. C++ code is not affected.
* `ReadOnly`: make the property read-only in the property view window.
* `Hidden`: hide the property from property view window.
* `Transient`: do not store the property when saving the document.
* `MaterialEdit`: enable material editing in property view.
* `NoMaterialListEdit`: disable material list editing in property view.
* `Output`: property marked with `output` will not trigger its container's `onChanged()` function
* `LockDynamic`: prevent dynamic property from being removed.
* `NoModify`: prevent calling `Gui::Document::setModified()` on property change.

## `PropertyLists`

Class `PropertyLists`, and most of its derived classed has been refactored for
more code reuse. The original motivation is to add a `TouchList` function, 
which allows the programmer to discover exactly which entry/entires has been
changed to save potentially expensive operations. 

A new class, `PropertyListsBase`, is added as an abstract parent class that is
_NOT_ derived from `Property`. This is to make it possible for other list like
property class to reuse the list handling code without having to derive
from `PropertyLists`. One example is the `PropertyLinkList` class described
later. This class, and all other link type property classes, are changed to
derived from a new `PropertyLinkBase`.

`PropertyListBase` provides a helper function `_setPyObject()` to set value
from a Python object. It accepts Python object as a `tuple`, `list`, or
newly supported `dict(index:value)`. For `dict`, it also supports appending by
using index of -1. The implementation of `_setPyObject()` extract the Python
objects for items from the input Python sequence or dictionary, put them into
C++ vector, and then calls the virtual function `setPyValues()` for further
assignment. The reason of the underscore in `_setPyObject()` is to avoid name
clash with `Property::setPyObject()`, as `PropertyListBase` is _NOT_ derived
from `Property`.

`PropertyListT` is a template class that implements the generic type logic of
list handling, and provides the implementation of `setPyValues()`. The
implementation is designed to be compatible with original various list
properties. It unifies the API for all list type properties,

* `get/setSize()`

* `getValue()/getValues()`, same function for returning the whole list,
  provided for backward compatibility

* `setValue(const_reference value)`, clear the list and store one value into
  the list.

* `setValue/setValues(list)`, set whole list.

* `operator[](int)`, indexer operator to return one value

* `_set1Value()`, a helper function for implementing `set1Value()`. It is
  not public because the original list properties have non-uniform behaviors
  for this function. Some will touch the container, while others will not. It
  is made protected to force derived class to provider its own `set1Value()`.

* `setPyObject()`, for property assignment from Python object. It first tries
  to assign the Python object as a single item. If failed, then call
  `PropertyListBase::_setPyObject()` to assign it as a list.

* `getPyValue()`, a template pure virtual function for derived class to provide
  logic of converting a Python object into a single item C++ object.

* `setPyValues()`, convert vector of Python object of items into vector of 
  C++ object. It relies on `getPyValue()` for converting.

`TouchList` is supported by API `getTouchList()` which returns a set of indices
of the changed values. Python code can use
`obj.getPropertyTouchList(prop_name)` to obtain the touch list. `PropertyListT`
will take care of maintaining the touch list on property changes.

## Link Properties

Current FreeCAD lacks a proper abstraction of the link type properties. The
involved core logic enumerates all link property types and deals with them
separately. However, link type properties become a lot more complicated due
to the introduction of external links and topological naming. The need of
a proper abstraction become a must. 

The main difficulty of providing a common abstract class for link property is
that the list type link property is derived from `PropertyLists`, while others
are derived directly from `Property`. The above refactor of `PropertyLists`
solved this problem by isolating the list handling logic to a non property
class.

A new class `PropertyLinkBase`, derived from `Proeprty`, is used as the common
parent class of all link type properties. It defines the following API
interface,

* `updateElementReference()`, called to update geometry element reference due
  to model changes. See [here](Topological-Naming#property-link) for more
  information

* `referenceChanged()`, check if element reference has been changed after
  document restore. If so, the core will mark the object for recompute to
  regenerate the element mapping.

* `getLinks()`, return the linked object(s) of this property, and optionally
  any [subname](Link#user-content-subname) references, including the geometry
  element references.

* `breakLink()`, reset the link property .

A new link scope, `Hidden`, has been added to allow some form of cyclic
dependency. The `getLinks()` API above will not return linked object if the
property has `Hidden` link scope. And various link property with this scope
will not call `DocumentObject::_addBackLink()`. Therefore, the dependency
information of `Hidden` link scope will be hidden from the dependency checking
logic in the core. However, the link property with `Hidden` scope still enjoys
benefits like auto element reference update, and auto link breaking from the
core. An example use case is the `DocumentObject::ColoredElements` property
which holds the element reference for coloring. This `PropertyLinkSubHidden`
property links to its own container.

A lot of logic has been added in various link properties to correctly handle
copying/importing/exporting objects from/to multiple documents. Basically, to
avoid name clashing of objects from different documents, the object name will
be changed to `<ObjectName>@<DocumentName>` when exporting, and then strip off
the postfix when importing. Any new style geometry element reference will also
be stripped during exporting, and be marked for regeneration when importing.

<a name="xlink"></a>A new link type property, `PropertyXLink`, is added
to support linking of object from external document. It is derived from
`PropertyLink` which makes it a drop-in replacement in case external linking is
required. No equivalent external link support for other type of link property,
such as `PropertyLinkList`, because of the added complexity of managing
external links. To link multiple external objects, it is recommended to group
the objects in the external document first, and then use a single object with
`PropertyXLink` to link them.

## `PropertyPersistentObject`

A new property is introduced to allow Python code to add any FreeCAD core
objects that are derived from `Base::Persistence` as a property. This allows one
to use the composite pattern, in other word, to implement an object using one
or more existing objects. 

This property accepts a string of the type name with its `setValue()`, and will
create the object of the given type. And the object can be retrieved by its
`getValue()`

One use case is the `ChildViewProvider` property of `Gui::ViewProviderLink`,
which are intended to hold any object derived from
`Gui::ViewProviderDocumentObject`. `ViewProviderLink` is designed to be able to
work with any object with `LinkBaseExtension`. It does not provide any actual
visual representation, but instead, relies on the linked object's view provider
to provide them. However, there may be cases where it makes sense for
`ViewProviderLink` to provide the visual for its attached object directly. With
`ChildViewProvider`, user code can now attach a secondary child view provider,
which may not be reside in the core.

A concrete use case is the `AsmPartGroup` container used by `Assembly3`. This
container is of type `Part::FeaturePython`, but it normally does not hold any
`Shape` of its own, instead, it uses `ViewProviderLinkPython` to borrow its
contained part object's visual. The assembly can later on be _frozen_, such
that any changes of its children parts has no effect on the assembly. In order
to achieve this, the part group's `Shape` now holds a compound of the
containing part shapes before _freezing_. And a child view provider of type
`PartGui::ViewProviderPart` is created to provide the visuals.

## `Application`

A few of new APIs are added to `App::Application` class. 

* `addPendingDocument()`, used by `PropertyXLink` to pend external documents
  when opening document with external links, so that the linked documents can
  be automatically opened together with the main document.

* <a name="transaction"></a>`get/set/closeActiveTransaction()`, these APIs are
  part of the implementation of the new core functionality called 
  _Auto Transaction_. The other part exists in `App::Document`. Programmer can now
  call `Application::setActiveTransaction()` to set a potential transaction
  with a given name. The transaction is only created when there is any actual
  changes. Moreover, each transaction now has an associated integer ID.
  `setActiveTransaction()` will create a new ID each time it is called. Each
  changed document will check for active transaction by calling
  `getActiveTransaction()` and create the actual transaction with the current
  active transaction ID. The net effect is that, all modified documents during
  a single logical operation will use the same ID, so that all related
  transactions can be undo/redo together like a single logical transaction.
  This function is essential to support editing document with external links.

  To make the new functionality transparent and yet backward compatible in
  exiting code, `App::Document::open/commit/abortTransaction()` has been
  modified to call instead `Application::set/closeActiveTransaction()`. The
  actual transaction operation is done by
  `App::Document::_open/_commit/_abortTranaction()`. This behavior is by
  default on, and can be turned off by a boolean parameter
  `BaseApp/Preference/Document/AutoTransaction`

* `checkLinkDepth/getLinksTo/hasLinksTo()`, a few helper functions to obtain
  `Link` object across all opened documents.

Although not part of `Application`, `ObjectLabelObserver` was implemented in
`Application.cpp`. This class is removed, and label duplication detection logic
is moved into `PropertyString` for more efficiency, and also to allow object
itself to control labeling policy. See description [below](#relabel) for more
details.

## `Document`

A new property, `ShowHidden`, has been added to tell tree view whether to show
hidden object of this document. And each `ViewProviderDocumentObject` has a new
property, `ShowInTree`, to mark if this object should be hidden in tree view.
These are added to make it more flexible for user to organize the tree view
hierarchy.

Another new property, `UseHasher`, is added to control whether to use hasher
to shorten element names generated by topological naming.

API changes are listed below,

* `getObjectByID()`, a new API to retrieve object based on its integer ID,
  which is auto generated by the document for each new object. The ID is unique
  within the document, it is mainly used for encoding shape history for
  topological naming.

* `_buildDependencyList()`, helper function to obtain a list of all dependent
  objects of the given object(s). It supports external linked object, meaning
  that the obtained object list may include objects from other document.

* `getDependencyList()`, replace the original implementation with
  `_buildDependencyList()`. In addition, improve error reporting by finding
  the objects involved in dependency loops.

* `getDependentDocuments()`, get a list of documents that are dependent to this
  document. 

* `mustExecute()`, check if any of the objects in this document must be
  executed. This function will account for external dependencies, so that it
  will report true if any externally dependent objects are touched. The
  `Recompute` button on the toolbar will be activated or deactivated by
  checking with this function.

* <a name="recompute"></a>`recompute()`, there are several changes to recomputation logic,

  * `recompute()` now accepts an optional input vector of objects, in order to
    support partial recomputation. Only the given objects, and all their
    dependent objects will be recomputed. And since it uses `getDependencyList()`
    to obtain the dependent object list, the recomputation may consist objects
    from multiple documents.

  * `recompute()` accepts a second parameter `force` to force recompute
    regardless of `SkipRecompute` flag of its owner document. Because of the 
    support of partial recompute, it is now possible to force recompute part of
    the objects without affecting other objects. So when the user sets the
    `SkipRecompute` of some document, he can still edit object, and after
    finished, only recompute the editing object and all its dependent objects.
    Note that without this `force` recomputation functionality, many object
    will report error when finish editing if its owner document is marked as
    `SkipRecompute`, because some editing logic expects the recomputation to
    work, and will check the recomputation result for verification.
    
    The force recompute is not initated from `App` namespace directly, but from
    `Gui` namespace through a new `signalSkipRecompute`. `Gui::Document` will
    catch this signal in `Gui::Document::slotSkipRecompute()`, and check if the
    recomputation request meets the following criteria,

    * The given list of recomputing objects must not be greater than one; and
    * The document is the active document; and
    * If we are the editing document, then force recompute the editing object; or
    * If no object is given, then force recompute the active object of this
      document.

    The above slightly convoluted logic is designed such that existing object
    editing code can now work when `SkipRecompute` is active, without any
    modification.

  * Add support for some form of dependency inversion using a two-pass
    iteration. An example of dependency inversion is `Sketcher::SketchExport`
    object, which is a child object of some `Sketcher::SketchObject`. However
    it requires the parent being update first in order to obtain the updated
    exported geometry. 
    
    The two-pass recomputation works like this. The first pass proceed as
    normal. After its done, any object that is still touched will be marked
    with a newly added `ObjectStatus::Recompute2` flag. The second pass will
    then be launched. Those objects that need dependency inversion shall check
    its dependent objects for this flag, and do not touch them if found. If
    there are still objects touched after the second pass, an error message
    will be printed.

    In the above `SketchExport` example, the parent `SketchObject` will update
    all its children `SketchExport` in its `execute()` function in the first
    recomputation pass. Since all children (i.e. dependent) objects are
    recomputed before its parent. Those `SketchExport` object will still be
    touched after the first pass, and will be marked with `Recompute2` in the
    second pass. `SketchObject` will then skip updating its children and
    complete the second pass without any problem.

  * Error handling logic is changed. When some object reports error during
    recomputation, this object and all objects in its `InListRecursive`
    will be removed from pending object list, after which recomputation
    resumes as normal.

* `recomputeFeature()` has an extra argument `recursive`. When set to true
  it will recompute the given object and all its dependent objects.
    
* `open/commit/abortTransaction()`, and `undo/redo()` has been modified to
  support grouped transactions. See [here](#user-content-transaction) for
  more details.

* <a name="exporting"></a>`isExporting()`, a new function added to distinguish
  different type of exporting. It can return the following value,

  * `NotExporting`

  * `Exporting`, normal exporting with all selected objects being copied.

  * `ExportKeepExternal`, like normal export but keep the external linked
    object as it is. This signals the various link properties to not
    auto rename its external references.

* `copyObject()` and `exportObjects()` have been modified to be aware of
  external linked objects, and offer user a choice of whether to copy those
  external objects or not. The object name mapping logic has been moved from
  helper class `MergeDocuments` to various link properties.

* `importLinks()`, with a given list of objects, this new function will copy
  all external referenced object into this document, and update all external
  linking property with internal reference, including any intermediate
  reference inside the subname. Consider a `PropertyLinkSub` containing
  a subname reference of `group1.group2.cube`, where the `group2` is externally
  linked by a `Link` object. After import, the object name may be changed due
  to name clash, to say `group3`. `importLinks()` will make sure to update the
  subname reference to `group1.group3.cube` as expected.

* `addObject()` has an extra argument `viewType` to allow overriding the
  objects view provider. For example, `Gui::ViewProviderLink` is designed to
  work for any type of object that contains an `App::LinkBaseExtension`.
  This makes it possible to change a `Part::FeaturePython` object's view
  provider to `ViewProviderLink`, so that the object can behave like a
  `Link` object and still be used by other object that only accept
  `Part::Feature` derived objects. See [here](Link#user-content-python-link)
  for and example.

* `Restore(), Save()`, modified to support [partial document loading](#partial-document-loading)

* `restore(), importObjects()`, modified to unified logic of finishing 
  restoring objects by calling a new helper function `afterRestore()`. 

* `afterRestore()`, called after objects are restored either from physical
  document or clipboard. It first sort the object using `getDependencyList()`,
  and then calls object's `onDocumentRestored()` function in the correct
  order, in order to make it possible for object to dynamically generate
  content during restoring from its dependent object. When calling each
  object's `onDocumentRestored()` function, it will also trigger the
  corresponding view object's `finsihRestoring()` using a new
  `signalFinishRestoreObject()`. 

### Partial Document Loading

A new feature is introduced to improve FreeCAD's scalability of handling 
complex and nested hierarchical models, such as a mechanical assembly.
A document object can now report if it can be loaded partially when restoring,
using the new API [canLoadPartial()](#user-content-canloadpartial).

When saving the document, before writing the actual objects, `Document` will
now save a dependency list of each object, containing the names of its direct
depending objects. `Document` will also call each object's `canLoadPartial()`
and save the result together with dependency list. 

When a document is opened by the end-user, which we'll be calling as the
_main document_, it will always be fully loaded. But when a document is
automatically opened due to external linking inside the main document, the
dependency list in the external document will be consulted, and only the
linked objects and their dependent objects will be considered for restoring.
The candidate objects will be further pruned using their stored 
`canLoadPartial()` return values. If an object returned 1, then all its direct
depending object will only be created, but not restored, which means the
further depending objects may not be created at all, because a newly created
object normally has no dependency. If an object returned 2, then the object
itself will be created but not restored. All other candidate objects will
be restored as normal.

The tree view will not show any _partially loaded_ objects, i.e. those created
but not restored. The _partially loaded_ documents will hide their window to
discourage the user to set them as active document. Any modification to the 
_partially loaded_ document will generate a warning, and will not be saved.
The tree view shows a grey icon for partial document. The user can double
click the grey icon to reload the document fully, Or right click the _main
document_ and select 'Reload document' to reload all its direct depending
document fully.

An example of implementing an object that supports partial loading is the
`Assembly` container in `Assembly3`. Each `Assembly` always has three child
objects, `Constraints`, `Elements` and `Parts`. An `Assembly` can be frozen,
making it immune to any changes of its composing part objects in `Parts`. This
is achieved by store a compound shape of all child parts before freezing in the
`Shape` property of `Parts`. The _freezing_ also allows the assembly to be
loaded partially, because it makes the assembly self sufficient without relying
on any other external dependencies. The `Assembly` container itself will return
0 in its `canLoadPartial()` function, which causes all three of its direct
depending children to be loaded. However, `Constraints` will return 2 in its
`canLoadPartial()`, because a frozen assembly do not need to be solved again,
and all its associated constraint objects will not be created at all. `Parts`
will return 1, so that `Parts` itself is fully restored, because we need the
`Shape`. All the part objects, which are the direct depending objects, will be
created but not restored, and therefore no further dependency will be
introduced.

The following screen cast shows the effect of partial document loading. The
example assembly consists of four externally linked part objects. The screen
cast shows that when you open the main document, the external depending
documents will be opened automatically. Then we proceed to freeze the assembly
object. A new file named _assembly_ is created, and an external link is added
to link to the frozen assembly. As you can see, after the _assembly_ file is
reopened, none of the external part files are opened automatically. The
original main document is partially opened with a grey icon. None of the
partially loaded part objects is shown in the tree. Finally, we double click
the partially opened document to reload it fully, and every part objects
reappeared in the linking document.

[[images/partial-loading.gif]]

## `DocumentObject`

A new property, `Visibility`, has been added to make it possible to set
and get visibility status of an object in `App` namespace. This makes it
possible to determine the children visibilities in some group object without
loading GUI, for example, to generate a compound shape representation of
a group.

Here is a list of changed/added APIs,

* `getID()`, obtain the internal integer ID of this object.

* `getViewProviderNameOverride()`, obtain the user specified overridden view
  provider. `App::Document::addObject()` allows user to override the view
  provider when creating an object. This overridden view provider type is
  persisted when saving the document, and can be retrieved using this function.

* `getExportName()`, obtain a suitable object name for exporting, depending
  on the exporting type reported by [Document::isExporting()](#user-content-exporting)

* `getLinkedObject()`, obtain the linked object, and optionally with the
  accumulated transformation matrix. For non-`Link` type object, or, a `Link`
  type object that is not linked, this function shall return the object itself.
  So it is always safe to call this function, and expecting it to return a
  valid object. In case there are multiple levels of linking involved, this
  function allows the caller to retrieve the linked object recursively.

  In most cases, programmer does not need to explicitly call this function to
  be able work with `Link`, which will automatically delegate most of the APIs
  to linked object. One particular use case of this function is that when given
  a [subname](Link#user-content-subname) reference, say
  `group1.group2.cubeLink`, if the last object `cubeLink` is a `Link` object
  that links to the actual `cube`, the `resolve()` function will return that
  `Link` as the resolved object, not the actual `cube`. The caller can use
  `getLinkedObject()` to obtain the linked object.

* `getSubObject()`, obtain a sub-object through a given subname reference,
  and optionally return the accumulated transformation matrix, and associated
  Python object. This is the most important API, created specifically to make
  `Link` work with group type objects. 

  The default implementation in `DocumentObject` is to search the `OutList` of
  the object for the next hierarchy sub-object. And once found, recursively
  calls the sub-object's `getSubObject()` for the next hierarchy. Group type
  object may override this behavior. Non group type object may also override
  this function to return the associated Python object when called with an
  empty subname reference, i.e. when reference to the object itself, or with
  a sub-element reference, i.e. a reference that is not ending with a dot. For
  example, `Part::Feature` overrides this function to return the object's
  shape, or sub-shape through the Python object. `Link` type object overrides
  this function to delegate the call to the linked object.

* `getSubObjects(reason)`, return a list of subname references for all the
  child sub-objects. The purpose of this function is mostly for exporting
  a group type object (with `reason = GS_DEFAULT`). Each subname reference
  returned may contain more than one hierarchy. For example, The `Assembly`
  container in `Assembly3` workbench uses this function to skip one hierarchy
  that contains constraints and stuff during exporting. The caller is expected
  to call `getSubObject()` with the returned subname reference to obtain the
  actual child sub-objects.

  A newly added _Box Element Selection_ also uses `getSubObjects()` for
  enumerating sub objects, with `reason = GS_SELECT`. `PartDesign::Body` will
  return child features when given this reason. For `GS_DEFAULT`, i.e. shape
  exporting, `Body` will not return its child feature so as to output its own
  shape instead.

* `canLinkProperties()`, tells `Gui::PropertyView` whether to show the
  properties of the linked object together with properties of `Link` object
  itself. For example, `App::Link` will only allow linking property when it
  is not linked to a sub-element, such as a face or an edge.

* `hasChildElement()`, `set/isElementVisible()`, controls the children visibility
  for a group type object. Because the child object may be claimed by other
  objects, it is essential to have independent control of children
  visibilities. These APIs are designed to abstract how group manages the child
  visibility. For example, `App::LinkGroup` stores the children visibilities as
  a `PropertyBoolList`

* `resolve()`, helper function to parse subname reference and resolve the final
  object, and optionally the immediate parent of the final object, the final
  object reference name (for calling `set/isElementVisible()`), and the
  sub-element reference if there is one.

* `resolveRelativeLink()`, helper function to adjust a subname reference in
  case the two given object belongs to the same parent. See the code 
  [comments](https://github.com/realthunder/FreeCAD/blob/ab72b904e55856be249751a7e34227542722c019/src/App/DocumentObject.h#L393) for an example.

* `recomputeFeature()`, has an extra argument, `recursive`, to recompute the
  dependent objects together with this object if set to true.

* `getInListEx()`, obtain the (optionally recursive) `InList` of this object.
  This function exists before FreeCAD completely switched to the new DAG
  handling code. It uses `OutList` of other objects to calculate the recursive
  `InList`. The algorithm takes external links into account.

* `getOutList()`, is slightly modified to keep an internal cache, so that
  it does not have to re-scan all properties every time to construct the
  `OutList`. The cache is invalidated when any property changes.

* <a name="relabel"></a>`allowDuplicateLabel()`, `onBeforeChangeLable()`, new
  APIs to let object control if it allows its label to be a duplicate with
  others. `PropertyString`, once detected it is a label, will call its owner's
  `allowDuplicateLabel()` to see if duplicate label is allowed. If not so, and
  the label is about to be changed, `PropertyString` will call its owner's
  `onBeforeChangeLable()` function to notify the new label. For example,
  `App::Link` supports label inside subname reference, and will auto adjust all
  affected label subname reference in `onBeforeChangeLabel()` function.

* `onUpdateElementReference()`, will be called when any element reference in
  some of the object's link property is changed, due to topological changes
  in the linked model.

* <a name="canloadpartial"></a>`canLoadPartial()`, allow the object to inform
  the document whether it can can be partially loaded when restoring as an
  externally linked object. 

  Partial loading means the object will be created but not restored when its
  document is restored.

  The return value of this function has the following meaning,
  * 0, the object must be fully loaded, 
  * 1, the object must be fully loaded, but all its directly depending objects
    can be partially loaded.
  * 2, the object itself can be partially loaded.

* `redirectSubname()`, allows an object to redirect selection to some other
  object when (pre)selected in the tree view. An example will be the
  [Relation](Navigation#relation-group) object in `Assembly3`, which redirects
  the selection to the corresponding part object. The `Relation Group` can
  be consider as an alternative organization of the assembly object tree.
  The _redirection_ functionality greatly simplified the implementation by
  reusing the part object's visual.

## `DocumentObjectExtension`

The following new extension point as been added for the new APIs of
`DocumentObject`, which are used by `App::LinkBaseExtension`

* `extensionGetLinkedObject()`
* `extensionGetSubObject()`
* `extensionGetSubObjects()`
* `extensionHasChildElement()`
* `extensionIsElementVisible()`
* `extensionSetElementVisible()`

## `FeaturePython`

Here is a list of new APIs that can be overridden with Python code. The syntax
is shown here in Python.

* `mustExecute(obj)`, return `True` to indicate the object must be executed.

* `allowDuplicateLabel(obj)`, return `True` to enable duplicated label of
  this object regardless of preference setting.

* `onBeforeChangeLabel(obj,newLabel)`, called when label is about to change to
  the string given by `newLabel`. The object can customize label change by
  returning a string that is different from `newLabel`.

* `getViewProviderNameOverride(obj)`, return a string of the view provider type of this
  object. This allows Python code to override the view provider of an object.

* <a name="getsubobject"></a>`getSubObject(obj,subname,retType,matrix,transform,depth)`,
  return the sub object referenced by `subname`. Python code is expected to
  resolve the first hierarchy referenced in `subname` among its children, and
  then recursively calls the resolved child's `getSubObject()` to continue down
  the hierarchy. The `getSubObject()` exposed by `DocumentObjectPy` has almost
  exactly the same argument as this function. The meaning of the argument is
  described below,

  * `obj`, this object

  * `subname`, dot separated [subname](Link#user-content-subname) reference.

  * `retType`, indicating the requested return type of this function
    * 1 indicates a return type of `tuple(subObj,matrix)`, where `subObj` is
      the resolve sub-object, and `matrix` is the accumulated transformation
      matrix. 
    * 2 indicates a return type of `tuple(subObj,matrix,pyObj)`, where the
      actual type of `pyObj` is implementation dependent. For example,`Part::Feature`
      derived object returns the `Shape` in `pyObj`.
    * 0 means to return only `pyObj`. This value is only meaningful when
      calling `DocumentObjectPy.getSubObject()`, but never for `FeaturePython.getSubObject()`.

  * `matrix` is the input transformation matrix.

  * `transform`, indicate whether to accumulate the placement of this object
    into the transformation matrix. The purpose of this argument is so that
    a `Link` object can choose whether or not to override the placement of its
    linked object.

  * `depth`, is an integer indicating the current recursive level. It is used
    to detect cyclic references. Increase this `depth` value when calling child
    object's `getSubObject()`.

* `getLinkedObject(obj,recurse,matrix,transform,depth)`, it has similar
  argument as `getSubObject()`, except `recurse`, which indicates whether to
  follow the link recursively in case of multi-level linking.

* `getSubObjects(obj)`, return a list of subname string referencing the child
  objects for exporting.

* `hasChildElement(obj)`, return `True` means this is a group type object

* `isElementVisible(obj,subname)`, return -1 if the object does not support
  children visibility control, and 1 if the child referenced in
  `subname` is visible, 0 if hidden. For performance reason, `subname` will
  only contain one hierarchy, i.e. reference to the immediate child object.

* `setElementVisible(obj,subname)`, change child object visibility

* `canLinkProperties(obj)`, return `True` to allow `PropertyView` to display
  linked object properties together with the object's own properties.

## Topologically Naming

There are also quite a few API changes for supporting the new _Topological Naming_
feature. Please refer to [this](Topological-Naming) and [this](Topological-Naming-Algorithm)
document for more details.

## Expression

There is a major refactor of the `Expression Engine`. You can find more details
at [[Expression and Spreadsheet]].

# `Gui` Namespace

## `ViewProvider`

* `getModeSwitch()`, `getTransformNode()`, exposes the internal `Coin3D` node
  used by the view provider. It is used by `ViewProviderLink` for 3D
  representation sharing.

* `getDefaultMode()`, expose the current display mode regardless of 
  the object's visibility. Used by `ViewProviderLink` to link object
  with independent visibility control.

* `getElementPicked()`, this new function is the core API for implementing
  independent selection of instances of the same object. The default
  implementation is to call the original `getElement()` function, which makes
  this new API backward compatible. For hierarchical view provider, such as,
  `ViewProviderLink` or group type view provider, the implementation can follow
  the `Coin3D SoPath` within the input `SoPickedPoint` to disambiguate the user
  selection, an then translate the path to a subname reference as output.

* `getDetailPath()`, this is the reverse function of the above
  `getElementPicked()`. It accepts a subname reference as input, and output
  the `Coin3D SoPath`, along with the `Coin3D SoDetail` to locate user selected
  the `Coin3D` node.

* `partialRender()`, new helper function to activate [partial rendering](Link#secondary-context)

* `beforeDelete()`, new API that is guaranteed to be called before the object
  is about to be deleted, either when the object is being removed, or the 
  document is about to be closed.

* `canDragAndDropObject()`, new API to tell tree view whether this object 
  supports removing the dropped object from its original parent. Current 
  FreeCAD supports holding `CTRL` key while dropping to signal the user's
  intention to drop the object without removing it from its original parent.
  For some type object, such as `Link`, it sometimes does not make sense to
  remove the dropping object from its original parent. 

* `canDropObjectEx()`, `dropObjectEx()`, new API for dropping object with
  object's hierarchical information, pass in as a subname reference and top
  parent object. It is added to support relative linking functionality of
  `Link` object, where it links directly to the top parent, and indirectly
  to the dropping object through a subname reference. In other words, the
  linking to the dropping object is relative to the given top parent.

* `canRemoveChildrenFromRoot()`, this function tells tree view to not remove
  children claimed by this object from the root. This API is originally
  added to demonstrate the fact that `LinkGroup` does not remove object
  from the global coordinate system, by leaving the object added to the
  group at the root level. You can toggle the visibility of the instances
  independent of each other. 

* `update()`, this function is made virtual, and is overridden in
  `ViewProviderDocumentObject` to correctly handle change of the newly added
  `Visibility` property of `DocumentObject`.

* `get/setElementColors()`, new API for managing sub-object/element colors.
  They are implemented in `PartGui::ViewProviderPartExt` for setting colors of
  faces, edges, and vertices, and also in `Gui::ViewProviderLink` to override
  colors of its linked object.

* `startEditing()`, this function is made virtual so that `ViewProviderLink`
  can redirect the call to its linked object.

* `setRenderCacheMode()`, new API for configuring [render caching](Link#user-content-cache-setting).
  Currently only implemented in `PartGui::ViewProviderPartExt`.

* `signalChangeIcon`, new signal used to inform tree view of icon change,
  exposed to Python through a method of the same name.

## `ViewProviderDocumentObject`

* `showInTree()`, override to let user control the tree item visibility.
  It simply reports the value of a newly added property, `ShowInTree`. User
  can hide the item by right click the item in the tree view, and select menu
  action _Hide item_. To review the hidden items, right click any item, or the
  document item, and select _Show hidden item_.

* `OnTopWhenSelected`, a new enumeration property added to provide
  always-on-top effect when selected. The actual visual effect is implemented
  in `View3DInventorViewer`. Supported values are,

  * `Disabled`, default value.
  * `Enabled`, make the object always-on-top when either the whole object or
    any sub-element is selected.
  * `Object`, on-top only when the whole object is selected
  * `Element`, on-top only when any sub-element is selected

* `reattach()`, new virtual API to be called when an object removal is undone.

* `update()`, override to handle change of the newly added `Visibility` from
  `DocumentObject`.

* `canDropObjectEx()`, override to disable dropping of external object,
  that is, object from an external document.

* `getElementPicked(), getDetailPath()`, override to support hierarchical
  selection of group type object, i.e. those object returns a non null node
  in `getChildRoot()`.

* `getBoundingBox()`, new function to obtain object bounding box using
  `Coin3D SoGetBoundingBoxAction`, regardless of object's visibility

* `forceUpdate(), isUpdateForced()`, new API added because some object (e.g.
  `Part::Feature`) has an optimization to disable visual update when the
  object is hidden, and `ViewProviderLink` needs a way to force update the
  visual in case there is a visible `Link` to that hidden object.

## `ViewProviderExtension`

New extension point corresponding to the newly added APIs,

* `extensionBeforeDelete()`
* `extensionCanDragAndDropObject()`
* `extensionCanDropObjectEx()`
* `extensionDropObjectEx()`
* `extensionModeSwitchChange()`, this extension function is called by
  `setModeSwith()` and `setOverrideMode()` function of `ViewProvider`
* `extensionStartRestoring()`
* `extensionFinishRestoring()`
* `extensionGetElementPicked()`
* `extensionGetDetailPath()`

## `ViewProviderPythonFeature`

Here is a list of new APIs that can be overridden with Python code. The syntax
is shown here in Python. 

* `getElementPicked(pp) -> String`, expect to return a string as the subname
  reference given the pick point. Return `None` means no selection is found
  with this object. `pp` here is an object of `pivy.coin.SoPickedPoint`.

* `getDetailPath(subname,path,append) -> SoDetail|Bool`, expect to return
  either an object of `pivy.coin.SoDetail` or `Boolean` given a subname
  reference. `True` means the whole object is referenced, or else no
  reference is found. 

  * `subname`: a String containing the subname reference
  * `path`: `pivy.coin.SoFullPath`, on input this is current path of the selection,
    as this function maybe recursively called by a higher hierarchy. On output,
    append `SoNode` of the current hierarchy.
  * `append`: if `True`, then append the view provider root node and switch node.
    Or else, append node below the switch node. This is used by `ViewProviderLink`,
    because it will replace the root and switch node.

* `setEditViewer(obj, viewer, mode) -> Bool`

* `unsetEditViewer(obj, viewer)`

* `canRemoveChildFromRoot() -> Bool`

* `canDragAndDropObject(obj) -> Bool`

* `canDropObjectEx(obj,parent,subname) -> Bool`
  * `obj`: the dropping object
  * `parent`: the top parent of the dropping object
  * `subname`: the subname reference representing the hierarchy path leading
    from top parent to the dropping object.

* `dropObjectEx(obj,parent,subname)`, this function as the same argument as
  `canDropObjectEx()` above.

* `startRestoring(), finishRestoring()`, these functions are modified to call
  C++ object counterpart as well as Python code, to make sure the view provider
  extension has a chance of handling restoring.

* `getIcon()`, add support for Python code to directly return a `QIcon` Python
  object. The actual implementation of extracting `QIcon` from Python object is
  in `Gui::PythonWrapper::toQIcon()`.

## `Application`

* `signalShowHidden`, new signal triggered when a document changes its
  `ShowHidden` property, i.e. whether to show the hidden object in tree view.

* `editDocument(), setEditDocument()`, for getting and setting the current
  editing document.

## `Document`

* `signalShowHidden`, new signal triggered when this document changes its
  `ShowHidden` property, i.e. whether to show the hidden object in tree view.

* `signalShowItem`, new signal triggered when an object of this document
  changes its `ShowInTree` property.

* `getViewProviderByPathFromHead()`, new API to return the first view
  provider found given a `Coin3D SoPath`, used to support context-aware
  hierarchical selection of group type object. This function obsoletes
  `getViewProviderByPathFromTail()`, because the default implementation
  in `ViewProviderDocumentObject::getElementPicked()` is able to walk down
  the hierarchy by itself.

* `getViewProvider(SoNode*)`, obtain a view provider given its root `Coin3D`
  node. The private class `Gui::DocumentP` now maintains a map (`_CoinMap`)
  from root node to view provider to accelerate view provider lookup.

* <a name="in_place_edit"></a>`setEdit(), set/getEditingTransform(),
  get/setInEdit()`, these APIs are either new or modified to support in-place
  editing. `Link` type object makes it possible for the same object to appear
  in multiple placements, and even across document boundary. These APIs,
  together with `View3DInvertorVewer::setupEditingRoot()`, allows the user to
  edit an object out of its original placement.

  It works like this. `setEdit()` is modified to accept an additional argument,
  `subname`, in order to support editing sub-object. If `subname` is null, then
  the subname reference will be deduced from the current selection by querying
  `Gui::SelectionSingleton`. `setEdit()` uses the subname to obtain the current
  object transformation, by calling `App::Document::getSubObject()`, and the
  resulting transformation matrix is exposed through `getEditingTransform()`,
  but can be overridden by user code through `setEditingTransformation()`.

  An editing root node is added to each `View3DInventorViewer` scene graph.
  `ViewProvider` that supports editing in multiple placements can either,

  * Add any editing helper nodes manually into the editing root by calling
    `View3DInventorViewer::setupEditingRoot(node,matrix)` with some
    transformation obtained from `getEditingTransform()` or through other
    meaning. This method is demonstrated in `Gui::ViewProviderDragger`.

  * Simply calls `setupEditingRoot(0,0)` which will cause `View3DInventorViewer`
    to transfer all children nodes of the editing view provider root node into
    the editing root with a replaced transformation node initialized using
    transformation obtained from `getEditingTransform()`. Node changes will be
    automatically reverted after editing is done. This method has the side
    effect of automatically hiding all other non editing instance of the
    editing object, which is desired in some use cases, while the first method
    keeps all other instances untouched. This method of editing is demonstrated
    in `SketcherGui::ViewProviderSketch`

  The ideal place to call `View3DInventorViewer::setupEditingRoot()` is in
  `ViewProvider::setEditViewer()` function.

* `handleChildren3D()`, improve efficiency of node tree building of group type
  view provider.

* `checkTransactiveID()`, private helper function to check and warn user if
  a multi-step undo/redo is about to break a [grouped transaction](#user-content-transaction).

* `undo(), redo()`, modified to check other document for grouped transaction
  by calling `checkTransactionID()`

* `isPerformingTransaction()`, unlike `App::Document::isPerformingTransaction()`,
  this function will report `True` while undo/redo this and possibly other
  document's grouped transaction.

* `beforeDelete()`, new function that will be called before this document is 
  about to be deleted. It will call all its children view provider's new
  `ViewProvider::beforeDelete()` API.

* `save()`, modified to check for dependent external documents, and ask user
  for permission to save them before saving the current document, so that
  the external file time stamp is up-to-date when current document is saved.

* `Restore()`, add toggling view provider `isRestoring` flag.

* `slotFinishRestore()`, catch the new
  `App::Document::signalFinishRestoreObject()` signal and calls the
  `ViewProvider::finishRestoring()`, and reset its `isRestoring` flag. This
  unifies the logic of both restoring from physical document and importing
  from clipboard, and fixed the missing call of `ViewProvider::finishRestoring()`
  when importing.

* `slotFinishRestoreDocument()`, modified to check for any object that is still 
  touched, and not reset `Modified` status if found.

* `slotNewObject()`, modified to call the new `DocumentObject::getViewProviderStored()`
  function to allow Python code to override C++ view provider type. And also
  call the new API `ViewProviderDocumentObject::reattach()` in case the new
  object is the result of undoing a previous delete operation.

* `slotDeleteObject()`, modified to handle the new in-place editing view provider, 
  and call the new `ViewProvider::beforeDelete()` API.

## `Command`

Because the addition of external `Link`, Python code can not always use
`App.ActiveDocument.getObject()` function to obtain the correct object in
commands. Great effort has been spent to correct this coding habit in various
workbenches, such as `Part`, `PartDesign`, and `Sketcher`. The following helper
macros, declared in `Gui/Command.h`, are provided to ease the migration.

* `FCMD_DOC_CMD(doc,cmd)`, execute the string command in `cmd` using C++
  document object `doc`, it expands to the following simplified code with
  safety checking omitted. Note that `cmd` can be a composite expression joined
  by operator `<<`.

```cpp
std::ostringstream ss;
ss << "App.getDocument('" << doc->getName() << "')." << cmd;
Gui::Command::runCommand(Gui::Command::Doc,ss.str().c_str());

// example
//      FCMD_DOC_CMD(doc,"addObject('Part::Feature','" << name << "')")
// issue command
//      App.getDocument(doc_name).addObject('Part::Feature',name)
```

* `FCMD_OBJ_DOC_CMD(obj,cmd)`, equivalent to the following macro call,
   with safety check on object pointer `obj`.

```cpp
FCMD_DOC_CMD(obj->getDocument(),cmd)`,
```

* `FCMD_OBJ_CMD(obj,cmd)`, execute a command with an object, expands roughly to,

```cpp
std::ostringstream ss;
ss << "App.getDocument('" << obj->getDocument()->getName() << "').getObject('" 
   << obj->getNameInDocument() << "')." << cmd;
Gui::Command::runCommand(Gui::Command::Doc,ss.str().c_str());

// example
//      double length = 100.0;
//      FCMD_OBJ_CMD(cube,"Length = " << length);
// issues command
        App.getDocument(doc_name).getObject(obj_name).Length = 100.0
```

* `FCMD_VOBJ_CMD(obj,cmd)`, same as above, but use view object instead of object.
  Note that the input argument `obj` is still required to be a document object.

```cpp
std::ostringstream ss;
ss << "Gui.getDocument('" << obj->getDocument()->getName() << "').getObject('" 
   << obj->getNameInDocument() << "')." << cmd;
Gui::Command::runCommand(Gui::Command::Gui,ss.str().c_str());

// example
//      bool visibility = true;
//      FCMD_VOBJ_CMD(obj, "Visibility = " << (visibility?"True":"False"));
// issues command
        Gui.getDocument(doc_name).getObject(obj_name).Visibility = True
```

* `FCMD_OBJ_CMD2(cmd,obj,...)`, same purpose as `FCMD_OBJ_CMD()` but use
  conventional C variadic function, instead of C++ stream,

```cpp
Gui::Command::doCommand(Gui::Command::Doc,"App.getDocument('%s').getObject('%s')." cmd,
    obj->getDocument()->getName(),obj->getNameInDocument(),## __VA_ARGS__);

// example
//      double length = 100.0;
//      FCMD_OBJ_CMD2("Length = %g", cube, length);
```

* `FCMD_OBJ_CMD2(cmd,obj,...)`, same purpose as `FCMD_VOBJ_CMD()` but use
  conventional C variadic function, instead of C++ stream,

```cpp
Gui::Command::doCommand(Gui::Command::Gui,"Gui.getDocument('%s').getObject('%s')." cmd,
    obj->getDocument()->getName(),obj->getNameInDocument(),## __VA_ARGS__);

// example
//      bool visibility = true;
//      FCMD_VOBJ_CMD2("Visibility = %s", obj, visibility?"True":"False");
```

* `FCMD_OBJ_HIDE(obj), FCMD_OBJ_SHOW(obj)`, shortcut of toggle visibility
  using a document object with the newly added `Visibility` property. They
  simply expands to,

```cpp
FCMD_OBJ_CMD(obj,"Visibility=True");
FCMD_OBJ_CMD(obj,"Visibility=False");
```

Below is a list of API changes of class `Gui::Command`,

* `isActive()` function is made public. So that other class can create
  context menus with only active commands. An example usage is in 
  `Gui::Workbench::createLinkMenu()`

* `getObjectCmd(obj,prefix=0,postfix=0)`, static helper function to generate
  Python code of accessing the given object. In case the object passed in is
  invalid or null, it returns a string of "None". For example,

```
    getObjectCmd(obj, "[", "]")

will generate,

    "[ App.getDocument(doc_name).getObject(obj_name) ]"
```

* `getUniqueObjectName()`, accepts an extra optional argument `obj` of
  a pointer to document object. If `obj` is non-zero, it will call the object's
  owner document's `getUniqueObjectName()` function, or else the active
  document's.

* The following APIs are renamed with a proceeding underscore to accept extra
  arguments of source file and line number for better debugging purpose. To
  maintain source code compatibility, macros are added with the same name as
  the original APIs 
  * `doCommand()`
  * `runCommand()`
  * `assureWorkbench()`
  * `copyVisual()`

## `MainWindow`

There are two changes in `MainWindow`

* Action status are no longer updated using a periodic timer, this is to
  prevent lengthy Python code hogging processing time. Instead, the actions are
  updated in response to various signals. A single shot timer is used to limit
  the frequency of action status checking. A new function `updateActions()` is
  added for others to trigger the action update. Search the code for calling of
  this function to see which signals or events are responsible for triggering
  the update.

* Originally, the `actionLabel` is used for display console log message in
  `MainWindow` status bar. And, `QStatusBar::showMessage()` is used to display
  a temporary message, e.g. tree status message, and 3D view pre-selection,
  etc.

  The problem is that `QStatusBar` temporary message takes precedence over
  normal message that displayed inside its child widget (i.e. `actionLabel` in
  this case), which means important messages may be easily overwritten by not
  so important information, like toolbar tips. To fix this problem, the message
  display location are swapped, such that `QStatusBar::showMessage()` is now
  used for console message, and `actionLabel` for the rest. The user now has
  a chance to catch the console message, especially errors and warnings,
  without the report view.

## `SelectionSingleton`

The changes in `Gui::SelectionSingleton` is among the most critical ones in order to
make `Link` work. The goal of the design change is first to make sure existing
code remains working without modification, and then allow new code to obtain
full object hierarchy information of the user selection. See
[here](Link#user-content-selection-context) for more details.

### APIs Extensions

Most of the APIs are extended by introducing an extra integer argument,
`resolve`, which has the following meaning,

* 0, no auto resolving, the returned selected object is the top level parent
  of the selection, and the hierarchical path of the actual selected object
  is returned as a subname reference.

* 1, as the default value, meaning auto resolving for backward compatibility.
  `Gui::SelectionSingleton` will walk down the hierarchy and return the actual selected
  object without selection context. Non-object sub-element reference is still
  kept in the subname reference as before.

* 2, like 1, but translate the sub-element reference to new style mapped
  element name, if there is one. This is used to make the new topological
  naming backward compatible. See [here](Topological-Naming#gui-selection) for
  more details

Signal for selection change is extended in a different way by adding two new
signals, as shown below

* `signaleSelectionChanges`, original selection change signal, for auto
  resolved object. Same as `resolve=1`;

* `signalSelectionChanges2`, no auto resolve, i.e. `resolve=0`;

* `signalSelectionChange3`, resolve with mapped element, i.e. `resolve=2`.

The APIs extended with the `resolve` argument are listed below,

* `addSelectionGate()`
* `countObjectsOfType()`
* `getObjectsOfType()`
* `getSelection()`
* `getSelectionEx()`
* `getComplementeSelection()`
* `hasSelection()`

The extra `resolve` argument is also added to the following class constructor 

* `SelectionObserver`, in addition to `resolve`, another extra augment,
  `attached` is added to allow caller decide if the new constructed observer is
  attached or not.

* `SelectionObserverPython`

### Picked List

Picked list for holding a list of alternative objects picked by mouse selection
in the 3D view. User code get access to picked list through the following APIs,

* `needPickedList()`, check if picked list is activated. User activates the
  picked list by select an checkbox in `SelectionView`.

* `enablePickedList()`, allow user code to enable picked list.

* `hasPickedList()`, check if picked list is empty.

* `getPickedList()`, obtain the picked list with the same format as `getSelection()`

* `getPickedListEx()`, obtain the picked list with same format as `getSelectionEx()`

The actual implementation of constructing the picked list is in
`SoFCUnifiedSelection`, and is passed to `SelectionSingleton` using the
following API of `Selection` with an extra argument, `pickedList`,

* `addSelection()`,
* `rmvSelection()`,

The following screen cast shows the picked list in action.

[[images/pickList.gif]]

### Selection Stack

With deep nested hierarchical models, such as an assembly with lots of
sub-assemblies, navigation among various objects become increasingly difficult.
`SelectionSingleton` now offers a new facility, called _Selection Stack_, to
help user keep track of their selections. It is exposed using the following
APIs,

* `selStackGoBack()`, exposed to Python and end-user through command `Std_SelBack`
* `selStackGoForward()`, exposed to Python and end-user through command `Std_SelForward`
* `selStackPush()`, exposed to Python as `Selection.pushSelSback()`

It works similar to document undo/redo command. There is no automatic saving of
every selection changes. Only commands that are meant for navigation purpose
will call `selStackPush()` command to manually save the current selection to
the selection stack. The core currently provides the following navigation
command,

* `Std_SelBack`, go back to previous save selection
* `Std_SelForward`, go forward to previous backed selection
* `Std_LinkSelectLinked`, select the linked object
* `Std_LinkSelectLinkedFinal`, select the deepest linked object in case of multi-level linking
* `Std_LinkSelectAllLinks`, select all link object that links to the current selection
* `Std_SelectAllInstances`, select all appearance of the current selection in all object hierarchies

These navigation command is best used together with the following new tree view
options, activated by tree view context menu, _Tree view options_
* _Sync Selection_, scroll the tree view to the selected item
* _Sync view_, auto activate the document of the selected item

Python user code can easily implement navigation command as follow, 
* Call `Selection.pushSelStack()` to save the current selection for backing,
* read the current selection, and calculate the next selection target(s)
* Call `Selection.clearSelection()`
* Call `Selectino.addSelection()` to select the intended navigation target(s)
* Call `Selection.pushSelStack()` again to save the selection for potential forwarding.

See [here](Navigation) for an demonstration of the _Selection Stack_ in action.

### Other Changes

* `setVisible()`, a helper function to set/unset/toggle visibility of the
  current selected object. Because the introduction of `Link`, the same object
  can appear in multiple places, some may have separate visibility control, but
  some not. For example, there a two `Link` objects that links to the same
  group, and user selects the same child object under these two linked group.
  Although the same object is selected, but they in different selection context,
  and thus, exists as separate entries in `Selection`. However, when toggling 
  visibility, we must only count them as one entry, or else the double toggling
  will negate each other, resulting no visibility changes. `setVisible()` is
  created to handle these kind of complex scenario.

* `updateSelection()`, new API to re-synchronize the selection and preselection
  status of a given object. For example, it is called by `setVisible()` to 
  update view provider selection highlight when its visibility is toggled.

* `setPreselect()`, accepts an extra integer argument, `signal` with the
  following meaning,

  * 0, default, for informing a preselection has been made;
  * 1, user initiated preselection;
  * 2, reserved for preselection initiated by mouse over tree view item.

## `SelectionChanges`

This is the class encoding the message of selection change. Current
implementation of the class uses raw pointer to refer to the internal list of
selected document and object names. This may cause problem when user code
decides to add or remove the current selection while handling a selection
change event. To make it more robust, `SelectionChanges` now holds all names
inside the class itself.

A few new message types are introduced, mostly for internal uses,

* `SetPreselectSignal`, indicating that the caller wants to initiate
  a preselection, unlike existing `SetPreselect`, which is for informing
  a preselection has been made. This message is handled in
  `SoFCUnifiedSelection::doAction` to make is easy for user code to initiate
  preselection.

* `Show/HideSelection`, for synchronizing view object's selection visual
  highlight when its visibility status changes. This fixes the problem of
  missing selection highlight when an object is made visible. It is handled in
  `View3DInventorViewer::onSelectionChanged()`

* `PickedListChanged`, for selection changes in the new `PickedList` from
  `Selection`. This message is meant to be used by user code. A new handler
  function `pickedListChanged()` is added to `SelectionObserverPython` for this
  purpose.

## `SelectionView`

The parent class is changed from `Gui::SelectionSingleton::ObserverType` to
`Gui::SelectionObserver`, so that `SelectionView` can obtain the full hierarchy
information of the selection. In addition, the observer will be auto detached
when the selection view widget is hidden to improve overall performance.

Add support for toggling `PickedList` function, and displaying of the current
picked list.

## `TreeWidget`

`Gui::TreeWidget` does not really expose any APIs for use by other classes. So
here we will mainly describes the GUI changes, with a brief mentioning of the
implementation changes, which is significant because of the introduction of
[selection context](Link#user-content-selection-context).

The internal object map has changed its key from the object name to object
pointer, because the map may store external object brought in by a `Link`, which
may have the same name as objects from the linking document. Currently, object
map is still maintained per document, which may result in some inefficiency, but
has the advantage of a clear logic separation. Future development may explore
the possibility of a single unified map for objects of all opened documents.

There are actually two tree view in FreeCAD. The other tree view is from the 
combo view. For easy debugging, each `TreeWidget` instance is given a name. And
its selection observer is detached when invisible.

Tree item status update is no longer triggered by a periodic timer for the
sake of both performance and easy debugging. It is now triggered by various 
signals and events, similar to `MainWindow::updateActions()`. Search the code
for calling of `updateStatus()` to find out the actual trigger. A single-shot
timer is used to limited the frequency of update.


Three options are exposed through context menu _Tree view options_,

* _Pre-selection_, enable pre-select of object in 3D view when mouse over tree
  item. The actual preselection logic is implemented by
  `View3DInventorViewer::checkGroupOnTop()`

* _Sync selection_, auto scroll to the tree item when the corresponding object is
  selected in 3D view.

* _Sync view_, auto activate the owner document and 3D view of the object
  selected in tree view.

The following new context menu actions are added when an object item is selected,

* _Show hidden items_, toggle whether to show hidden items in an document.

* _Hide/Show item_, toggle the visibility of object item in the tree view.

* _Recompute object_, recompute only the selected objects and their dependencies.

The tree view will now catch `signalRecomputed`, and auto expand and scroll to
the first object that reports error during recomputation.

New context aware drag and drop support, including dragging object across
document boundary.

## `PropertyView`

Add support for `Link` object, which display the properties of the linked object
together with the ones from link itself.

Added timer to limit the frequency of handling selection change, which fixes
the severe slow down on large selection.

Clear property view on undo/redo to avoid potential crash.

Dynamically detach selection observer on hidden.

## `DlgPropertyLink`

Added support for `PropertyXLink` by adding a combo box to switch between
documents for linking to external objects.

Change list view to tree view in order to support linking into a sub-object.

## `View3DInventorViewer`

The parent class is changed from `Gui::SelectionSingleton::ObserverType` to
`Gui::SelectionObserver`, so that `SelectionView` can obtain the full hierarchy
information of the selection.

Added a _On-Top_ `Coin3D` group node in the scene graph to support  the new
`ViewProvider::OnTopWhenSelected` property, and object highlight (which is also
made on-top) on mouse over tree view item. The logic is implemented in function
`checkGroupOnTop()`, and called in response to various selection change event.
The 3D rendering of the on-top object relies on a new `Coin3D` node class
`SoFCPathAnnotation`, which is similar to `Coin3D SoAnnotation` but is capable
of rendering a specific `Coin3D` path, in order to show the correct object
hierarchy.

The following APIs are added to support [in-place editing](#user-content-in_place_edit).

* `setupEditingRoot()`, called to initiate in-place editing.

* `resetEditingRoot()`, revert scene graph changes after in-place editing.

* `setEditingTransform()`, setup the editing transformation matrix.

* `getPointOnRay()`, this helper function is moved from `Gui::ViewProvider`
  here because of the newly added in-place editing may move all its children
  node into the editing root node of `View3DInventorViewer`. The current only
  user of `getPointOnRay()`, `SketcherGui::ViewProviderSketch`, supports
  in-place editing this way.

## `SoFCUnifiedSelection`

This class is the key to make selection context to work in 3D view. The
selection and pre-selection logic is refactored and moved from `doAction()`
into dedicated functions, `setSelection()` and `setHighlight()`. The
implementation looks for the first view provider root node found in the
`Coin3D` path inside a `SoPickedPoint`, and calls that view provider's
`getElementPicked()` to obtained selection hierarchy context. See
[here](Link#selection-logic-flow) for more details.

New support for `PickedList` is implemented by exploring the picked point list
from `Coin3D SoHandleEventAction`.

## `SoFCSelection`

This is the legacy class for handling selection in the past. It is still used
to handle selection of non-`Part` derived objects, such as
`ViewProviderOriginFeature`. In order to have the same selection context
support as `SoFCUnifiedSelection`, the selection handling logic is skipped in
its `handleEvent()` function, but instead, handled inside
`SoFCUnifiedSelection` like everyone else, thus truly _unifies_ 3D view
selection logic. A new attribute, `useNewSelection` is added to control this
behavior. 

## `SoFCSelectionRoot`

This class, derived from `Coin3D SoSeparator`, stores the actual selection
context, and allows other shape classes to dynamically look up the current
selection context, and property render selection and preselection highlight.
Three shape classes, `SoBrepPointSet`, `SoBrepEdgeSet`, and `SoBrepFaceSet`
(also `SoFCSelection`, for selection rendering of non-`Part` derived object)
are modified to support context-aware selection [rendering](Link#user-content-rendering).

## `TaskElementColor`

A new task class to allow user specifying element colors. It uses the new API
`ViewProvider::get/setElementColors()`. It is currently used by
`ViewProviderLink` to override element color, and also
`PartGui::ViewProviderPartExt`, which replaces the original `TaskFaceColors`.

# `Part`

<a name="getshape"></a>The `Part` module does the actual geometry modeling.
It has been modified so that other modules can obtain geometry shape through
`Link`. The most import function here is a static helper function
`Part::Feature::getTopoShape()`. It allows one to obtain a geometry shape from
a object that is not derived from `Part::Feature`, including `Link`, and group
type object. For group object, `getTopoShape()` relies on the new
`DocumentObject:getSubObjects()` function to obtain the children object, and
creates a compound shape on demand. To improve performance, it uses internal
shape caches to reduce number of rebuilds.

`getTopoShape()` is exposed to Python as `Part.getShape()`. Please checkout
its doc string for more details.

# `PartDesign` and `Sketcher`

`PartDesign` and `Sketcher` modules are modified to support the new 
[[Topological Naming]] functionality. 

# `Draft`

New class `_DraftLink`, along with its companion `_ViewProviderDraftLink`, are
added As a demonstration of how to use `App::LinkBaseExtension` in Python to
implement `Link` type object. `_Array` and `_PathArray` are modified to derive
from this class to support linked array. A new attribute is added to turn on
link function when the object is created. Linked array behave almost the same
as original array object, with the additional benefit of external link.
Moreover, when `ShowElement` property is turned on, each element of the array
is revealed as a child object to let user customize the placement and material.

# `Import`

Class `ImportOCAF2` and `ExportOCAF2` has been added as a new implementation of
`STEP` importer/exporter, which uses `Link` and `LinkGroup` to greatly improve
performance of handling nested assembly structures. Proper support for edge
and visibility has been added, as well as support of assembly component
instance styling override using `Link's` color and visibility override
function. New options are exposed to `STEP` import/export preference page,
including options to access original upstream importer. 

# Spreadsheet

There is a major refactor of the `Expression Engine`, along with the
`Spreadsheet` module. You can find more details at [[Expression and Spreadsheet]].

