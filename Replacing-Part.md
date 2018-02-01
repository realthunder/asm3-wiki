Continue from the Multi-join assembly we created in previous 
[[tutorial|Common-Operations#create-a-super-assembly-with-external-link-array]]. 
Suppose we want to change the sub-assembly with another part. The screencast
below shows how to do it.

[[images/replace-part.gif]]

As explained in previous [[section|Concepts#Element]], all geometry element
references used in any constraints will be referenced through some auto created
`Element` object. Find out those elements, and give them some meaningful name.
When you rename some `Element` object that are referenced by some constraint,
or high level `Element` objects, all involved subname references will be
automatically updated. Be careful, though. If you are editing an external
assembly that are referenced in some other super assembly file that are not
currently opened in FreeCAD, you will obviously break the link reference. 

Now, to replace the sub-assembly, manually create the corresponding elements
with selection, drag and drop, and give them the same names. Because the
original sub-assembly is brought in by a `Link` object, simply re-point the
`Link` to the replacement part, and re-solve the constraint system. Done!

Note that, in the screencast, we didn't rename the first two elements. It is
because that those elements are used internally by the original sub-assembly. It
is not involved in the parent assembly. If you modified the geometry model of
the sub-assembly, then you should double check all the elements for any geometry
topological name changes.


