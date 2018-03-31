# Overview

This article details the actual algorithm of generating the topological names
using the shape modeling history. The history information is provided by `OCCT`'s 
[BRepBuilderAPI_MakeShape](https://www.opencascade.com/doc/occt-7.0.0/refman/html/class_b_rep_builder_a_p_i___make_shape.html)
and its derivatives. They provides the mapping from the input geometry elements
to the newly generated or modified elements in the resulting shape. However, As
`OCCT` being `OCCT`, not all of `OCCT's` maker classes follow the same interface.
There are exceptions, which is why `TopoShape` uses an abstract class
`TopoShape::Mapper` as the mapping interface. Special handling is required for
the following `OCCT` maker,

* `BRepBuilderAPI_Sewing` (used in `makEOffset`, etc.)
* `BRepOffsetAPI_ThruSections` (used in `makELoft`, etc.)

# Algorithm

The core function to generate element names is `TopoShape::makESHAPE()`. At
a high level, the algorithm can be described as using four steps to try to name
as name elements as possible.

1. Find any unchanged and named elements that appear in both the input and out
   shape, and just copy those names over to the new shape.
2. Assign names for generated and modified elements by querying the `Mapper`.
3. For all unnamed elements, assign an indexed based name using its upper named
   element. For example, if a face is named `F1`, its unnamed edges will be named
   as `F1;U1`, `F1;U2`, etc. In this step, an element may be assigned multiple
   names, if it appear in more than one named upper elements. This is to improve
   the name stability of the next step.
4. For all remaining unnamed elements, use the combination of lower named
   elements as its name. For example, if a face has all its edges named, then
   the face will be named as something like `(edge1,edge2,edge3);L`.

A simplified version of the algorithm is presented with the following pseudo
Python code,

```python
def makESHAPE(result_shape, mapper, input_shapes, opcode):

    # first, map any unchanged elements into the result shape

    for shape in input_shapes:
        result_shape.mapSubElement(shape)

    # collect names from input shapes that generates or modifies elements in
    # the result shape

    # a map of map to store the names
    names = defaultdict(dict)

    for element_type in ('Vertex','Edge','Face'):
        for shape in input_shapes:
            for i,sub_shape in enumerate(shape.getSubShapes(element_type)):
                # get the mapped element name of this sub shape. 'True' here 
                # means we are asking for reverse mapping, i.e. query the mapped
                # name using the index name
                mapped_name = str(shape.Tag) + '_' + 
                        shape.getElementName(element_type+str(i+1),True)

                # get all the new_sub_shape that is modified from sub_shape
                for k,new_sub_shape in enumerate(mapper.getModified(sub_shape)):

                    # get the topological indexed names of this new sub shape,
                    # i.e. Vertex1, Edge2, Face3, etc.
                    index_name = result_shape.getIndexName(new_sub_shape)

                    # stores a map entry from new_element_index ->
                    # input_mapped_element_name Also stores the input shape's
                    # Tag, and the order index for disambiguation, because a new
                    # element may be modified from more than one input elements
                    names[index_name][mapped_name] = k+1

                # now, do the same for new_sub_shape that are generated from sub_shape
                for k,new_sub_shape in enumerate(mapper.getGenerated(sub_shape)):
                    index_name = result_shape.getIndexName(new_sub_shape)

                    # we use a negative order index to indicate its a generated element
                    names[index_name][mapped_name] = -(k+1)

    # Construct names from the information collected in the above steps                            
    for index_name,name_map in names.items():
        postfix=None
        for n,(mapped_name,index) in enumerate(name_map.items()):
            if n==0:
                # special treatment of the first mapped element name from the
                # input shape. We shall use this as prefix for advanced
                # application, e.g. to find all shapes that are generated or
                # modified from a common source.
                first_tag = mapped_name.split('_')[0]
                first_name= mapped_name[len(first_tag)+1:]
                first_index = index
                name_type = 'M' if index>0 else 'G'
                continue

            # name type tells us if this name is for a modified (='M') or
            # generated (='G') shape. Although it shouldn't happend, but we are
            # prepared to accpet a shape that is both generated and modified (='MG')
            if (name_type=='M' and index<0) or \
               (name_type=='G' and index>0):
               name_type = 'MG'

            # throw all the rest of names into a pair of brackets
            if n==1:
                postfix = '('
            else:
                postfix += ','
            postfix+= mapped_name + ':' + str(index)

        if postfix:
            postfix += ')'

        result_shape.setElementName(
                    index_name,
                    name = first_name,
                    prefix = opcode + first_tag + '_',
                    postfix = ';' + name_type + str(abs(first_index)) + postfix)

        # Suppose Face1 of result shape is generated by Edge2 of shape tag 1,
        # and Edge3 of shape tag 2, and Edge4 of shape tag 3, with opcode 'XTR',
        # it will be named as 
        #
        #    XTR1_Edge2;G1(2_Edge3:2,3_Edge4:2)
        #
        # The 1 in XTR1 the owner shape tag of Edge2.
        #
        # The 1 in G1 is the shape tag 1 Edge2 generation index. This edge may
        # generate more than one shape. The index is here for disambiguation. In
        # the actual algorithm, we omit any index of '1' to reduce some
        # verbosity.
        #
        # The 2 in 2_Edge3 is the owner shape tag of Edge3.
        #
        # The :2 in 2_Edge3:2 is the generation index of shape tag 2 Edge3.


    # Now, the reverse pass. Starting from the highest level element, i.e.
    # Face, for any element that are named, assign names for its lower unamed
    # elements. For example, if Edge1 is named E1, and its vertexes are not
    # named, then name them as E1;U1, E1;U2, etc.
    #
    # In order to make the name as stable as possible, we may assign multiple
    # names (which must be sorted, because we may use the first one to name
    # upper element in the final pass) to lower element if it appears in
    # multiple higher elements, e.g. same edge in multiple faces.

    names.clear()
    for upper_type,element_type in (('Face','Edge'),('Edge','Vertex')):
        for i,sub_shape in enumerate(result_shape.getSubShapes(upper_type)):
            # get the mapped element name of the new sub shape. 'True' here 
            # means we are asking for reverse mapping, i.e. query the mapped
            # name using the index name.
            index_name = upper_type+str(i+1)
            upper_mapped_name = result_shape.getElementName(index_name,True)

            # skip any non-mapped names
            if upper_mapped_name == index_name:
                continue

            n=0
            for element_shape in sub_shape.getSubShapes(element_type):
                # get the indexed name of this lower element in the result shape
                index_name = result_shape.getIndexName(element_shape)

                # skip any named elements
                if result_shape.getElementName(index_name,True) != index_name:
                    continue;

                n = n+1
                names[index_name][upper_mapped_name] = n

        # assign the actual names
        for index_name,name_map in names:
            for upper_mapped_name,index in name_map.items():
                result_shape.setElementName(
                                index_name,
                                name = upper_mapped_name,
                                prefix = opcode + '_',
                                postfix = ';U' + str(index))

    # Finally, the forward pass. For any elements that are not named, try
    # construct its name from the lower elements

    for element_type,lower_type in (('Edge','Vertex'),('Face','Edge')):
        for i,sub_shape in enumerate(result_shape.getSubShapes(element_type)):
            index_name = element_type + str(i+1)
            
            # skip any named element
            if result_shape.getElementName(index_name, True) != index_name:
                continue

            names = set()
            for element_shape in sub_shape.getSubShapes(lower_type):
                # get the indexed name of this lower element in the result shape
                index_name = result_shape.getIndexName(element_shape)

                mapped_name = result_shape.getElementName(index_name,True)
                if mapped_name == index_name:
                    # only name the upper element if all lower elements are
                    # named
                    names.clear()
                    break

                # strip common prefix to reduce verbosity
                if mapped_name.startswith(opcode+'_'):
                    mapped_name = mapped_name[len(opcode)+1:]
                names.insert(mapped_name)

            if not names:
                continue

            # For exmaple, if opcode is XTR, and a face with edges Edge1,
            # Edge2, Edge3 will be named as XTR_(Edge1,Edge2,Edge3);L

            result_shape.setElementName(index_name,
                                        name = '(' + ','.join(names) + ')',
                                        prefix = opcode + '_',
                                        postfix = ';L')

```

# Test Example

Let's take an example to see the new topological naming in action. 

[[images/topo-naming-1.gif]]

We make a fusion with a simple `Cube` and `Cylinder`. Note that, I have turned
off string hash in advance to show the raw topological names. You can do this
by typing in the console,

```python
App.ActiveDocument.UseHasher = False
```

See the following section for more details of the string hasher.

You can see that the modified top Cube face is named as

```
FUS1_Face6;M2
 | |   |  | |
 | |   |  | --modification index, means the input element has at least 
 | |   |  |   been modified into two elements in the resulting shape
 | |   |  | 
 | |   |  --';M' is the modification maker
 | |   |
 | |   |
 | |   --input element name
 | |
 | --Cube's object ID
 |
 --fusion op code
```

And the modified top cylinder face is named as,

```
FUS2_Face2;M2
 | |   |  | |
 | |   |  | --modification index, means the input element has at least 
 | |   |  |   been modified into two elements in the resulting shape
 | |   |  | 
 | |   |  --';M' is the modification maker
 | |   |
 | |   |
 | |   --input element name
 | |
 | --Cylinder's object ID
 |
 --fusion op code
````

The middle smaller face is a modification from two input elements, 

```
FUS1_Face6;M(2_Face2)
   |     |   |     |
   -------   -------
      |         |
      |         --The second modification source is included in the postfix,
      |           i.e. Face2 from shape tag 1 (the Cylinder)
      |
      --The first modification source appears in the main name part, i.e. Face6
        from shape tag 1 (the Cube). The order of the source shape is sorted by
        their tag and element name.
```

Now, we turn on model refine. And the top three faces are merged into one face,
and its name got a lot longer. 

[[images/topo-naming-2.gif]]

```
RFI_FUS1_Face6;M(2_Face2);M(FUS1_Face6;M2,FUS2_Face2;M2)
 |  |                   ||  |           | |           |
 |  |                   ||  |           | -------------
 |  |                   ||  -------------       |
 |  ---------------------|        |             --The modified top cylinder
 |            |          |        |               face in the previous step
 |            |          |        |
 |            |          |        --The modified top face of the cube
 |            |          |
 |            |          --The modifiction marker
 |            |
 |            --The face modified from the top faces of both the cylinder and
 |              the cube in the previous step
 |
 --Refine op code
```

<a name="fillet"></a>Next, we create a fillet by selecting an edge, `FUS1_Edge5.Edge5`. As you can see,
when we turned off `Refine` in the fusion, the edge index jumped from 5 to 9, but
its mapped element name is still `FUS1_Edge5`, 1 being the Cube's object ID, i.e.
`Edge5` from Cube 1.

[[images/topo-naming-3.gif]]

To further showcase a more advanced application of the new naming scheme, we
add a new cylinder to the fusion in the following screen cast. 

[[images/topo-naming-4.gif]]

The fillet is now broken, because the originally selected edge is gone. Its
name will not be reused like old index based names.You can see from the above
screen cast that the original edge is modified, and its name has changed to
`FUS1_Edge5;M`, with an additional modification marker `;M`. For each invalid
edge found, the fillet editing tool will construct a prefix by appending the
modification marker to the new style mapped name of the edge, and then query
the shape's element map for any element starting with this prefix. This is how
the fillet editing tool can auto select the related edge as shown in the screen
cast. If no edge is found this way, the tool will construct another prefix by
trimming any existing modification marker found in the mapped name, and query
the element map again. So, if we are to remove the cylinder from the fusion,
the editing tool will be able auto select the original edge, too.

The following screen cast shows various ways to mess around the fusion, and
the fillet tool can always guess the correct edge.

[[images/topo-naming-5.gif]]

Note that the fillet feature relies on `PropertyLinkSub` to obtain the edge
reference from the mapped element name. If the model changed such that the
mapped element is gone, or in other word, the mapped element name is changed,
the fillet will report error as invalid edge link. It is the fillet editing
tool that does the guessing by querying various prefixes. This guessing is not
always reliable, and requires user attention, which is why it is not automated
inside fillet feature.

# Other Details

The length of the generated names will be kept manageable by the 
[[string hasher|Topological-Naming#string-hasher]]. And we will also be adding
[[version string|Topological-Naming#element-map-versioning]] to account for
any changes is the name generation process.

# Overhead

As you may have expected from the above description, the element naming algorithm
is complex, and the number of names generated is potentially big. There bound to
have some overhead in terms of both the computation time and file size. Here is
a non-formal test result to show the negative impact. Note that the 
[[string hasher|Topological-Naming#string-hasher]] is turned on for this test. 

A model file with 148 objects, with only one root level Part workbench object.
The dependency graph is shown below.

[[images/topo-name-test-model.png]]

|    | Without element map | With element map |
| -- | ---------------- | ------------------- |
| Total recomupation time | 40 s | 50 s |
| Uncompressed File Size | 17.8 MiB | 25.2 MiB |
| Compressed (level 3) size | 2.8 MiB | 4.1 MiB |

