parallel
========

[![Build Status](https://travis-ci.org/krakjoe/parallel.svg?branch=develop)](https://travis-ci.org/krakjoe/parallel)
[![Build status](https://ci.appveyor.com/api/projects/status/cppfcu6unc0r0h0b?svg=true)](https://ci.appveyor.com/project/krakjoe/parallel)
[![Coverage Status](https://coveralls.io/repos/github/krakjoe/parallel/badge.svg?branch=develop)](https://coveralls.io/github/krakjoe/parallel)

A succinct parallel concurrency API for PHP 7:

```php
final class parallel\Runtime {
	/*
	* Shall construct a new Runtime
	* @param string bootstrap (generally an autoloader)
	* @param array  ini configuration
	*/
	public function __construct(string $bootstrap, array $configuration);
	/**
	* Shall construct a new Runtime
	* @param string bootstrap (generally an autoloader)
	**/
	public function __construct(string $bootstrap);
	/**
	* Shall construct a new Runtime
	* @param array ini configuration
	**/
	public function __construct(array $configuration);

	/*
	* Shall schedule a Closure for execution passing optional arguments
	* @param Closure handler
	* @param argv
	* Note: A Future shall only be returned if $handler contains a return statement
	*	Should the caller ignore the return statement in $handler, an exception
	*	shall be thrown
	*/
	public function run(Closure $handler, array $argv = []) : ?\parallel\Future;
	
	/*
	* Shall request the Runtime shutdown
	* Note: Closures scheduled for execution will be executed
	*/
	public function close() : void;

	/*
	* Shall kill the Runtime
	* Note: Closures scheduled for execution will not be executed,
	*	currently running Closure will be interrupted.
	* Note: Cannot interrupt syscalls
	*/
	public function kill() : void;
}

final class parallel\Future {
	/*
	* Shall wait until the value becomes available
	*/
	public function value() : mixed;
}
```

Implementation
==============

In PHP there was only one kind of parallel concurrency extension API, the kind that pthreads and pht try to implement:

  * They are hugely complicated
  * They are error prone
  * They are difficult to understand
  * They are difficult to deploy
  * They are difficult to maintain

`parallel` takes a totally different approach to these two by providing only a very small, easy to understand API that exposes the power of parallel concurrency without any of the aforementioned headaches.

By placing some restrictions upon what a Closure intended for parallel execution can do, we remove almost all of the cognitive overhead of using threads, we hide away completely any mutual exclusion and all the things a programmer using threads must think about.

In practice this means Closures intended for parallel execution must not:

  * accept or return by reference
  * accept or return objects
  * execute a limited set of instructions

Instructions prohibited directly in Closures intended for parallel execution are:

  * declare (anonymous) function
  * declare (anonymous) class
  * lexical scope access
  * yield

__No instructions are prohibited in the files which the Runtime may include.__

Requirements
============

  * PHP 7.1+
  * ZTS
  * <pthread.h>

Configuring the Runtime
=======================

The optional bootstrap file passed to the constructor will be included before any code is executed, generally this will be an autoloader.

The configuration array passed to the constructor will configure the Runtime using INI.

The following options may be an array, or comma separated list:

  * disable_functions
  * disable_classes

The following options will be ignored:

  * extension - use dl()
  * zend_extension - unsupported

All other options are passed directly to zend verbatim and set as if set by system configuration file.

Hello World
===========

```php
<?php
$runtime = new \parallel\Runtime();

$future = $runtime->run(function(){
	for ($i = 0; $i < 500; $i++)
		echo "*";

	return "easy";
});

for ($i = 0; $i < 500; $i++) {
	echo ".";
}

printf("\nUsing \\parallel\\Runtime is %s\n", $future->value());
```

This may output something like (output abbreviated):

```
.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*.*
Using \parallel\Runtime is easy
```

