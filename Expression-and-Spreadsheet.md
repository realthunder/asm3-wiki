# Overview

Starting from version 0.9, there is a major upgrade of FreeCAD 
[Expression Engine](http://www.freecadweb.org/wiki/index.php?title=Expressions) and
[Spreadsheet Workbench](http://www.freecadweb.org/wiki/index.php?title=Spreadsheet_Module).
This article highlights the difference and enhancement made in my branch
comparing to the upstream. It assumes that the reader is already familiar
with the basic usage of expression and spreadsheet. If not, please check out
the links above first, and also 
[this](https://yorikvanhavre.gitbooks.io/a-freecad-manual/content/working_with_freecad/using_spreadsheets.html)
book chapter.

Here is a brief list of the enhancements,

* Expression syntax has been greatly extended to become a full blown scripting
  language. The syntax is borrowed from Python with a few extension to support 
  FreeCAD unit system, document object reference, etc. It is _mostly_ backward
  compatible with upstream syntax with very few [exceptions](#python-syntax-mode).

* Because of the extended syntax, expression can now evaluates into any type
  of Python object. And the expression engine has been extended to support
  binding to any type of property. See [here](#expression-binding) for more
  details.

* Both the `PropertyExpressionEngine` and `PropertySheet` has been modified to
  behave similarly as a link property that supports external objects. This
  means,
  * The external referenced document will be automatically opened together with
    the owner document;
  * Supports the new [[Topological Naming]] (with the [new syntax](#user-content-subobject))

* New alternative [edit mode](#spreadsheet-edit-mode) support in `Spreadsheet`,
  including button, combo box, and label, which makes it possible for user to
  create simple customized GUI using spreadsheet.

* Support relative and absolute cell and range reference when copying and
  pasting spreadsheet cell(s). See [here](#cell-and-range-reference) for more
  details.

For those who do not care much of the details, and would like to jump straight
into the business, you can skip to [here](#demonstration) for a demo.

# Security Concern

The syntax of the expression language is borrowed from Python, minus the class
definition stuff. One thing worth mention before we get into the details is
that although the scripts look like Python, it is not interpreted by the Python
interpreter. For security reason, the FreeCAD expression classes and parser are
refactored to interpret the script by itself. This way, we have more control to
what the script can or cannot do. By default, the following Python built-in
functions are blocked,

* `eval()`
* `execfile()`
* `exec()`
* `__import__()`
* `file()`
* `open()`
* `input()`

No Python module is allowed except the followings,

* `builtins` (or `__builtin__` for Python 2)
* `math`
* `re`
* `collections`
* `FreeCAD` (i.e. the `App` module)
* `FreeCADGui` (i.e. the `Gui` module. However, `Gui.docommand()` is blocked)
* `Part`

User can manually white list modules by adding a boolean parameter under
`BaseApp/Preferences/Expression/PyModules` with the absolute module reference
as the parameter name, as shown below. And because of its special purpose, this
parameter group is protected against writing using expressions.

[[images/module-whitelist.png]]

Even with all the above precautions, the new power of `Expression` may still
expose potential danger to the end user, because it can now call into other
Python code, which may have unknown security risk. There will be more security
measure implemented in the future, possible candidates are,

* Explicitly showing every expression usage to the user before loading a file
* Implement more granulated access control, such as,
  * No extended scripting allowed (but still allow upstream like expression usage),
  * No import allowed,
  * No callable allowed,
  * No property writing.
* Support digital signature of files, which not only can deny execution of
  untrusted files, but also make it more convenient to grant more access right
  to trusted ones, like those with user's own signature.

# Python Syntax Mode

Great effort has been made to make the extended expression syntax mimicking
that of Python and yet backward compatible with upstream FreeCAD. However,
there are exceptions. At the time of this writing, the only one I am aware of
is that `in` as the unit inch is no longer recognized, as it badly conflicts
with Python keyword `in`. You can still use `"` for the unit inch.

The extended syntax is fully defined in 
[this](/realthunder/FreeCAD/tree/LinkStage3/src/App/ExpressionParser.y) file.

By default, all upstream keywords (except `in`) are recognized, which
includes those for units, built-in functions and constants. It adds up to about
a hundred keywords, and therefore greatly interfere with Python-like script
code. And not to mention the awkward way to quote string using `<<` and `>>`
because `'` and `"` are taken up by unit foot and inch. To alleviate this
problem, a pseudo statement is introduced to switch syntax to Python compatible
mode

```
#@pybegin
```

Although it looks like a Python comment, it is recognized by the expression
parser as a statement, just like `pass` or `return`, so you must honour the
usual Python indentation rules. In addition, this statement only takes effect
in runtime, not parse-time. This means that if you add this statement outside
of a function definition, the code inside of the function may not be run in
Python mode depending on its calling context. 

Once inside Python mode, 

* Units keywords are no longer recognized;

* Expression built-in functions are still recognized, but can be overload by 
  variables and your own function definition. 

* Variable name lookup will include Python builtin module, and take precedence
  over expression built-in functions.

* String literal now supports the usual `'` and `"`. Note that `<<` and `>>`
  as well as the long string `'''` and `"""` are always supported regardless of
  the current mode.

* `,` can no longer be used as decimal separator. In default mode, expression
  `1,2` is interpreted as decimal `1.2`, while `1, 2` means two integers. In
  Python mode, they are both interpreted as two integers.

* Python comments work as expected. However, in default mode, comment is only
  recognized if `#` is followed by at least one space. This is to allow external
  object referencing syntax, `document#object`. One thing to note that, after
  you entered the script into a spreadsheet cell, it is immediately parsed
  with all comments stripped away. So there won't be any comments left if you
  edit the script again. You can always use the string expression as comment,
  which will be kept untouched after parsing, especially the `'''` or `"""`
  long string. However, since string expression is considered a statement, it
  must obey the usual indentation rules. If you really want to use and keep the
  normal comments, you can enter the script as plain string (by starting the
  script with a single `'`, instead of `=`). You can then run the script using
  the built-in function `eval()` (which is not the Python `eval`, but FreeCAD's
  own implementation), or `func()`. See [here](#eval) for more details.

To end the Python mode,

```
#@pyend
```

Note that the mode switching statement is not counted, so it can be ended by
a single `#@pyend` regardless how many `#@pybegin` ahead.

# Syntax Difference vs. Python

As mentioned above, the extended expression syntax is borrowed from Python,
Python 3 to be exact. This section highlights the difference between the two.


## Not Implemented

Here are the things Python has, but not implemented (yet) by FreeCAD expression
parser,

* No support of class definition, decorator or annotation.

* No implementation of `async`, `yield`, `with` or `assert` statement.

* No bitwise or matrix operator. The reason for not supporting bitwise operator
  is because the shift operator `<<` and `>>` conflicts with expression string
  quoting.

* Limited string literal prefix. Only support `r` and `u`, in other words
  no support of binary or formatted string literal.

* Limited string escaping support. Only support `\t, \r, \n \\, \', \"`. And
  because expression supports quoting string with `<<` and `>>`, `\>` is
  also supported. There is no support for octal, hex or Unicode escaping.

* Variable declaration and assignment can only be used inside a function body
  or by a script invoked through `eval()` or `func()`. This restriction also
  applies to implicit variable assignment with `for` statement, but not
  applicable to list/set/dict comprehensions, which can be used anywhere.

* The _global_ variable scope is defined as the variables defined in top level
  function or the first calling of `eval()`. And _local_ means the current
  executing function or `eval()`. The variable scope, or more specifically, the
  evaluation stacks are isolated from the Python interpretor running inside
  FreeCAD. The `global`, `local`, `nonlocal` and `del` statements are supported
  but with altered definition of scope described above.

* `from ... import ...` statement only supports absolute import.

Anything not mentioned above should be supported by the extended expression
syntax, and works about the same as in Python.

## Keywords

In default mode, FreeCAD expression supports over 60 unit keywords. And you
can create more unit by doing arithmetics with them, such as `m/s`. The full
list of supported units can be found [here](https://github.com/realthunder/FreeCAD/blob/837faf3dde92dcf541fa3859b608e3a0da197eed/src/App/ExpressionParser.l#L361).
These are mostly the same as upstream FreeCAD.

## Function Definition

Named function is defined just like in Python, with `def name(args...)`.
However, unlike Python, this statement returns a Python callable object,
similar to `lambda` expression. This is to make it easy for the user to turn
a spreadsheet cell into a callable, which can be accessed by others just like
a newly defined instance method. Moreover, if a cell contains nothing but a
single function definition, the function name will be automatically used as
the alias of the cell.

Another subtle difference from Python is that if a function reaching the end of
control without a `return` statement, it will return the result of the last
statement instead of `None`. However, it is always better to use an explicit
`return` statement to avoid any surprises. For example, if the last statement
is a string expression, it will return `None` instead of the string. This is an
optimization for using string expression as comment.

## Document Object Property Reference

FreeCAD expression has dedicated syntax for referencing a property of a document
object. The word `identifier` used below is defined as usual in most
programming language, that is, a text string starting with an alphabet or `_`,
followed by any alpha numericals or `_`. There are a few new additional
referencing scheme supported comparing to upstream, all of which are described
as follows,

* `identifier`. A single identifier can be used to reference a property of the
  owner document object of the expression, which we shall call it reference to
  a _local property_. However, when inside a function body, or in a script
  invoked by `eval()`, name lookup will first be conducted from the inner local
  variable scope, all way up till global scope. And when in Python mode, the
  builtin modules will also be searched if no matched variable is found. If
  still no match is found, then the owner object's property list will be
  searched at last.

* <a name="pseudo-property"></a>`.identifier`. A new syntax is introduced to
  make it easy for user to directly referencing a _local property_ without
  ambiguity, by preceding the `identifier` with a `.`, similar to Python's
  relative import syntax.

  As a convenience, there is a list of pseudo properties defined as below,
  which can be use on any object referencing scheme, not limited to local
  property reference.

  * `_shape`, using `Part.getShape()` to obtain the referenced object's
    `Shape`. `Part.getShape()` supports getting shape from many objects
    without having a property of `Shape`, including `App::Link`,
    `App::LinkGroup` and `App::Part`.

  * `_pla`, using `DocumentObject.getSubObject()` to obtained the accumulated
    placement of a [sub-object](#user-content-subobject) reference.

  * `_matrix`, same as above, but return as a `App.Matrix`

  * `__pla`, similar to `_pla`, except that if the sub-object is an
    `App::Link`, then it will use `DocumentObject.getLinkedObject()` to
    accumulate the link's placement as well.

  * `__matrix`, same as `__pla`, but return as a `App.Matrix`

  * `_self`, refers to the object itself. Because none of the referencing scheme
    can refer to an object without property, the user can use this pseudo
    property to access non-property attributes of any object.

  * `_app`, refers to the `App` module.

  * `_gui`, refers to the `Gui` module.

  * `_part`, refers to the `Part` module.

  * `_re`, refers to Python `regex` module.

  * `_py`, refers to Python `builtins` module.

  * `_math`, refers to Python `math` module.

  * `_coll`, refers to Python `collections` module.

* `<<label>>.identifier`. Referencing an object using its user changeable
  label. Note that the label must be a string with FreeCAD expression quoting
  style. In case the referenced object's label is changed, this expression
  will be automatically updated. This syntax has no ambiguity, unlike the one
  below.

* `idenfifier1.identifier2`. This is a troublesome syntax inherited from upstream
  FreeCAD, because there are many ways to interpret it, as shown below
  * `local_property.sub_property`
  * `object_name.property_name`
  * `object_label.property_name`
  * And because of the extended syntax, it could also be a reference to a
    local variable.

  In order to reduce ambiguities, the new expression parser will try to
  resolve the identifier at parsing time, and possibly change the input
  expression to an alternative and less ambiguous form, with the following
  rules applied in listed order,

  * If there is an object with internal name of `identifier1` and having
    a property named `identifier2`, then there will be no runtime variable
    lookup.

  * If there is an object with label of `identifier1` and having a property
    named `identifier2`, then the expression will be changed to 
    `<<identifier1>>.identifier2`
    
  * If there is a local property of name `identifier1` then the expression
    will be changed to `.identifier1.identifier2`

  * If none of the above rules is applicable, then a runtime search among
    all defined variables will be performed.

* <a name="subobject"></a>`identifier1.<<subname>>.identifier2`, a reference
  to a sub-object using a [subname](Link#user-content-subname) reference,
  where `identifier1` is referencing the parent object by its internal name,
  and `identifier2` is the sub-object's property name. You can use label
  reference inside `subname` by preceding the label with `$`, and the labels
  will also be auto updated if changed by user. In addition, you can also
  include geometry reference inside `subname`, and it will be auto updated
  if the referenced geometry model is updated, thanks to the new 
  [[Topological Naming]] feature. 

  For example, an expression of `Assembly.<<Parts.$Fusion.Edge10>>._shape`
  will give you a Python edge shape object transformed into the global
  coordinate space. The `$` inside means that the fusion sub-object is
  referenced by its label. So if user re-label the object to, say, `Fuse`, then
  the expression will be auto updated to
  `Assembly.<<Parts.$Fuse.Edge10>>._shape`. And if the fusion object is
  modified, say, refined, and `Edge10` is shifted to `Edge9`, then the
  expression will be auto updated accordingly, too.

* `<<label>>.<<subname>>.identifier`, same as above, except that the parent
  object is referenced by its label.

* `identifier1#identifier2.identifier3`, this syntax is used to reference
  a property `identifier3` of an object with internal name `identifier2` that
  belongs to an external document labelled `identifier1`. Unlike upstream,
  you have to save both the referencing and referenced documents at least
  once before using this type of reference, otherwise exception will be
  thrown. Whenever a document containing this type of reference is opened,
  the referenced external document will be automatically opened together.

  `identifier1` and `identifier2` can also be a quoted string to reference the
  document and/or object by their label. And there can also be `subname`
  reference. For example, `<<asm>>#<<top assembly>>.<<Parts.Body.Pad>>.Length`.

## Cell and Range Reference

FreeCAD has a [Spreadsheet Workbench](http://www.freecadweb.org/wiki/index.php?title=Spreadsheet_Module),
which allows the user to create spreadsheet similar to Microsoft Excel or
Google Sheet. The cells inside a spreadsheet are named with the same
convention, that is one or two alphabets to refer to a column, and an integer
starting from one to refer to a row, e.g. `A1`, `AB1234`. The maximum supported
row index is 16384.

The cell naming conforms to the definition of an `identifier`. And also because
the spreadsheet object will create a property for each non-empty cell, the cell
can be referenced in an expression like any other properties. There is a new
feature implemented to support the concept of _relative vs. absolute cell reference_.
The normal cell references are all treated as _relative_, meaning that when you
copy the cell (or a range of cells), and paste into some other cell location,
all relative cell reference will be offset accordingly. For example, if cell
`B1` references to `A1`, and the user copies `B1` to `B2`, the `A1` reference
will but auto changed to `A2`.

If auto offset is not wanted, the user can use the _absolute cell reference_,
by preceding either the column, or row, or both with `$`. For example, `$A1`
means absolute column `A`, but relative row one, while `$A$1` refers to an
absolute cell. Although the absolute reference does not conform to the usual
`identifier` definition, it is specifically allowed to be used inside an object
property reference just like relative cell reference.

A range of cell can be referred to like `A1:C2`. Some of the built-in functions
accepts range reference directly as input argument, like `sum(A1:C2)`, etc.
However, a range cannot be used inside object property reference. It can,
however, be unpacked into a list or tuple, like `[*A1:C2]`, or `(*A1:C2,)`. The
same unpack syntax can be used to pass a range of cell into normal function as
arguments, such as `test(*A1:C2)`.

## `IDict`

In addition to normal `dict`, expression supports another convenience form
called `idict`, meaning identifier dictionary, with the following syntax

```
{ identifier=obj, ... }
```

The key must be a valid `identifier`, and the value can be anything. The
expression evaluates to a normal Python `dict` with string keys. This
convenience form is provided to save user from having to quote the key using
awkward `<<` and `>>`.


## Special Built-in Functions

In addition to all built-in function in upstream FreeCAD, here are some
additional ones.

### `eval`

Not to be confused with the Python built-in with the same name, this function
evaluates scripts with FreeCAD expression extended syntax. The reason why
Python's `eval()` is dangerous is because Python's system modules gives the
script access to user's local system. And `eval()` allows an attacker to run
any script code. FreeCAD expression engine, by default, does not have access to
user's local system. So allowing `eval()` exposes no more danger than allowing
user to enter expression in a spreadsheet cell.

The usage of `eval()` is,

```
eval(cmd, ...)
```

`cmd` can be a single string or a sequence of strings. You can pass in one or
more optional input arguments as pre-defined variables before script evaluation.
The argument supports positional and keyword argument, as well as sequence and
dictionary unpacking. Positional argument is named as `_index_`, where `index`
is the argument's position starting with one. For example, 

```
# suppose
c = [ 3, 2 ]
d = { 'g':4 }

# calling eval with the following arguments
eval(cmd, 5, 6, a=[1,2,3], *c, **d)

# is equivalant to pre-defining the following variable
_1_ = 5
_2_ = 6
a = [1,2,3]
_3_ = 3
_4_ = 2
g = 4
```

One advantage of `eval()` over function definition is that the script is stored
as string, and all comments inside are kept untouched.

### `func/func_d`

`func()` and `func_d()` has the exact same call signature as `eval()`. The
difference is that `func/func_d()` returns a callable object instead of
evaluating the input scripts. You can think of it compiles the script, but not
running it. The `d` in `func_d` means `delayed`, that is, the argument passed
into `func_d()` is not evaluated at compile time, but delayed until runtime,

### `import_py`

Similar to Python's `__import__()` function, `import_py(name)` returns a module
object of given name. The difference is that `import_py()` only allows to
import modules that are explicitly enabled by user with a boolean parameter
with the same name as the module, defined in `BaseApp/Preferences/Expression/PyModules`.

# Expression Binding

Because of the extended syntax, an expression can now evaluates to any type of
Python objects, which in turn allows binding of any type of property. The
property view has been modified to enable just that.

By default, the property view will only show some of properties of the selected
object, among which only some types of properties allow expression binding, by
clicking the small blue `f(x)` button shown below.

[[images/property-edit1.png]]

Now, you can reveal all properties by right clicking anywhere in the property 
view and select `Show all`. As shown below, you can even use an expression to
generate a shape and bind it to the `Shape` property, by using the `Expression...`
menu action. Be careful though, some properties are hidden for a reason.
Consider this as an advanced feature, and make sure you know what you are
doing.

[[images/property-edit2.png]]

The other menu actions are used for dynamic access control of properties. See
[here](Core-Changes#properties) for more details. One property status that is
particular important to `ExpressionEngine` is the `Output` status. When an
object is recomputed, its `ExpressionEngine` will compute all `non-Output`
property expression binding first, then calls object's `execute()` function,
and finally computes any `Output` property expression binding. In other words,
the `Output` property status controls the order of expression binding
evaluation within an object.


# Spreadsheet Edit Mode

Apart from the aforementioned [relative cell](#cell-and-range-reference) feature,
another important new feature of spreadsheet is the introduction of alternative
_edit mode_, which enables user to build simple customized GUI using spreadsheet.
Simply right click the cell and choose one of the edit mode in the context menu
to activate this feature. `Normal` mode is the default mode, which directly edits
the cell content. Other modes are described as follows.

[[images/edit-mode.png]]

## Button Edit Mode

Any cell that defines a function can be switched into button edit mode, where
a button widget is shown in the cell's position. The button text is looked up
in the following places in the given order,
* Function doc string,
* Function name,
* Cell alias,
* Cell address.

The function is invoked when the button is clicked, after which the whole
current document is recomputed.

## Combo Box Edit Mode

The ComboBox edit mode requires a cell with a list or tuple expression of two
or more items. The first item must be a mapping of string key. There is no
requirement on the value type. The second item must be a string object. When
the ComboBox edit mode is activated, the cell will only show the second string
item as the content. When the cell is edited by double clicking, or keyboard
enter, it shows a ComboBox widget populated with the first item's string keys.
When the user choose an item, the second item will be assigned the selected
string. If there is a third callable item in the cell expression, it will be
invoked on value change with arguments `callable(cell_address, seq)`, where
`cell_address` is the current cell address in text form, and `seq` is the
cell's evaluated list or tuple object.

## Label Edit Mode

The label edit mode requires a cell with a list or tuple expression of at least
one item. The first item must be a string. The rest of the items can be any
thing, and are not touched. When activated, the cell only shows the first string
item as its content, and can be edited as usual. The other items are hidden and
not used. See the following demonstration for an example usage of this mode.

# Demonstration

Here is a demonstration that uses spreadsheet to populate a simple BOM list.
The spreadsheet offers the user to select the document and assembly, and then
populate the list with the parts information from the selection.

[[images/spreadsheet.gif]]

<a name="edit"></a>To reveal the script of a cell with non-default edit mode, you can right click
it and select `Edit mode -> Normal`. And then you can edit the cell as usual.
Alternatively, you can right click the spreadsheet object in the tree view, and
select `Expression actions -> Copy selected`. You can then paste the scripts to
your favorite text editor. This gives you expressions of every cell in the
spreadsheet. Other actions in `Expression actions` can let you copy expressions
of the active document or all opened documents, from not only the spreadsheet,
but also all expression bindings in any objects. You can modify the script in
your editor, and copy back using `Expression actions -> Paste`. You can paste
only part of the scripts, as long as you include the `##@@` marker of the
expression you want to change. Make sure your copied text starts with one of
the `##@@` marker.

Here are the script,

```python
##@@ A1 bom#Spreadsheet.cells (Spreadsheet)
<<Please select document:>>

##@@ A2 bom#Spreadsheet.cells (Spreadsheet)
<<Please select assembly:>>

##@@ A3 bom#Spreadsheet.cells (Spreadsheet)
def Populate():
    #@pybegin
    rstart = 5
    r = rstart
    count = 0
    try:
        while ._self.get('A' + str(r)) :
            count += 1
            r += 1
    except:
        pass

    if count: ._self.removeRows(str(rstart), count)

    r = rstart
    asm = .B2[0][.B2[1]]
    parts = asm.Proxy.getPartGroup()
    for obj in parts.Group :
        for _, col, func in .B3[1] :
            addr = col + str(r)
            ._self.set(addr, "'" + func(obj))

        r += 1

##@@ A4 bom#Spreadsheet.cells (Spreadsheet)
[<<Document>>, ._self.column(), lambda obj : obj.Document.Name]

##@@ B1 bom#Spreadsheet.cells (Spreadsheet)
[._app.listDocuments(), <<None>>]

##@@ B2 bom#Spreadsheet.cells (Spreadsheet)
[{ obj.Label : obj for obj in ._app.getDocument(.B1[1]).Objects if .isAssembly(obj) }, <<None>>]

##@@ B3 bom#Spreadsheet.cells (Spreadsheet)
[<<template>>, [*A4:D4]]

##@@ B4 bom#Spreadsheet.cells (Spreadsheet)
[<<Label>>, ._self.column(), lambda obj : obj.Label]

##@@ C2 bom#Spreadsheet.cells (Spreadsheet)
def isAssembly(obj):
    #@pybegin
    return str(getattr(obj, 'Proxy', None)).startswith('<freecad.asm3.assembly.Assembly ')

##@@ C4 bom#Spreadsheet.cells (Spreadsheet)
[<<Part No.>>, ._self.column(), lambda obj : .partInfo(obj, <<pn>>)]

##@@ D2 bom#Spreadsheet.cells (Spreadsheet)
def partInfo(obj, key):
    #@pybegin
    try:
        return eval(obj.Label2).get(key, <<?>>)
    except:
        return <<?>>

##@@ D4 bom#Spreadsheet.cells (Spreadsheet)
[<<Description>>, ._self.column(), lambda obj : .partInfo(obj, <<desc>>)]

```

<a name="label2"></a>Cell `C2` defines a function to filter for assembly object only. `D2` defines
a function to extract part information defined in the description field of
the object (i.e. property `Label2`). In this example, an `idict` is used to 
define the part information. `D2` just calls `eval()` to evaluate the script
inside `Label2`. You can of course use various other ways, such as simple comma
separated strings, and then use `string.split(',')` to obtain the fields.

As mentioned before, a single function definition in a cell automatically uses
the function name as alias, so the above `C2` is called by their function name
directly inside `B2`.

The BOM list columns (`A4:D4`) are defined using `Label` edit mode, where the
first item of the list is the column heading, the second item stores the
current column address, and the third item is a lambda to extract the
corresponding part information.

`B3` is defined as a convenience to consolidate all column definition. It is 
using `Label` edit mode to reduce cell display verbosity.

Once thing to note is that `B1` containing a list of currently opened documents
for user to select. There is no auto recompute mechanism in case of new or
deleted document. The user must manually recompute the cell by right click it
and choose `Recompute`.

