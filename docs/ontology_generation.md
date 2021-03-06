# Understand and Build Ontology

Forte is built on top of an _Ontology_ system, which defines the relations
between NLP annotations, such as the relation between words and documents,
or between two words. This is the core of Forte.

The ontology can be specified in a JSON format and Forte provides tools 
to help convert the ontology into production code (Python). 
Make sure Forte has been installed before starting to build the ontology.

### A Simple Ontology Config
Imagine you need to develop an information retrieval system for a pet shop for accounting purposes, 
first thing first, you need to understand what the expected outputs from the documents are. Let's 
say you need to sort out key information related to assets such as `Pet` and `Revenue`from text. 
We built an example ontology here: [pet shop ontology](https://github.com/asyml/forte/blob/master/examples/ontology/pet_shop.json)

Now, before we drill down into the details, under Forte's root directory, try run the following command:
```shell
generate_ontology create -i examples/ontology/pet_shop.json -o examples/ontology -r
```
If command succeeds, you will find some python code generated in `examples/ontology`,
under the package `awesome.pet.com`. This is what the Forte ontology system does - generate
the python classes needed to handle the NLP data structures.

The JSON ontology spec is self-explanatory, where we defined types like `Pet` and `Owner`. 
Each type has some attributes, for example `Pet` has attributes like `pet_type` and `color`.
In addition, the `Owner` has a list of `Pet` given that one owner might have multiple pets. 
The logic structure was defined in the python script. 

In the rest of this tutorial, we will walk through this example to let you learn how to:
  * Define a simple ontology spec for your project.
  * Import other ontology(s) to build yours.
  * Generate the corresponding python classes automatically and use them in 
    your project.

### Before We Start
You need to understand a few basic concepts of Forte's ontology system before you start. 

* *Entry* - An entry is referred to one NLP unit in the document, such as an annotated sequence or 
relationship between annotated sequences. `Token`, `Sentence` and `DependencyLink` are some examples of entries. 
One entry defined in the config is used to generate one python class.

* *Ontology* - A collection of `entries`, where each entry represents a class. 
The entries of an ontology can span one or more Python modules. In this way, the entry classes
can be imported regardless of the ontology config they were originally generated from.
The modules that contain these entry classes belong to the package `ft.onto`.

* *Attribute* - An attribute is referred to a label or property that is
associated with an entry, for example, `pos_tag` for the entry `Token`.

* *Top* - Top entries are a set of entries that are pre-defined in the module 
`forte.data.ontology.base.top` in the Forte library. All user-defined entries can extend one of 
the top entries. 
 
We provide a set of commonly used NLP entry types in the module 
[``forte.data.ontology.base_ontology.py``](https://github.com/asyml/forte/blob/master/forte/ontology_specs/base_ontology.json). 
Those entries could be used directly in your project!
 
### A Simple Ontology Config Example
Let us consider a simple ontology used for analyze a pet shop's accounting documents.
```json
{
    "name": "pet_shop_ontology",
    "description": "An ontology used to manage the pet shop assets and pets",
    "definitions": [
        {
            "entry_name": "ft.onto.pet_shop.Pet",
            "parent_entry": "forte.data.ontology.top.Annotation",
            "description": "Pets in the shop.",
            "attributes": [
                {
                    "name": "pet_type",
                    "type": "str"
                }
            ]
        },
        {
            "entry_name": "ft.onto.pet_shop.Owner",
            "parent_entry": "forte.data.ontology.top.Annotation",
            "description": "Owner of pets.",
            "attributes": [
                {
                    "name": "name",
                    "type": "str"
                },
                {
                    "name": "pets",
                    "description": "List of pets the owner has.",
                    "type": "List",
                    "item_type": "ft.onto.pet_shop.Pet"
                }
            ]
        }
    ]
}
```

#### Ontology Breakdown

- The top level `name` and `description` are annotation keywords serving for descriptive
purposes only.

- The `definitions` is used to enlist entry definitions, where each entry represents a json object. 
Each entry refers to one concept in the ontology, as well as a Python class. 

##### ```definitions```
Each definition is a dictionary of several keywords:
* The `entry_name` defines the full name of an entry in the form
```<package_name><module_name>.<entry_name>```.
    * The `<package name>` is `ft.onto`, used to create the 
    package directory tree in which the generated module resides.
    * The `<module_name>` is the name of the generated file in which the entry 
    would be placed.
    > Note: Entries in the same config can have different module names.
    * The `<entry_name>` is the generated class name.
    
 * The `parent_entry` defines the base class of the generated entry class. All 
 the user-defined entries can inherit either from any of the top entries or user-defined entries.
 
 * The `description` (optional) provides descriptions of the generated Python class and a user can choose to skip. 
  
 * The `attributes` defines a list of attributes that would be used as instance variables of 
 the generated class. 

##### ```attributes```
Each entry definition can have a couple (can be empty) attributes, mimicking the class variables:

* The `name` defines the name of the property unique to the entry.
* The `description` (optional) provides description of the attribute.
* The `type` defines the attribute type. Currently supported types include:
    * Primitive types - `int`, `float`, `str`, `bool`
    * Composite types - `List`, `Dict`
    * Pre-defined type in the `top` module - The attribute type can inherit from the base
    entry type (defined in the `forte.data.ontology.top` module) or use the class name.
    * User-defined types - The attribute type can inherit from the user-defined entry type
    (defined in (a) the same config;(b) any imported configs). To avoid ambiguity, only the 
    ful name of user-defind types are supported. 
* `item_type: str`: defines the type of the items contained in the set defined by `type`. For example,
   If the `type` of the property is a `List`, then `item_type` defines the type of the items contained in the list. 
* `key_type` and `value_type`: defines the types of value and key in the set defined by `type`. For example, 
   If the `type` of the property is a `Dict`, then `key_type` and `value_type` represent the types of the key and value of the dictionary,
   currently, only primitive types are supported as the `key_type`.

### Major Ontology Types - Annotations, Links, Groups and Generics
The most commonly used **entry types** include: 

* **Annotation**: an annotation contains two integer attributes, `begin` and `end`
  to denote the offsets of a piece of text. For example, a `sentence` can be an annotation. 
  In our example, we use `awesome.pet.com.Color` to annotate the words used to describe colors in the text. 
  All annotations need to inherit from `forte.data.ontology.top.Annotation`. 

* **Link**: a link connects two entries. For example, a dependency link can connect two words. 
  All links need to inherit from `forte.data.ontology.top.BaseLink`. ``parent_type`` and ``child_type`` 
  need to be defined to specify the linked entries in the ontology. 
    
* **Group**: a group can assort multiple entries. For example, a coreference cluster can contain several entity 
  mentions. All links need to inherit from `forte.data.ontology.top.BaseGroup`. The ``member_type`` need to be 
  defined to specify the type of entries in the group. 
  
* **Generics**: there are some entries that do not have the above characteristics, such as
  general meta data storage information. These are `Generics` types. 

To see more examples of these different entry types, you can refer to the example 
[pet shop ontology](https://github.com/asyml/forte/blob/master/examples/ontology/pet_shop.json), or the [base ontology](https://github.com/asyml/forte/blob/master/forte/ontology_specs/base_ontology.json),
an ontology provided by Forte to represent general NLP concepts.

### Import Another Ontology
`imports` is an optional keyword used to import an existing ontology to help build
a new one. This is similar to `import` used in common programming languages:

* The entries of the imported configs can be used in the current config as
types or parent classes.
* The imports could either be 
  - absolute paths
  - relative to the directory of the current config or the current working directory
  - relative to one of the user-provided ``spec_paths`` (see [generation steps](#ontology-generation-steps).)
    
For example, ``ft.onto.ft_module.Word`` has the parent entry defined in the
generated module ``ft.onto.example_import_ontology``. The generation 
framework ensures that the imported JSON configs are generated prior to the
current config. In case of cycle dependency issue between the JSON configs, an 
error would be thrown.

### Package Naming Convention
Each entry should be named following a package convention, such as `ft.onto.Pet`
in this example. This allows the generator to create a package structure for the python
class.

In order to avoid polluting your package space accidentally, the package names
that can be used on the types are restricted. The default package name allowed
is `ft.onto`. However, in many cases you may want to use custom package name, such as`awesome.pet.com`, 
how can we achieve it?
  
We can explicitly define more attributes in the `additional_prefixes`, as shown in the 
following snippet:

```json
{
  "name": "pet_shop_ontology",
  "additional_prefixes": [
    "awesome.pet.com"
  ],
  "description": "An Ontology Used to manage the pet shop assets and pets",
  "definitions": [
    {
      "entry_name": "awesome.pet.com.Color",
      "parent_entry": "forte.data.ontology.top.Annotation",
      "description": "Annotation for color words.",
      "attributes": [
        {
          "name": "color_name",
          "type": "str"
        }
      ]
    },
    {
      "entry_name": "awesome.pet.com.Pet",
      "parent_entry": "forte.data.ontology.top.Annotation",
      "description": "Pets in the shop.",
      "attributes": [
        {
          "name": "pet_type",
          "type": "str"
        },
        {
          "name": "color",
          "type": "awesome.pet.com.Color"
        }
      ]
    }
  ]
}
```
    
### Generate Python Classes from Ontology.
* Write the json spec(s) as per the instructions in the previous sections. 
* Use the command `generate_ontology --create` (add during the installation of Forte) to 
generate the ontology, and `generate_ontology --clean` to clean up the generated ontology. 
The steps are detailed in the following sections.

##### Ontology Generation Steps
Let's drill down into details about how to generate ontology. 

* To verify that the `generate_ontology`
command is found, run `generate_ontology -h`, and the output should look like the
following -
    ```
    $ generate_ontology -h
    usage: generate_ontology [-h] {create, clean} ...
    
    Utility to automatically generate or create ontologies.
    
    *create*: Generate ontology given a root JSON config.
    Example: python generate_ontology.py create --config forte/data/ontology/configs/example_ontology_config.json --no_dry_run
    
    *clean*: Clean a folder of generated ontologies.
    Example: python generate_ontology.py clean --dir generated-files
    
    positional arguments:
      {create, clean}
    
    optional arguments:
      -h, --help      show this help message and exit
    ```

* All the arguments of `generate_ontology create` are explained as below: 
     ```
    usage: generate_ontology create [-h] -i SPEC [-r] [-o DEST_PATH]
                                    [-s [SPEC_PATHS [SPEC_PATHS ...]]]
                                    [-m MERGED_PATH] [-e] [-a]
    
    optional arguments:
      -h, --help            show this help message and exit
      -i SPEC, --spec SPEC  The main input JSON specification.
      -r, --no_dry_run      Generates the package tree in a temporary directory if
                            true, ignores the argument `--dest_path`
      -o DEST_PATH, --dest_path DEST_PATH
                            Destination directory provided by the user. Only used
                            when --no_dry_run is specified. The default directory
                            is the current working directory.
      -s [SPEC_PATHS [SPEC_PATHS ...]], --spec_paths [SPEC_PATHS [SPEC_PATHS ...]]
                            Paths in which the root and imported spec files are to
                            be searched.
      -m MERGED_PATH, --merged_path MERGED_PATH
                            The destination file path for the mergedfile path.
      -e, --exclude_init    Excludes generation of `__init__.py` files in the
                            already existing directories, if`__init__.py` not
                            already present.
      -a, --gen_all         If True, will generate all the ontology,including the
                            existing ones shipped with Forte.

     ```

* Run the `generate_ontology` command in `create` mode. Let the destination path be a directory `src` in user project.

    ```bash
    $ generate_ontology create -i configs/example_complete_ontology_config.json -o src
       
    INFO:scripts.generate_ontology:Ontology will be generated in a temporary directory as -r is not specified by the user.
    INFO:scripts.generate_ontology:Ontology generated in the directory /var/folders/kw/ffm8tdn57vg3msn7fhr2htpw0000gp/T/tmpc1kp_xdp.
    ```

* Note that the ``-o`` is not used at all and the data is generated in a temporary directory, as, by default, ``generate_ontology`` runs in the ``dry_run`` mode. To use the value of `--dest_path`, `--no_dry_run` has to be passed.
    ```bash
    $ generate_ontology create -i configs/example_complete_ontology_config.json -o src -r
    INFO:scripts.generate_ontology:Ontology generated in the directory /Users/mansi.gupta/user_project/src
    ```

 * Let's look at the generated file structure.
     ```bash
    tree awesome
    
    awesome
    ├── __init__.py
    └── pet
        ├── com.py
        └── __init__.py
    ```
 * Ontology generation is completed!
 
#### Clean the generated ontology
* Use `clean` mode of `generate_ontology` to clean the generated files from a given directory.
* All the arguments of `generate_ontology clean` are explained as below:

 ```bash
usage: generate_ontology clean [-h] --dir DIR [--force]

optional arguments:
  -h, --help  show this help message and exit
  --dir DIR   Generated files to be cleaned from the directory path.
  --force     If true, skips the interactive deleting offolders. Use with
              caution.
 ```
 
* Now, let's try to clean up the automatically generated files and directories *only* . 
Say, there are user-created files in the generated folder, ``user_project/src/ft``, 
called `important_stuff` that we do not want to clean up. 
     ```bash
  $ mkdir user_project/src/ft/important_stuff
  $ touch user_project/src/ft/important_stuff/file.py
  $ tree user_project
    .
    ├── configs
    │   ├── example_complete_ontology_config.json
    │   └── example_import_ontology_config.json
    └── src
        ├── custom
        │   └── user
        │       └── custom_module.py
        └── ft
            ├── important_stuff
            │   └── file.py
            └── onto
                ├── example_import_ontology.py
                └── ft_module.py
    ``` 

* Run the cleanup command and observe the directory structure. The cleanup preserves the 
partial directory structure in the case that there are files that are not generated by the framework.
    ```bash
    $ generate_ontology clean --dir user-project/src
  
    INFO:scripts.generate_ontology.__main__:Directory /Users/mansi.gupta/user_project/src not empty, cannot delete completely.
    INFO:scripts.generate_ontology.__main__:Deleted files moved to /Users/mansi.gupta/user_project/.deleted/2019-11-28-02-53-29-952561.
    ```
    
* For safety, the deleted directories are not immediately deleted but are moved to a timestamped 
 directory inside ``.deleted`` and can be restored, unless `--force` is passed. 
 
 * If the directories that are to be generated already exist, the files will be generated in the  existing directories.
 
 * Automatically generated folders are identified by an empty marker file with the name ``.generated`` or special headers. 
 If the headers or marker files are removed manually, the cleanup won't affect them.
 
