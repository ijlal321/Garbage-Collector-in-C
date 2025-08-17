# Memory Leak Detector in C + why its a bad fit for C.

`MLD` or Memory Leak detector is a hot take on implementation mark-and-sweep Garbage Collector type Memory Leak detector. 
Meaning this implementation will provides a fully functional replacement
for the standard POSIX `malloc()`, and `free()` calls.

The focus of `MLD` is to give you a demonstration of how Garbage Collectors works in languages like Java. 
And what difficulties we face when using same concept in a Language like C. 
The purpose of this code is NEVER TO GIVE YOU A PRODUCTION CODE. Use at your own risk. 
Its for educational purposes only!

## Table of contents

* [Table of contents](#table-of-contents)
* [Documentation Overview](#documentation-overview)
* [Quickstart](#quickstart)
  * [Basic usage](#basic-usage)
* [Core API](#core-api)
  * [Init Struct Database and Register Structs](#init-struct-database-and-register-structs)
  * [Init Object database and storing Objects](#init-object-database-and-storing-objects)
* [Basic Concepts](#basic-concepts)
  * [Struct Database](#struct-database)
  * [Object Database](#object-database)
* [Limitations - Why no MLD can exist?](#limitations)
  

## Documentation Overview

* Read the [quickstart](#quickstart) below to see how to get started quickly
* The [concepts](#concepts) section describes the basic concepts and design
  decisions that went into the implementation of `MLD`.
* Interleaved with the concepts, there are implementation sections that detail
  the implementation of the core components, see [hash map
  implementation](#data-structures), [dumping registers on the
  stack](#dumping-registers-on-the-stack), [finding roots](#finding-roots), and
  [depth-first, recursive marking](#depth-first-recursive-marking).


## Quickstart

### Download, compile and test

    $ git clone https://github.com/ijlal321/Memory-Leak-Detector-in-C
    $ cd Memory-Leak-Detector-in-C
    
To compile, you can use:

    $ make 
    
To clean the files

    $ make clean
    
### Basic usage

For Basic Usage, you can refer `app.c` which shows how to use this
```c
...
#include "MLD.h"
...


int main(int argc, char* argv[]) {
    ...
    /*Step 1 : Initialize a new structure database */
    struct_db_t *struct_db = calloc(1, sizeof(struct_db_t));
    mld_init_primitive_data_types_support(struct_db); // Optional
        
    /*Step 2 : Create structure record for structure emp_t*/
    static field_info_t emp_fields[] = {
        FIELD_INFO(emp_t, emp_name, CHAR,    0),
        FIELD_INFO(emp_t, emp_id,   UINT32,  0),
        FIELD_INFO(emp_t, age,      UINT32,  0),
        FIELD_INFO(emp_t, mgr,      OBJ_PTR, emp_t),
        FIELD_INFO(emp_t, salary,   FLOAT, 0),
        FIELD_INFO(emp_t, p, OBJ_PTR, 0)
    };
    /*Step 3 : Register the structure in structure database*/
    REG_STRUCT(struct_db, emp_t, emp_fields);

    /*Working with object database*/
    /*Step 1 : Initialize a new Object database */
    object_db_t *object_db = calloc(1, sizeof(object_db_t));
    object_db->struct_db = struct_db;
    
    /*Step 2 : Create some sample objects, equivalent to standard 
     * calloc(1, sizeof(student_t))*/
    student_t *abhishek = xcalloc(object_db, "student_t", 1);
    mld_set_dynamic_object_as_root(object_db, abhishek);

    student_t *shivani = xcalloc(object_db, "student_t", 1);
    strncpy(shivani->stud_name, "shivani", strlen("shivani")+1);
    // abhishek->best_colleage = shivani;

    ...

    run_mld_algorithm(object_db);
    printf("Leaked Objects : \n");
    report_leaked_objects(object_db);

    // look here, all leaked memory
}

```

## Core API

This describes the core API, see `MLD.h` for more details and the low-level API.

### Init Struct Database and Register Structs

In order to initialize and Memory Leak Detector, first we need to initialize our `struct database`.
```c
struct_db_t *struct_db = calloc(1, sizeof(struct_db_t));
```

Now time to register a struct in this database.
```c
typedef struct emp_ {

    char emp_name[30];
    unsigned int emp_id;
    unsigned int age;
    struct emp_ *mgr;
    float salary;
    int *p;
} emp_t;

...

 static field_info_t emp_fields[] = {
     FIELD_INFO(emp_t, emp_name, CHAR,    0),
     FIELD_INFO(emp_t, emp_id,   UINT32,  0),
     FIELD_INFO(emp_t, age,      UINT32,  0),
     FIELD_INFO(emp_t, mgr,      OBJ_PTR, emp_t),
     FIELD_INFO(emp_t, salary,   FLOAT, 0),
     FIELD_INFO(emp_t, p, OBJ_PTR, 0)
 };
 /*Register the structure in structure database*/
 REG_STRUCT(struct_db, emp_t, emp_fields);
```

### Init Object database and storing Objects

Initialize our `object database` which will store all of our objects.
```c
    object_db_t *object_db = calloc(1, sizeof(object_db_t));
    object_db->struct_db = struct_db;
```

Now time to add a object in this database.
An object can be set as `root` or `non root`. Root meaning it is the head and always exist on its own.
Non root means it exist inside another object, as a pointer. Like `Object1.some_ptr = Object2`.
```c
    student_t *abhishek = xcalloc(object_db, "student_t", 1);
    mld_set_dynamic_object_as_root(object_db, abhishek);
```

### Running Algorithm at demand
```c
    run_mld_algorithm(object_db);
    printf("Leaked Objects : \n");
    report_leaked_objects(object_db);
```

## Basic Concepts

The fundamental idea behind Memory Leak detector is to automate the capture of memory leaks.
This is accomplished by keeping track of all the objects created. And then filter 
all objects what are not root, and cant be reached even from any root objects.
Thus give you the simulation of a Memory Lead Detector.

We have 2 databases here. One is for storing all the structs information. Other is for storing multiple Objects.

We use `is_visited` to check for cycles. Making sure our program dont stuck in an infinite loop.
Better algorithms can be applied for leaked detection. PR is always appreciated in such scenerios 🙏.

### Struct Database
```c
typedef struct _field_info_{
    char fname [MAX_FIELD_NAME_SIZE];   /*Name of the field*/
    data_type_t dtype;                  /*Data type of the field*/
    unsigned int size;                  /*Size of the field*/
    unsigned int offset;                /*Offset of the field*/
    // Below field is meaningful only if dtype = OBJ_PTR, Or OBJ_STRUCT
    char nested_str_name[MAX_STRUCTURE_NAME_SIZE];
} field_info_t;

/*Structure to store the information of one C structure
 * which could have 'n_fields' fields*/
struct _struct_db_rec_{
    struct_db_rec_t *next;  /*Pointer to the next structure in the linked list*/
    char struct_name [MAX_STRUCTURE_NAME_SIZE];  // key
    unsigned int ds_size;   /*Size of the structure*/
    unsigned int n_fields;  /*No of fields in the structure*/
    field_info_t *fields;   /*pointer to the array of fields*/
};

/*Finally the head of the linked list representing the structure
 * database*/
typedef struct _struct_db_{
    struct_db_rec_t *head;
    unsigned int count;
} struct_db_t;
```

### Object Database

```c
/*Object Database structure definitions Starts here*/

typedef struct _object_db_rec_ object_db_rec_t;

struct _object_db_rec_{
    object_db_rec_t *next;
    void *ptr;
    unsigned int units;
    struct_db_rec_t *struct_rec;
    mld_boolean_t is_visited; /*Used for Graph traversal*/
    mld_boolean_t is_root;    /*Is this object is Root object*/
};

typedef struct _object_db_{
    struct_db_t *struct_db;
    object_db_rec_t *head;
    unsigned int count;
} object_db_t;

```


# Limitations
At this point, one thing remains. So many bright minds in world, 
why not use this method or something better to create a Memory leak detector in C ?
Lets look at some scenerios first:

## Case1: Storing Pointer in non-pointer Data types
At the end of day, a numerical numbers which is holding some pointer address.
```c
struct str1{
 char name[20];
 unsigned int designation;
}

str1 var1;
str1 var2;
...
var1.designation = var2;
```

As its just an int, our db will treat it as a number, even though it is holding an address to some space in memory.

## Case2: Indirect refrence to objects.

```c
struct manager{
 char name[20];
 int fav_employee_age;
}
struct employee{
 char name[20];
 int age;
}
 ...
`manager.fav_employee_age = employee.age; // indirect refrence.
```

## Case3: Embedded Objects.

lets compare a struct in Java and C
```Java
class str1{
    char name[32];
     struct2 str2; // this is a refrence. Not embedded object
}
```

But in C, you can do 

```c
struct str1{
    char name[32];
     struct2 str2; // embedded inside struct
}.

// or

struct str1{
    char name[32];
     struct2 * str2; // this is a refrence. Not embedded object
}
```

In java, as you dont have embedded objects,it makes garbage cleaning alot more easier.
But in C, you have to hadnle cases, when a `xmalloc` can be an embedded object or just a refrence.

## Case4: Cannot handle Unions
Unions have a fized size = Size of largest element. Thus it is difficult to store them properly in our db's.



limitations

