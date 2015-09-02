![#PyangBind](http://rob.sh/img/pyangbind.png)

**PyangBind** is a plugin for [Pyang](https://github.com/mbj4668/pyang) which converts YANG data models into a Python class hierarchy, such that Python can be used to manipulate data which conforms with a YANG data model.

**PyangBind** does not exhaustively support all YANG data types - and is rather focused to support those that are used within commonly used standard, and vendor-specific models. Contributions to improve YANG (RFC6020) coverage are always welcome!

Development of **PyangBind** has been motivated by consumption of the [**OpenConfig**](http://www.openconfig.net) YANG models - and in general, are tested against them. The classes generated by **PyangBind** are intended to provide network operators with a starting point for using OpenConfig models within their network management. It is intended that factory methods to convert Python data trees into output suitable for transport to network devices will be created (e.g., a PyangBind to NETCONF-XML, or JSON objects suitable for use with other transport protocols).

## Usage

**PyangBind** can be called by using the Pyang ```--plugindir``` argument:

```
pyang --plugindir /path/to/repo -f pybind <yang-files>
```

All output is written to ```stdout``` or the file handle specified with ```-o```. Optionally, the ```--xpath-helper``` command-line argument can be used to invoke a module to allow XPath expressions for leafrefs to be resolved ([see the relevant documentation](#leafref-helper)).

Once bindings have been generated, each YANG module is included as a top-level class within the output bindings file. Using the following module as an example:

```javascript
module pyangbind-example {
    yang-version "1";
    namespace "http://rob.sh/pyangbind/example";
    prefix "example";
    organization "pyangbind";
    contact "pyangbind@rob.sh";

    description
        "A test module that checks that the plugin works OK";
    revision 2015-04-30 {
        description "Initial Release";
        reference "release.00";
    }

    container parent {
        description
            "A container";

        leaf integer {
            type int8;
            description
              "A test leaf for uint8";
        }
    }
}
```

The module is compiled using ```pyang --plugindir <pyangbind-dir> -f pybind pyangbind-example.yang -o bindings.py``` from the command-line.

A consuming application imports the Python module that is output, and can instantiate a new copy of the base YANG module class:

```python
#!/usr/bin/env python

from bindings import pyangbind_example

bindings = pyangbind_example()
bindings.parent.integer = 12
```

Note, that as of release.01, pyangbind expects a ```lib``` directory locally to the ```bindings.py``` file referenced above. This contains the ```xpathhelper``` and ```yangtypes``` modules - such that this code does not need to be duplicated across each ```bindings.py``` file. In the future, it is intended that these functions be provided as an installable module, but this is is still *TODO*.

Each data leaf can be referred to via the path described in the YANG module. Each leaf is represented by the base class that is described in the [type support](#type-support) section of this document.

Where nodes of the data model's tree are not Python-safe names - for example, ```global``` or ```as```,  an underscore ("\_") is appended to the data node's name. Additionally, where characters that are considered operators are included in the node's name, these are translated to underscores (e.g., "-" becomes "\_").

### YANG Dependencies
With some outputs (e.g., ```tree``` or ```jstree```), Pyang does not require the ability to resolve all typedefs (e.g., it can just display that the type is inet:ip-address). PyangBind **requires** the ability to resolve all typedefs to their underlying definition, such that a new datatype can be defined. In some cases, this will lead to needing to include 'types' YANG files on the command line, such that pyang can load them - and they are made available to PyangBind to build types.

For example; openconfig-bgp will require the path to ietf-yang-types to be specified via:

```
/usr/local/bin/pyang --plugindir ~/Code/pyangbind -f pybind \
-o bindings.py -p ../policy/ bgp.yang ~/path/to/ietf-yang-types.yang
```
The error messages when a type is not correctly resolved are not very pretty currently and will be made more so in the future.

### Generic Wrapper Class Methods
Each native type is wrapped in a YANGDynClass dynamic type - which provides helper methods for YANG leaves:

* ```default()``` - returns the default value of the YANG leaf.
* ```changed()``` - returns ```True``` if the value has been changed from the default, ```False``` otherwise.
* ```yang_name()``` - returns the YANG data element's name (since not all data elements name are safe to be used in Python).

### YANG Container Methods

Each YANG container is represented as an individual ```class``` within the hierarchy, wrapped with the generic wrappers described above.

In addition, a YANG container provides a set of methods to determine properties of the container:

 * ```elements()``` - which provides a list of the elements of the YANG-described tree which branch from this container.
 * ```get(filter=False)``` - providing a means to return a Python dictionary hiearchy showing the tree branching from the container node. Where the ```filter``` argument is set to ```True``` only elements that have changed from their default values due to manipulation of the model are returned - when filter is not specified the default values of all leaves are returned.

### YANG List Methods

The special ```add()``` and ```delete()``` methods can be used to create and remove entries from the list. Utilising the ```add()``` method will also set the associated key values.

Since YANG lists are essentially keyed - as per Python dictionaries - a ```keys()``` method is provided to retrieve list members. In addition, iterations over a YANGList behave as would be expected from a dictionary in Python (rather than a list).

### YANG String 'pattern' Restrictions

Where a ```pattern``` restriction is specified in the definition of a string leaf, the Python ```re``` library is used to compile this regular expression. As per the definition in RFC6020, ```pattern``` regexps are assumed to be compatible with the definition in [XSD-TYPES](http://www.w3.org/TR/2004/REC-xmlschema-2-20041028/). To this end, all regexps provided have "^" prepended and "$" appended to them if not already specified. This behaviour is different to the default behaviour of ```re.match()``` which will usually allow a user to match a partial string.

### Errors thrown by PyangBind

A number of the errors that PyangBind raises are based on the YANG model input - and are thrown from ```YANGDynClass```. In general, these will be raised when initiating an instance of the generated class (rather than during the initial generation of the bindings) -- in general, PyangBind expects Pyang's validation to have thrown errors *before* the model is parsed (it remains to be seen whether this is a safe assumption!).

Otherwise, PyangBind's generated classes try to be consistent with the error type that a Python programmer would expect from the native types Python provides. That is to say:

* ```KeyError``` is raised when a ```__getitem__``` call results in a member not being found in the referenced item, this includes where members of a YANG ```list``` are not found.
* ```ValueError``` is raised when a value does not conform with the YANG datatype. Since the intention is that a Python programmer does not need to care what types are used 'behind the scenes' by PyangBind ```ValueError``` will only be raised where the input cannot be cast to the  native type (for example, a YANG ```uint32``` will accept ```True``` as an input value, but really store the value of '1' in the leaf).
* ```AttributeError``` will be thrown where a method that does not exist in the encapsulated type is used (as per standard Python errors); a leaf that does not exist in the YANG model is referenced; or an attempt to set a non-configurable value is made. Note that in some cases, ```dir(...)``` on the object may show that the method is available in the YANGDynClass object, since the requirement to capture calls where the data value is changed results in the need to generically overload these methods.


### Setting ```config: false``` Leaves

```AttributeError``` is also thrown when an attempt to set a ```config: false``` leaf value is made. Backend systems that are populating a model (when reading state from a device for example) should call the ```_set_FOO()``` method directly (which is generated independently of the ```config``` state of the leaf) to set the value.

For example:

```python
>>> b.bgp.global_.state.as_ = 12
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: can't set attribute
>>> b.bgp.global_.state._set_as_(12)
>>> b.bgp.global_.state.as_
12
```

### <a anchor="leafref-helper"></a>leafref Nodes and xpathhelper.YANGPathHelper

The ```YANGPathHelper``` class in the xpathhelper module provides a lightweight means to be able to establish a tree structure against which pyangbind modules register themselves. In order to enable this behaviour use the ```---with-xpathhelper``` flag during code generation.

When ```--with-xpathhelper``` modules will automatically register themselves against the tree structure that is provided using the ```path_helper``` keyword-argument. ```path_helper``` is expected to be set to an instance of ```lib.xpathhelper.YANGPathHelper``` - which is created independently to the root YANG module, as shown below:

```python
from lib.xpathhelper import YANGPathHelper
from oc_bgp import bgp
from oc_routing_policy import routing_policy as rpol

# Create an instance of YANGPathHelper() acting as the "/"
# node in the configuration hierarchy
config_tree_helper = YANGPathHelper()

# Create an instance of the OpenConfig BGP module
# which will register itself at "/bgp" within the path helper.
bgp_cfg = bgp(path_helper=config_tree_helper)

# Create an instance of the OpenConfig routing policy module
# which will register itself at "/routing-policy" within the path
# helper.
rpol_cfg = rpol(path_helper=config_tree_helper)
```

As well as providing PyangBind a means to internally validate the existance of leaves (and their values), the YANGPathHelper can be used to extract particular elements of the model hierarchy directly via XPath expressions, rather than using the ```.get()``` method to return a dictionary hiearchy. YANGPathHelper provides:

 * ```get(object_path, caller=False)``` - where object_path is an XPath expression and caller indicates the calling object's path (such that relative paths can be resolved). For instance, ```object_path="../../uncle"``` and ```caller="/grandfather/father/son"``` will resolve to ```/grandfather/uncle```. Absolute paths can also be used.
 * ```tostring(pretty_print=False)``` - prints a representation of the tree that is in use by the YANGPathHelper. It is not expected that this is of particular use to a consuming application as it simply stores a set of references to class instances (via an intermediate UUID identifier).
 * Internal ```register()``` and ```deregister()``` methods exist, but are unlikely to be of significant use to consuming applications.

```YANGPathHelper``` raises ```xpathhelper.XPathError``` when invalid paths are specified.

The intention of ensuring that the YANGPathHelper is external to any generated PyangBind classes is to allow an application to build an arbitrary model hierarchy via different modules, thus ensuring that it is not required to build a single PyangBind class hierarchy for all models (which can become cumbersome due to large files!).

### leaf-ref Validation when path_helper is set.

Leaves that are specified as ```type leafref``` will utilise the ```path_helper``` module in order to validate the content that they are set to. Three distinct validation cases exist:

##### Cases where the leafref target is a single leaf

When a leafref's target is a single leaf PyangBind essentially creates a pointer, such that the following module:

```
container test-container {
	leaf target {
		type string;
	}

	leaf pointer {
		type leafref {
			path "../target";
		}
	}
}
```
Results in the ```pointer``` leaf returning the same value as the value of ```target```. Setting the ```pointer``` leaf will also set the value of the ```target``` leaf.

##### Cases where the leafref target is a list or leaf-list and require-instance is true

When a leafref specifies that ```require-instance``` is true, PyangBind will ensure that the value that the ```leafref``` leaf is set to is a member of the ```leaf-list``` or ```list``` that is targeted by the leafref.

##### Cases where the leafref target is a list or a leaf-list and require-instance is false

When ```require-instance``` is set to false, PyangBind will simply treat the leafref like a string. In the future, this behaviour may change to ensure that the value that is set would be a valid element in the ```leaf-list``` or ```list``` but this is still *TODO*.


## <a anchor="type-support"></a>YANG Type Support

**Type**            | **Sub-Statement**   | **Supported Type**      | **Unit Tests**
--------------------|--------------------|--------------------------|---------------
**binary**          | -                   | [bitarray](https://github.com/ilanschnell/bitarray)           | tests/binary
-                   | length              | Supported           | tests/binary
**bits**            | -                   | Not supported           | N/A
-                   | position            | Not supported           | N/A
**boolean**         | -                   | YANGBool                | tests/boolean-empty
**empty**           | -                   | YANGBool                | tests/boolean-empty
**decimal64**       | -                   | [Decimal](https://docs.python.org/2/library/decimal.html) | tests/decimal64
-                   | fraction-digits     | Supported               | tests/decimal64
**enumeration**     | -                   | Supported               | tests/enumeration
**identityref**     | -                   | Supported               | tests/identityref
**int{8,16,32,64}** | -                   | [numpy int](http://docs.scipy.org/doc/numpy/user/basics.types.html) | tests/int
-                   | range               | Supported               | tests/int
**uint{8,16,32,64}**| -                   | [numpy uint](http://docs.scipy.org/doc/numpy/user/basics.types.html) | tests/int
-                   | range               | Supported               | tests/int
**leafref**         | -                   | Supported               | tests/xpath/...
**string**          | -                   | *str*                   | tests/string
-                   | pattern             | Using python *re.match* | tests/string
-                   | length              | Not supported           | N/A
**typedef**         | -                   | Supported               | tests/typedef
**container**       | -                   | *class*                 | tests/*
**list**            | -                   | YANGList                | tests/list
**leaf-list**       | -                   | TypedList               | tests/leaf-list
**union**           | -                   | Supported               | tests/union
**choice**          | -                   | Supported               | tests/choice

## Licence

```
Copyright 2015, Rob Shakir (rjs@rob.sh)

This project has been supported by:
          * BT plc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

## Bugs and Comments

Please send bug-reports or comments to rjs@rob.sh - or open an [issue on GitHub](https://github.com/robshakir/pyangbind/issues).

## ACKs
* This project was initiated as part of BT plc. Network Architecture 'future network management' projects.
* Members of the [OpenConfig](http://www.openconfig.net) working group have assisted greatly in debugging, and designing a number of the approaches used in PyangBind.