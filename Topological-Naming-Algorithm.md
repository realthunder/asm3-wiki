##### Table of Contents
- [Overview](#overview)
- [Algorithm](#algorithm)
- [Test Example](#test-example)
- [Hashing Element Name](#hashing-element-name)
- [Tracing Model History](#tracing-model-history)
- [Element Coloring](#element-coloring)
- [Overhead](#overhead)

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
as many elements as possible.

1. Find any unchanged and named elements that appear in both the input and out
   shape, and just copy those names over to the new shape.
2. Assign names for generated and modified elements by querying the `Mapper`.
3. For all unnamed elements, assign an indexed based name using its upper named
   element. For example, if a face is named `F1`, its unnamed edges will be named
   as `F1;:U1`, `F1;:U2`, etc. In this step, an element may be assigned multiple
   names, if it appear in more than one named upper elements. This is to improve
   the name stability of the next step.
4. For all remaining unnamed elements, use the combination of lower named
   elements as its name. For example, if a face has all its edges named, then
   the face will be named as something like `(edge1,edge2,edge3);:L`.

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
                mapped_name = str(shape.Tag) + '_' + \
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

                # now, do the same for new_sub_shape that are generated from
                # sub_shape
                for k,new_sub_shape in enumerate(mapper.getGenerated(sub_shape)):
                    index_name = result_shape.getIndexName(new_sub_shape)

                    # we use a negative order index to indicate its a generated
                    # element
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
            # generated (='G') shape. Although it shouldn't happen, but we are
            # prepared to accept a shape that is both generated and modified
            # (='MG')
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

        first_index = abs(first_index)
        postfix = ';:{}{}{};{};:T{}:{}:{}'.format(name_type,
                                                  first_index,
                                                  postfix,
                                                  opcode,
                                                  first_tag,
                                                  len(first_name),
                                                  index_name[0])

        result_shape.setElementName(index_name, name=first_name,postfix=postfix)

        # Suppose Face1 of result shape is generated by Edge2 of shape tag 1,
        # and Edge3 of shape tag 2, and Edge4 of shape tag 3, with opcode 'XTR',
        # it will be named as 
        #
        #    Edge2;:G1(2_Edge3:2,3_Edge4:2);XTR;:T1:4:F
        #
        # Marker ; is used as separator for different components of the name.
        # Special marker ;: is reserved by TopoShape to mark any internal used
        # components, to distinguish with user supplied op code.
        #
        # ;:G indicates the element is generated. The following 1 is the
        # generation index. Because any source elements may generate more than
        # one shape, the index is here for disambiguation. In the actual
        # algorithm, we omit any index of '1' to reduce some verbosity.
        #
        # The 2 in 2_Edge3 is the owner shape tag of Edge3.
        #
        # The :2 in 2_Edge3:2 is the generation index of shape tag 2 Edge3.
        #
        # The ending ;:T1 marks the tag (1 in this case) of the first source
        # shape, and the following :4 is the length of the first source element
        # name. And the next :F marks the element type of this name, to make
        # sure we don't lost element type information if the element is gone due
        # to model changes. This ending mark, starting with ;:T, is very
        # important for tracing back element evolving history.


    # Now, the reverse pass. Starting from the highest level element, i.e.
    # Face, for any element that are named, assign names for its lower unnamed
    # elements. For example, if Edge1 is named E1, and its vertices are not
    # named, then name them as E1;:U1, E1;:U2, etc.
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
                # get the indexed name of this lower element in the result
                # shape
                index_name = result_shape.getIndexName(element_shape)

                # skip any named elements
                if result_shape.getElementName(index_name,True) != index_name:
                    continue;

                n = n+1
                names[index_name][upper_mapped_name] = n

        # assign the actual names
        for index_name,name_map in names:
            for upper_mapped_name,index in name_map.items():
                postfix = ';:U{};{}'.format(index,opcode)
                # In case the upper name has a trailing ;:T marker, extract the
                # source tag
                upper_tag = getTagFromName(upper_mapped_name)
                if upper_tag:
                    postfix += ';:T{}:{}:{}'.format(
                            upper_tag,len(upper_mapped_name),index_name[0])
                result_shape.setElementName(
                                index_name,
                                name = upper_mapped_name,
                                postfix = postfix)

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
                # get the indexed name of this lower element in the result
                # shape
                index_name = result_shape.getIndexName(element_shape)

                mapped_name = result_shape.getElementName(index_name,True)
                if mapped_name == index_name:
                    # only name the upper element if all lower elements are
                    # named
                    names.clear()
                    break

                names.insert(mapped_name)

            if not names:
                continue

            # Construct the element name. For example, if opcode is XTR, and a
            # face with edges Edge1, Edge2, Edge3 will be named as
            # Edge1;:L(Edge2,Edge3);XTR

            names = list(names)
            first_name = names[0]
            postfix = ';:L({});{}'.format(','.join(names[1:]),opcode)

            # In case the lower name has a trailing ;:T marker, extract the
            # source tag
            lower_tag = getTagFromName(first_name)
            if lower_tag:
                postfix += ';:T{}:{}:{}'.format(
                        lower_tag,len(first_name),index_name[0])

            result_shape.setElementName(index_name,
                                        name = first_name,
                                        postfix = postfix)

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
Face6;:M2;FUS;:T1:5:F
 |     |   |   |  | |
 |     |   |   |  | -- The type of this element, 'F' for 'Face'
 |     |   |   |  |
 |     |   |   |  -- The length of the source element name. This is used to 
 |     |   |   |     trace back the model history. In this case, the first
 |     |   |   |     five characters gives the original element name of 'Face6'.
 |     |   |   |
 |     |   |   -- ';:T' is the marker for tag of owner shape of the original
 |     |   |      element. The following '1' is the actual tag value. In this
 |     |   |      case, it is the Cube's object ID.
 |     |   |
 |     |   -- op code
 |     |
 |     --';:M' is the modification maker, and the following '2' is the 
 |        modification index, in case the source element modifies into more
 |        than one elements.
 |
 -- The source element name. The length is recorded at the end of the mapped
    name. In case a string hasher is used. This will be the hashed name, and
    the length is the length of the name after hashing.
```

The modified top cylinder face is similarly names as `Face2;:M2;FUS;:T2:5:F`

The middle smaller face is a modification from two input elements, and is named
as `Face6;:M(Face2;:T2:5:F);FUS;:T1:5:F`. Ignoring what's inside the bracket, we get
`Face6;:M();FUS;:T1:5`, and this is similarly named as the first modified face
originated from the `Cube`. The `Face2;:T2:5:F` inside the bracket is the second source
name and its tag and name length. In case there are more sources, it will be
included in the bracket separated by `,`.

Next, we turn on model refine.

[[images/topo-naming-2.gif]]

The top three faces are merged into one face, and its name got a lot longer. 
```
Face2;:M2;FUS;:T2:5:F;:M(Face6;:M(Face2;:T2:5:F);FUS;:T1:5:F,Face6;:M2;FUS;:T1:5:F);RFI;:T2:21:F
```

This name is difficult for human to read, but easy for program to parse. The ending
`;:T2:21:F` tells the tag, the name length of the first source element, and the element type. With that, we
can easily find out the source element name being the first 21 characters,
`Face2;:M2;FUS;:T2:5:F`. Now, strip out the first source name and the trailing
tag and length information, we have

```
;:M(Face6;:M(Face2;:T2:5:F);FUS;:T1:5:F,Face6;:M2;FUS;:T1:5:F);RFI
 |  |                                 | |                   |   |
 |  |                                 | |                   |   -- op code, indicates
 |  |                                 | ---------------------      refine operation
 |  |                                 |           |
 |  -----------------------------------           -- The third source element name
 |                 |
 |                 -- The second source element name
 |
 -- The modification marker
```

<a name="fillet"></a>Next, we create a fillet by selecting an edge,
`Edge5;:T1:5:E.Edge5`. The trailing `.Edge5` indicates the old style indexed
element name. As you can see, when we turned off `Refine` in the fusion, the
edge index jumped from 5 to 9, but its mapped element name is still
`Edge5;:T1:5:E`, with `1` being the Cube's object ID, i.e. `Edge5` from Cube 1.

[[images/topo-naming-3.gif]]

To further showcase a more advanced application of the new naming scheme, we
add a new cylinder to the fusion in the following screen cast. 

[[images/topo-naming-4.gif]]

The fillet is now broken, because the originally selected edge is gone. Its
name will not be reused like old index based names.You can see from the above
screen cast that the original edge is modified, and its name has changed from
`Edge5;:T1:5:E` into `Edge5;:M;FUS;:T1:5:E`. `TopoShape` offers function called
`getRelatedElements()` to help obtaining potential replacement of the missing
references. This is how the fillet editing tool can auto select the related
edge when activated, as shown in the screen cast. The algorithm of 
`getRelatedElements()` is described below,

```python
def getRelatedElements(shape,name):
    names = {}

    tag,length,type = getTagInfoFromName(name)
    if not tag:
        return

    # First, try to recover the first source element. 

    source = name[:length]

    # try to recover source name
    dehashed = deHash(name[:length])

    tag2 = getTagFromName(dehashed)
    if not tag2:
        # In case the tag number is not in the dehashed name, we
        # will append it manually, because when any name is mapped,
        # we will always append a tag as shown in the above naming
        # algorithm
        dehashed += ';:T{}:{}:{}'.format(tag,len(dehashed),type)

    # check if there is actually an element with this recovered name
    element = shape.getElementName(dehashed)
    if element != dehashed:
        names[dehashed] = element

    # Next, search any element that is modified from the given name.
    #
    # This step is simplified here as it does not include details of dealing
    # with possible hashing.
    
    modNames = shape.getElementWithPrefix(name + ';:M')
    names.update(modNames)

    # Finally, search any element that are modified from the same source of
    # the given name

    modNames = shape.getElementWithPrefix(source + ';:M')
    names.update(modNames)

    return names
```

The following screen cast shows various ways to mess around the fusion, and
the fillet tool can always guess the correct edge.

[[images/topo-naming-5.gif]]

Pay attention to the first modification, where we move the cylinder into the
middle of the `Cube` edge, the `Fillet` object seems to moved the fillet edge
downwards and missing the upper one. The reason is because, if an element is
modified into more than one new elements, its name will include a modification
index. The order of this index relies on what `OCCT` reports to us. So in this
case, what `Fillet` object sees is that the element reference still exists, and
it is the lower edge, because `OCCT` reports it as the first. We can arguably
include more information into the naming, e.g. add any existing lower element
names into the modified name, which will be the vertex name in this case. But
then, imaging that the edges got split into more than two segments, the middle
ones still requires the index to disambiguation, and problem still remains.

The fillet feature relies on `PropertyLinkSub` to obtain the edge
reference from the mapped element name. If the model changed such that the
mapped element is gone, or in other word, the mapped element name is changed,
the fillet will report an error of invalid edge link. It is the fillet editing
tool that does the guessing by querying various prefixes. This guessing is not
always reliable, and requires user attention, which is why it is not automated
inside fillet feature.

# Hashing Element Name

The length of the generated names will be kept manageable by the 
[[string hasher|Topological-Naming#string-hasher]]. In order to keep as much
information as possible, the hashing is performed in multiple segments of the
element name, in relation to how names are constructed. For example, the refined face
above will be hashed as follow,

```
Face2;:M2;FUS;:T2:5:F;:M(Face6;:M(Face2;:T2:5:F);FUS;:T1:5:F,Face6;:M2;FUS;:T1:5:F);RFI;:T2:21:F
|                   |   |                                                         |
|                   |   -----------------------------------------------------------
---------------------                                |
         |                                           -- second segment
         |
         -- first segment, i.e. the name of the first source element
```

The final hashed name is,

```
#a8;:M#a7;RFI;:T2:2:F
 |     |  |    |  | |
 |     |  |    |  | -- The element type, 'F' for 'Face'
 |     |  |    |  |
 |     |  |    |  -- The length is now changed to be the length of the hashed 
 |     |  |    |     name, i.e. '#a8'
 |     |  |    |
 |     |  |    -- The tag of the first source element stays the same.
 |     |  |
 |     |  -- op code stays the same
 |     |
 |     -- Second hash.
 |
 -- First hash, where 'a8' is a hex number indicating the hashed string ID,
    which can be used to recove the original string from the hash table if the
    original string length does not exceed hash table threshold.
```

# Tracing Model History

The naming algorithm is specifically designed to ease the recovery of the 
first source element name. `TopoShape` offers a function named `getElementHistory()`,
in both C++ and Python, to decode the input name and return the first source
element name and tag. `Part::Feature` offers a function with the same name that
recursively calls its `TopoShape's` `getElementHistory()` to give you a linear
history of any given element.

[[images/fillet-history.png]]

For example, with the above fillet object, to obtain the history of the top
refined face,

```python
for o in App.ActiveDocument.Fillet.getElementHistory('Face2'):
    print '{} {}'.format(o[0].Name, o[1:])

Fillet ('#21;:M;FLT;:T3:3:F', [])
Fusion ('#8;:M#7;RFI;:T2:2:F', ['Face2;:M2;FUS;:T2:5:F'])
Cylinder ('Face2', [])
```

The history is returned in a list of 

```
tuple(sourceObject, sourceElementName, list[intermediateSteps...])
```

The `intermediateSteps` are any intermediate modeling steps performed by an
object. As the above naming algorithm shows, each step will append the shape
tag of the first source element. If the object invokes more than one
`TopoShaper` new style maker, then the same tag will be appended more than
once. The intermediate modeling steps are recovered by recursively decoding the
element name until the tag changes.

Note that the `getRelatedElements()` described in previous section works fine
when guessing missing element names caused by changes in the previous model
step. It works poorly when the change happens several steps into the history.
A more general method is exposed as `Part.getRelatedElement()` to combat this
problem. It calls the above `TopoShape::getRelatedElements()` first. And if
nothing is returned, it will try to trace each and every elements history, and
return those having the same initial source element name as the missing one.

# Element Coloring

One very practical application of history tracing is element color mapping. We
know that FreeCAD already has element mapping in boolean `Part` features.
However, without a stable element reference, the color mapping is less
meaningful, because you either rely on the algorithm to combine colors from
child objects for you, or choose to override the color of the entire object.
You can set individual face color, but it will get overwritten every time you
do a recompute.

With the new stable element reference, plus the capability of element history
trace, `ViewProviderPartExt`, which is the parent view provider of all derived
`Part::Feature` now has a generalized implementation of element color mapping,
which are controlled by four new properties, `MapFaceColor`, `MapLineColor`,
`MapPointColor`, and `MapTransparency`. By default, only `MapFaceColor` is on,
which will map child object's face color into parent, just like current boolean
feature, but better. To change default setting, create a boolean parameter with
the same name under `BaseApp/Preferences/Mod/Part`.

To demonstrate its capability, let's pick a pure Python feature, the
[Part::Connect](https://www.freecadweb.org/wiki/Part_JoinConnect), which has
some complex modeling logic. The color mapping code, as well as the element
naming algorithm, are general enough that they work for `Connect` without
modifying a single line of its Python code.

[[images/element-color.gif]]

Notice how the `Fillet` object auto colors the new fillet face with one of its
connected faces. That fillet face is generated from an edge.
`ViewProviderPartExt` will trace back to the source object, i.e. the `Connect`
object, and use the first `Face` in the `Connect` that contains this edge for
coloring.

In order to maintain backward compatibility, no changes in the view object
will touch the document object for recompute. You'll need to manually mark the
object for recompute to see the color in effect.

The screen cast above also shows how easy it is to override individual element
color and transparency. The element color is stable across recomputation thanks
to the stable element reference. Unlike how `Fillet` object treats the edge
reference, `ViewProviderPartExt` will auto deduce related element names in case
any reference is gone. To color any element using Python,

```python
vobj = App.ActiveDocument.Connect.ViewObject
# obtain the existing element color as a dict
colors = vobj.ElementColors
# add some new color
colors['Face3'] = (1.0,0.0,0.0,0.5)
vobj.ElementColors = colors
```

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
| Total recomupation time | 40 s | 52 s |
| Uncompressed File Size | 17.8 MiB | 22.6 MiB |
| Compressed (level 3) size | 2.8 MiB | 3.6 MiB |

