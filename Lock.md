The `Lock` constraint
![Lock](../raw/master/freecad/asm3/Gui/Resources/icons/constraints/Assembly_ConstraintLock.svg?sanitize=true),
is used to lock the placement of a related child feature. Like all other constraints, `Lock`
constraint, as a container, only accepts geometry element when drag and drop. If you
drop in a `Vertex` or linear `Edge`, then the owner feature is allowed to be
rotated around the geometry element. These types of locking is more useful
when creating [skeleton sketch](Create-Skeleton-Sketch) If you drop in a non-linear `Edge` or
`Face`, then the feature is completely fixed within the owner assembly with
this constraint. If no `Lock` constraint is found in an `Assembly`, then the
feature that owns the first element of the first constraint is fixed. When
you try to move a _Locked_ part, all the other parts will re-settle around
it. You can prevent accidentally move a locked part by toggle the 
![LockMover](../raw/master/freecad/asm3/Gui/Resources/Assembly_LockMover.svg?sanitize=true),
which will prevent the mover button from activating.


