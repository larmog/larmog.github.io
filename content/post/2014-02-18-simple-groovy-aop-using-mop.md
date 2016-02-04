+++
date = "2014-08-18T21:37:45+01:00"
draft = false
title = "Simple Groovy AOP using MOP"
author = "Lars Mogren"
tags = [ "Development", "Groovy" ]
categories = [ "Development" ]

+++

I was looking for a simple solution for adding cross cutting concerns to Groovy
classes. The most obvious solution was to implement `GroovyInterceptable` but I wanted a
less intrusive solution. After a bit of googling I've stumbled across
`DelegatingMetaClass`.

<!--more-->

Here is a simple interceptor that works like a around advice. It only logs all method
calls for a class.

```java

class Interceptor extends DelegatingMetaClass {
  	Interceptor(final Class cls) {
      	super(cls)
      	initialize()
  	}

  	public Object invokeMethod(Object obj, String method, Object[] args) {
      	String cls = obj.class.simpleName
      	println "before: $cls.$method, args:$args -->"
      	def val = null
      	try {
          	val = super.invokeMethod(obj, method, args)
      	} catch(Exception e) {
          	println "after: $cls.$method, has thrown:$e <--"
          	throw e
      	}
      	println "after: $cls.$method, return value:$val <--"
      	return val;
  	}

  	def static injectIn(Class cls) {
      	cls.metaClass = new Interceptor(cls)
  	}
}

```

We can now use the `Interceptor` to trace calls to `ArrayList` like this:
```java
	Interceptor.InjectIn(ArrayList)

	def list = []
	list << "Joe"
```
If we run the above it will produce:
```shell
	before: ArrayList.leftShift, args:[Joe] -->
	after: ArrayList.leftShift, return value:[Joe] <--
```
It works width mixin's too, for example:
```java
	class Person {
		def name

		def sayHello() { println "Hi, my name is $name!"}
	}

	class Dancer {
		def dance() { println "I can dance" }
	}

	class Singer {
		def sing() { println "I can sing" }
	}
```
We can now create a person who dance and sing. The `Interceptor` must be injected
after the mixin:
```java
	Person.mixin Singer
	Person.mixin Dancer
	Interceptor.injectIn(Person)

	def p = new Person(name: "Jill")
	p.sayHello()
	p.sing()
	p.dance()
```
This will produce the following output:
```shell
	before: Person.sayHello, args:[] -->
	before: Person.println, args:[Hi, my name is Jill!] -->
	Hi, my name is Jill!
	after: Person.println, return value:null <--
	after: Person.sayHello, return value:null <--
	before: Person.sing, args:[] -->
	I can sing
	after: Person.sing, return value:null <--
	before: Person.dance, args:[] -->
	I can dance
	after: Person.dance, return value:null <--
```
 As we can see in the example above the `println`-method is part of the Groovy object.
