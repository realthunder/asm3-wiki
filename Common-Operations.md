# Create a Simple Assembly with a Constraint

* Start FreeCAD and create a new document
* Switch to the `Part` workbench and create a `Cube` and a `Cylinder`
* Switch to the `Assembly3` workbench, click
  ![AddAssembly](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_New_Assembly.svg?sanitize=true) 
  to create a new assembly
* Select both the `Cube` and the `Cylinder` and drag them into the new assembly
* Select any face of the `Cylinder` or `Cube`, and click
  ![Move](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_Move.svg?sanitize=true) to
  activate part manual movement. Click any arrow to drag the `Cylinder` on top
  of the `Cube`, press Esc to leave part movement (Alternative:
  right-click `Assembly` in the tree view and click `Finish editing`)
* Select the top face of the `Cube` and (while holding the `CTRL` key) select the bottom
  face or edge of the `Cylinder` and then click
  ![AddCoincidence](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintCoincidence.svg?sanitize=true)
  to create a plane coincidence constraint.
* Finally, click
  ![Solve](../raw/master/freecad/asm3/Gui/Resources/icons/AssemblyWorkbench.svg?sanitize=true)
  to solve the constraint system.

[[images/simple.gif]]

You can click ![Auto recompute](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_AutoRecompute.svg?sanitize=true) 
to enable auto-solving with any changes in constraint.

After a new constraint is created, the selected elements will be highlighted in
red. You can easily change the color of individual constraining elements under
its view object property page. Or, set the color of the entire constraint by
changing the color of constraint object itself. Make sure you set the
`OverrideMaterial` view property to `True`. 

In case you find that the constraining element highlight obscure the assembly
3D view, you can enable the _Auto Element Visibility_ feature by clicking
![AutoElementVis](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_AutoElementVis.svg?sanitize=true).
When enabled, all constraining elements will be hidden by default. You can
reveal them by selecting any constraint or constraint element object in the
tree view.

Now, save this document with whatever name you like.

# Create a Super Assembly with External Link Array

We are going to build a multi-joint _thing_ using the above assembly as the
base part.

* Create a new document, and save it to whatever name you like. Yes, you need
  to save both the link and linked document at least once for external linking
  to work, because `PropertyXLink` need the file path information of both
  document to calculate relative path.
* Make sure the current active 3D view is the new empty document. Now, in the
  tree view, select the assembly we just created previously, and then hold on
  `CTRL` key and right click the new document item in the tree view, and select
  `Link actions -> Make link`. A `Link` will be created that brings the
  assembly into the new document. You probably need to click `Fit content`
  button (or press `V,F` in 3D view) to see the assembly.
* Select the link in the tree view, and change the `ElementCount` property to
  four. Now you have four identical assemblies.
* Create a new assembly, and then drag the link object into it.
* Select any face of any `Cube`, click
  ![Move](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_Move.svg?sanitize=true)
  and click any arrow to drag to spread out the parts. Click the edges of the
  movement sphere to turn the part. Press `Esc` to leave part movement.
* Select any face of the left most `Cube` in 3D view, and click
  ![Lock](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintLock.svg?sanitize=true)
  to lock the left most sub assembly.
* Orient the parts whatever you like. Select two face from any two assembly,
  and create a plane coincidence constraint. If you've enabled _auto
  recompute_, then the two assembly will now be snapped together
* Do the same for the rest of the parts.

[[images/super.gif]]

Now that we've made this multi-joint thingy, try to save this document, and
FreeCAD will ask if you want to save the external document, too. If you answer
yes, then FreeCAD will take care of ordering, and save the external document
first before linking document.

Close both documents. Open the multi-joint assembly document, FreeCAD will
automatically open any externally referenced documents, too. If you close the
external document while leaving the linking document open, all externally
linked object will vanish from 3D view. Open external document again and the
objects will re-appear. This allows you to easily swap in a replacement
document for whatever reason. But, it demands the replacement document having
an object of the same internal name as the original linked one. You can of
course, easily re-assign the link to any other object in the opened documents.
Just use the property editor, click the edit button of `LinkedObject` property.
In the editor window, select the desired document in the drop list, and then
select the desired object. But now, you need to make sure the new linked object
has the same element interface, or else the constraints will be broken.

A few more words about link array. Assembly3 normally treats any object added
to its part group as a stand alone entity that can be moved as a whole.
However, it has special treatment for a link array object. Each array element
will be treated as separate entities, that can be constrained and moved
individually. If you actually want to add the array as an integral part, simply
wrap the array inside a dummy assembly without any constraint, and add that
assembly instead into the parent assembly. 

By the way, the `Draft` workbench now has two variation of link array, the
`LinkArray` and `LinkPathArray`, which provide the same functionality as
`Draft` `Array` and `PathArray`, but use link to provide duplicates. Those link
arrays, by default, do not show individual element in tree view. You can still
access each element through `subname` reference as usual. Having less
objects can improve document saving and loading performance. It is particularly
noticeable if you have large amount of array elements. You can, however, show
the array element at any time by toggle property `ShowElement`. Once the
elements are visible, they can be moved independently by change their
placements.

# Add/Modify Element and ElementLink

It is quite easy to directly create a new constraint as shown above, with all
involved `Element` and `ElementLink` being taken care of for you by Assembly3.
It is also straightforward to manually add new or modify existing `Elements`
and `ElementLink`. Simply select a geometry element in 3D view, and its
corresponding owner object will be selected in the tree view (Remember to turn
on _Sync selection_ option in tree view as mentioned before). You can then drag
the selected item to the `ElementGroup` of an `Assembly` to create a new
`Element`, or to a `Constraint` to add an `ElementLink`. You can modify an
existing `Element` or `ElementLink` by simply dragging the item onto an
existing item of `Element` or `ElementLink`. Checkout [[this|Replacing-Part]]
tutorial for demonstration.

# Part Move

Assembly3 has extensive support of manual movement of nested assembly. In 3D
view, select any geometry element (Face, Edge) that belongs to some assembly,
and click ![Move](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_Move.svg?sanitize=true) 
to activate part dragging. The dragger will be centered around the selected
geometry element. In case of multi-hierarchy assemblies, you will be dragging
the first level sub-assembly of the top-level assembly. If you want to drag
intermediate sub-assembly instead, add that assembly as the second selection
(`CTRL` select) before activating part move. 

If you have enabled ![AutoRecompute](../raw/master/freecad/asm3/Gui/Resources/icons/Assembly_AutoRecompute.svg?sanitize=true),
any movement of the sub-assembly will cause the parent assembly to auto
re-solve its constraints, as shown below. Because there are too many degree of
freedom left, hence many possible solutions of the constraint system, the
movement of the multi-joint assembly is jerky. Besides, the part move command
is very complicated, and probably need a lot more work to make it perfect. In
case when the parts are moved such that they stuck in some invalid position, as
you can see from the screen cast below, simply `CTRL+Z` to undo the movement.
Every time you release the mouse button, a transaction will be committed, so
that you can undo/redo the previous mouse drag. You can also temporary bypass
recomputation by holding `CTRL` key while dragging.


[[images/move.gif]]

# Import External Assembly

In some cases, it will be easier to distribute your multi-hierarchy assembly as
a single self-contained document. FreeCAD core provides a convenient command to
help with this otherwise not so trivial task. Simply right-click any item in
the document you want to distribute, and select `Link actions -> Import all
links`, and that's all. Click
![Solve](../raw/master/freecad/asm3/Gui/Resources/icons/AssemblyWorkbench.svg?sanitize=true)
to see if every thing is okay. You can of course selectively import any object
you want. Simply right click that item and select `Link actions -> Import
link`.

