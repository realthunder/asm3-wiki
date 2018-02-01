Assembly3 supports multiple constraint solver backend. The user can choose
different solver for each `Assembly`. The type of constraints available may be
different for each solver. At the time of this writing, two backend are
supported, one based on the solver from SolveSpace, while the other uses SymPy
and SciPy, but is modeled after SolveSpace. In other word, the current two
available solvers supports practically the same set of constraints and should
have very similar behavior, except some difference in performance. 

Assembly3 exposed most of SolveSpace's constraints. If you want to know more
about the internals, please read 
[this document](https://github.com/realthunder/solvespace/blob/python/exposed/DOC.txt)
from SolveSpace first. The way Assembly3 uses the solver is that, for a given
`Assembly`,

* Create free parameters corresponding to the placement of each movable child
  feature, that is, three parameters for position, and four parameters for its
  orientation (quaternion).
* For each constraint, create SolveSpace `Entities` (i.e. points, normals,
  rotations, etc) for each geometry element as a transformation of its owner
  feature's placement parameters.
* Create SolveSpace `Constraints` with the `Entities` created in the previous
  step.
* Ask SolveSpace to solve the constraints. SolveSpace formulate the constraint
  problems as a non-linear least square minimization problem, generates
  equations symbolically, and then tries to numerically find a solution for the
  free parameters.
* The child features placements are then updated with the found solution.

For nested assemblies, Assembly3 will always solve all the child assemblies
first before the parent.

One thing to take note is that, SolveSpace is a numerical solver, which means
it is sensitive to initial conditions. In other word, you must first roughly
align two features according to their constraints, or else the solver may not
be able to find an answer. Assembly3 has extensive support of easy manual
placement of child feature. See the following section for more details.

Assembly3 provides several new constraints that are more useful for assembly
purposes. Most of these constraints are in fact composite of the original
constraints from SolveSpace, except for the `Lock` constraint. Some more common
constraints are available as toolbar buttons for easy access, while all
constraint types are accessible in property editor. Each `Constraint` object
has a `Type` property, which allows the user to change the type of an existing
constraint. Not all constraints require the same types of geometry elements,
which means that changing the type may invalidate a constraint. The tree view
will mark those invalid constraints with a red exclamation mark. Hover the
mouse over those invalid items to see the explanation. Just follow the
instruction to manually correct the element links.

Special mention about the `Lock` constraint, which is used to lock the
placement of a related child feature. Like all other constraints, `Lock`
constraint, as a container, only accepts geometry element drag and drop. If you
drop in a `Vertex` or linear `Edge`, then the owner feature is allowed to be
rotated around the geometry element. If you drop in a non-linear `Edge` or
`Face`, then the feature is completely fixed within the owner assembly with
this constraint. If no `Lock` constraint is found in an `Assembly`, then the
feature that owns the first element of the first constraint is fixed.
