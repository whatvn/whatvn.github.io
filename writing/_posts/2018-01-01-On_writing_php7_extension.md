---
title: On writing php 7 extension, or How do I write a php extension
date: 2018-01-01 12:00:00
---

Our company has a long history of moving technical stack from here to there. At first everything was built on PHP, with Memcached, MySQL. Now we're moving to Java, use diferent technologies for backend, and frontend application: new webserver, new database system, some is inhouse, some is opensource database (redis, for example). 

The hard thing about maitaining both old and new architecture is we have to keep both well functional, at its best performance, with updated features, everything cannot be migrated at once, and the result is we have to strike to solved many performance issues of the old php application. 

Some A/B testing shows PHP7 makes the huge performance gain, compare with php5, with less code migration. Problem is some of our extensions work well with php5, are not compatiple with php7, most of them was built using [swig](https://github.com/swig/swig) , which is [slowly](https://github.com/swig/swig/issues/571) developed to adapt php7 changes.   

I tried to port old extension to work with php7. PHP extension is poor, old, and absolute documentation. Then I found [php-cpp](http://www.php-cpp.com/), a C++ library for developing php extension, it also supports php7. 

PHP-CPP document is well updated, easy to understand, and support most php extension features. I can quickly port one [extension](https://github.com/whatvn/51Degrees-PHP7) to php7 using its old swig defination file, also keep the old php user interface, our developer do not have to change anything in their php code. After some tests we discover php-cpp [has memory leak](https://github.com/CopernicaMarketingSoftware/PHP-CPP/issues/266) problem, and it does not support php built with threadsafe (zts) enabled. 

All problems above lead us to the option: writing native php extension. 

I have experience on writing Apache Httpd and nginx module, and those API are also poorly document, same with PHP. The best way to learn how to write nginx, or apache httpd module is reading others module, and follow their style, trial and error. So I came across php-src and read their [extensions](https://github.com/php/php-src/tree/master/ext). Although finally (1) i can write native extension for php7, I am still not happy to write php extension again. 

In this post you will find my experience on how to write a php extension.

# Some magic
 
PHP Zend API introduces many magic things, from how to declare global values, to how to return a value to user space. For example, to declare a global value, you will have to do this:

```c
ZEND_BEGIN_MODULE_GLOBALS(example)
int counter;
ZEND_END_MODULE_GLOBALS(example)
```  

Behind the scene, ***ZEND_BEGIN_MODULE_GLOBALS*** is a macro which transform your module name `example` to a struct name `_zend_example_globals`, and ***ZEND_END_MODULE_GLOBALS*** is another macro which end your defination struct, and give your struct a new name: `zend_example_globals` 

```c
#define ZEND_BEGIN_MODULE_GLOBALS(module_name)		\
	typedef struct _zend_##module_name##_globals {
#define ZEND_END_MODULE_GLOBALS(module_name)		\
	} zend_##module_name##_globals;

```

all these code will be translated to this: 

```c
typedef struct _zend_example_globals {
	int counter;
} zend_example_globals; 
```  

Why not just make thing explicit? 
Guess how we return a string value to user space, when you can example->getValue()? Here is how it is implemented: 

```c
ZVAL_STRING(return_value, (char*) value);

```

`return_value` is internal ZEND API variable, you give it a value, that value will be returned to user space, magic!


# Writing our extension 

Rest of this post I will decribe how to write a php extension, with class defination, and its method. Hope someone will find it useful. 
In php, you usually create new class instance by doing this: 

```php
$c = new myClass(); 
```


### introduction

To declare `myClass` in php extension, you will have to do these following steps:

- define a customize struct 
- map it to a zend_class_entry, write your own struct constructor and destructor, PHP will call these methods to create and destroy your class instance 
- define a method to convert from zend_class_entry to your original struct 
- and finally write a `__construct` method and register that method to your php extension 


### implementation 

here is how you create a class example in php extension (comment included in below lines of code) 

```c
// define customize struct
typedef struct {
	char* a_value;
	zend_object std; //your struct must contain zend standard object 
} example_t; 

// a method to convert zend_class_entry to your original object 

static inline example_t* example_obj_fetch(zend_object* obj) {
    return (example_t*) ((char*) (obj) - XtOffsetOf(example_t, std));
}

// later you can use Z_EXAMPLE_P(zend_object) to retrieve your original object
#define Z_EXAMPLE_P(zv) example_obj_fetch(Z_OBJ_P((zv)))

// object constructor and destructor 

static inline zend_object* example_obj_new(zend_class_entry *ce) {
    example_t* example;
    example = ecalloc(1, sizeof (example_t) + zend_object_properties_size(ce));
    zend_object_std_init(&example->std, ce);
    object_properties_init(&example->std, ce);
    example->std.handlers = &example_obj_handlers;
    return &example->std;
}

static void example_obj_free(zend_object *object) {
    example_t* example; 
    example = example_obj_fetch(object);
    if (!example) {
        return;
    }
    zend_object_std_dtor(&example->std TSRMLS_CC);
    efree(example);
}

// declare a standard zend object handler 

static zend_object_handlers example_obj_handlers;

// map your class to zend_class_entry, have to do it within PHP_MINIT_FUNCTION
static zend_class_entry *example_ce;
PHP_MINIT_FUNCTION(example) {
	.....
	zend_class_entry ce;
    	INIT_CLASS_ENTRY(ce, "example", example_methods);
    	example_ce = zend_register_internal_class(&ce TSRMLS_CC);
   	example_ce->create_object = example_obj_new;
    	memcpy(&example_obj_handlers, zend_get_std_object_handlers(), sizeof (example_obj_handlers));
    	example_obj_handlers.offset = XtOffsetOf(example_t, std);
    	example_obj_handlers.free_obj = example_obj_free;
	.....
}

// construct method

PHP_METHOD(example, __construct) {
    char* value;
    strlen_t len = 0;

    if (zend_parse_parameters(ZEND_NUM_ARGS(), "s", &value, &len) == FAILURE) {
        RETURN_FALSE;
    }
    return_value = getThis();
    example_t* example;
    example = Z_EXAMLE_P(return_value);
    example->a_value = value;
}


// register construct method to extension 

ZEND_BEGIN_ARG_INFO_EX(arginfo_example_none, 0, 0, 0)
ZEND_END_ARG_INFO()
zend_function_entry example_methods[] = {
	PHP_ME(example, __construct, arginfo_example_none, ZEND_ACC_CTOR | ZEND_ACC_PUBLIC)
}

``` 

#### static method 

In php you can call a static method, by doing `example::some_static_method()`, you also can define static method in php extension, below is the function to set `a_value` for class example, but it's static method: 

```c
PHP_METHOD(example, set_value) {

    char* value;
    strlen_t len = 0;
    if (zend_parse_parameters(ZEND_NUM_ARGS(), "s", &value, &len) == FAILURE) {
        RETURN_FALSE;
    }
    object_init_ex(return_value, example_ce);
    example_t* example;
    example = Z_EXAMLE_P(return_value);
    example->a_value = value;
}

```

because we do not construct method by calling `new example()`, but `example::set_value("something")`, we must construct object ourself inside our method, this is where we call `object_init_ex` with return_value and example class entry we defined before, register this static method so we can call it from php user space: 

```
zend_function_entry example_methods[] = {
    PHP_ME(example, __construct, arginfo_example_one, ZEND_ACC_CTOR | ZEND_ACC_PUBLIC)
    PHP_ME(example, set_value, arginfo_example_one, ZEND_ACC_STATIC | ZEND_ACC_PUBLIC)
    PHP_FE_END
}

```

Now we can use `$e = example::set_value("something");` from php code, we will get the same result as `$e = new example("something");`


## retrieve value 

To get value you assigned when initialised class example by `new example("A value")`, we need to define a get method 

```c
PHP_METHOD(example, get) {
    example_t* example;
    if (zend_parse_parameters_none() == FAILURE) {
        RETURN_FALSE;
    }
    example = Z_EXAMLE_P(getThis());
    ZVAL_STRING(return_value, (char*) example->a_value);
}
```


## and register it 

```c
zend_function_entry example_methods[] = {
    PHP_ME(example, __construct, arginfo_example_one, ZEND_ACC_CTOR | ZEND_ACC_PUBLIC)
    PHP_ME(example, get, arginfo_example_none, ZEND_ACC_PUBLIC)
    PHP_FE_END
};

```

## initialize & shutdown function 

A standard php extension has to implement (at least) 2 global function: PHP_MINIT_FUNCTION, PHP_MSHUTDOWN_FUNCTION, and here is it:

```c
PHP_MINIT_FUNCTION(example) {

    zend_class_entry ce;
    INIT_CLASS_ENTRY(ce, "example", example_methods);
    example_ce = zend_register_internal_class(&ce TSRMLS_CC);
    example_ce->create_object = example_obj_new;
    memcpy(&example_obj_handlers, zend_get_std_object_handlers(), sizeof (example_obj_handlers));
    example_obj_handlers.offset = XtOffsetOf(example_t, std);
    example_obj_handlers.free_obj = example_obj_free;

    return SUCCESS;

}

PHP_MSHUTDOWN_FUNCTION(example) {
    return SUCCESS;
}

```

PHP_MINIT_FUNCTION to register our class entry, and how to handle it. it's where we do step 2, map zend standard class entry to our struct, and define its object handler.  
Finally, register own extension, and everything will be ready to go:

## register php module 

```c
zend_module_entry example_module_entry = {
    STANDARD_MODULE_HEADER,
    "example",
    NULL,
    PHP_MINIT(example),
    PHP_MSHUTDOWN(example),
    PHP_RINIT(example),
    NULL,
    PHP_MINFO(example),
    EXAMPLE_VERSION,
    STANDARD_MODULE_PROPERTIES
};

ZEND_GET_MODULE(example)

```

## configuration/metadata

To install php extension, php require a metadata file which provides extension name, and extension dependencies, where to get include header, which files need to be compiled along with our main source file.
Because our extension is pretty simple, we just need to provide simpile metadata, put lines below in config.m4: 

```
dnl $Id$
dnl config.m4 for extension example

dnl Comments in this file start with the string 'dnl'.
dnl Remove where necessary. This file will not work
dnl without editing.

dnl If your extension references something external, use with:

PHP_ARG_WITH(example, for example support,
dnl Make sure that the comment is aligned:
[  --with-example            Include example support])

dnl Otherwise use enable:

dnl PHP_ARG_ENABLE(example, whether to enable example support,
dnl Make sure that the comment is aligned:
dnl [  --enable-example           Enable example support])

if test "$PHP_EXAMPLE" != "no"; then
  dnl Write more examples of tests here...

  dnl # --with-example -> check with-path
  SEARCH_PATH="/usr/local /usr"     # you might want to change this
  EXAMPLE_DIR=$PHP_EXAMPLE
  dnl
  if test -z "$EXAMPLE_DIR"; then
      EXAMPLE_CFLAGS= "$EXAMPLE_CFLAGS"
  fi


  dnl
  PHP_SUBST(EXAMPLE_CFLAGS)
  PHP_SUBST(EXAMPLE_SHARED_LIBADD)

  PHP_NEW_EXTENSION(example, example.c, $ext_shared)
fi

```
 
## installation 

To install example extension, do the following steps: 

```bash
phpize
./configure
make && make install 

```


Let php know we need example extension, add `extension=example.so` in php.ini 


## usage

Now we can use example extension likes this:

```php
<?php

$c = new example("Hello");

echo $c->get(), "\n";

```

and done. 

## source code 

I put example extension used in this post in this [repository](https://github.com/whatvn/php7-example-module). 



# Conclusion:

Actually I can not understand why php zend api has so much complicated macro, note that those was mentioned in this post is just some of many magical things in ZEND API. Maybe with a better document, things will be easier and developer will write more php extension for php community. 


(1) Here is my [extension](https://github.com/whatvn/php7-51degrees) for 51degrees device detection library(2)

(2): [device detection](https://github.com/51Degrees/Device-Detection)  



