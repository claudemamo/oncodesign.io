+++
date = "2016-07-11T13:04:38+02:00"
title = "bean mappers"
draft = true
+++

As developers, we like partitioning a system into layers to decouple dependencies. We create classes for each layer allowing us to easily insulate the changes made to one layer from the rest of the system. For statically typed languages like Java, moving data from one layer to another requires us to map the object, encapsulating the data, to the corresponding object in the destination layer. Many enterprise applications have often some sort of object mapping going on. For the purposes of this post, I broadly define object mapping as copying an in-memory object's state to another in-memory object such that both objects may or may not belong to the same complex class. In technical discussions, you'll often hear the former object referred to as the source while the latter as the target or destination.

Object mapping isn't constrained to the domain of enterprise applications. In fact, object mapping often goes hand-in-hand with enterprise integration. When integrating applications, an application client library for a source system will return POJOs that aren't what the application client library for the target system expects. The object mapper takes the objects from the source and...

Object Mapper

Manually enumerating every mapping for each of the source and target objects' fields is a tedious job and prone to mistakes. We typically rely on libraries, or object mappers, to automate this behaviour. Part of this automation, for object mappers in Java, is achieved by following bean conventions. That is, the object mapper expects a getter for each class field to be mapped in the source object and a setter for each corresponding field in the target object. It's for this reason object mappers are commonly known as bean mappers. Normally, a bean mapper discovers at run-time the getters and setters as well as copies values normally rely on reflection to know which fields to copy but, as we will see further on, this is not the only way.

Configuration

In some cases, the source and target classes won't be exact replicas of each other so one cannot hope that a bean mapper will know which field goes where without prior configuration. Rudimentary bean mappers like Apache Common's BeanUtilsBean only offer a Java API to configure mappings. For this particular mapper, the mapping is customised by implementing the Converter interface and hooking it up with the...:

Using Java for configuring the mappings might be sufficient if the amount of object mapping is small or you need the control structures that the Java language provides. However, in other cases, a declarative language can be of great help in reducing the time and complexity to configure mappings. Dozer is good example of a mapper that supports declarative configurations in XML or Java annotations. Taking the example above, in Dozer configuration, it becomes the following:



Mapping time: Dynamic vs Static vs Byte-code generation

The bulk of the bean mappers leverage reflection to invoke the getter and setters at run-time. What should happen if the getter or setter is missing is implementation specific. Some mappers simply throw an exception. Others ignore the getter/setter. Though typically, you can override the default behaviour. Despite its convenience, it's not very safe to invoke methods using reflection given that the compiler cannot guarantee whether the method to be invoked at run-time exists.



POJO-to-POJO vs. O

Bean mappers can be strictly

Most of us are likely used to having the....  This will work if you have objects occupying a few bytes to kilobytes of main memory. But as soon....  

Depth: Deep vs shallow

There are quite a few object mappers

Configuration
Mapping time
type of object (i.e., only beans, more flexible like constructors, totally, same class)
Stream vs all-at-once






On most Java projects I've seen the requirement to map




Dozer
Smooks
BeanUtilsBean
