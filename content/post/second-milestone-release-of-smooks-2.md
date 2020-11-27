+++
title = "Second Milestone Release of Smooks 2"
date = 2020-11-27
tags = ["Smooks", "SAX NG", "2.0.0-M2", "Visitor Memento"]
+++

<img src="/images/smooks-logo.png" alt="Smooks logo" style="max-width:70%"/>

We're delighted to announce the [second milestone](https://github.com/smooks/smooks/releases/tag/v2.0.0-M2) release of Smooks 2. 
The second milestone is geared towards simplifying Smook's API along with a few goodies. Here’s a rundown of the new notable 
features:

### SAX NG

Earlier versions of Smooks could have visitors applying operations on either DOM nodes or SAX events. To ensure interoperability, 
one would implement a visitor supporting both DOM node and SAX event processing which meant implementing two 
different application APIs:

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=MyDomAndSaxVisitor.java"></script>

The above broadly translates into two different execution paths and all the baggage it entails. Smooks 2.0.0-M2 unifies 
the DOM and SAX visitor APIs without sacrificing convenience or performance. The new SAX NG filter drops the API 
distinction between DOM and SAX. Instead, it streams SAX events as **partial** DOM elements to SAX NG visitors:

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=MySaxNgVisitor.java"></script>

Traditional DOM trees are still supported with the help of the _max.node.depth_ global config parameter. The _max.node.depth_ 
knob instructs the SAX NG filter to build DOM trees for the targeted elements such that the tree's maximum depth is equal 
to the _max.node.depth_ value.

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=smooks-config.xml"></script>

It’s all transparent from a visitor’s perspective. A visitor only sees an element but with the caveat that an element may 
have child nodes when _max.node.depth_ is greater than 1.

The latest cartridge releases have been migrated to SAX NG. We recommend future visitor implementations extend the
_org.smooks.delivery.sax.ng.SaxNgVisitor_ interface given that it supersedes the _org.smooks.delivery.dom.DOMVisitor_ 
and _org.smooks.delivery.sax.SAXVisitor_ interfaces.


### JSR Annotations

Smooks-specific annotations were dropped in favour of JSR annotations. Smooks 1 resources would have their dependencies
injected like so:

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=MySmooks1Resource.java"></script>

Smooks 2.0.0-M2 does away with all these adhoc annotations and provides a single [_@Inject_](https://javaee.github.io/javaee-spec/javadocs/javax/inject/Inject.html) for this purpose:

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=MySmooks2Resource.java"></script>

Additionally, the *@Initialize* and *@Uninitialize* lifecycle annotations were replaced with the standard [_@PostConstruct_](https://docs.oracle.com/javase/8/docs/api/javax/annotation/PostConstruct.html)
and [_@PreDestroy_](https://docs.oracle.com/javase/8/docs/api/javax/annotation/PreDestroy.html) annotations, respectively.


### EDIFACT Java Bindings

By popular demand, we’re generating and distributing the Java bindings for the EDIFACT schemas starting from the M2 release of 
the EDIFACT cartridge. The bindings are available from Maven Central at the coordinates: `org.smooks.cartridges.edi:`_[message version/release]_`-edifact-binding:2.0.0-M2`.
For instance, the D03B EDIFACT binding dependency can be declared in your POM with:

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=pom.xml"></script>

Visit the [java-to-edifact](https://github.com/smooks/smooks-examples/tree/v1.0.1/java-to-edifact) project in the examples catalogue to view Java bindings used together with Smooks. 


### Selector Namespace Prefixes

In Smooks 1, you would bind selector namespace prefixes as follows:

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=smooks-1-config.xml"></script>

This is no longer required. As of 2.0.0-M2, you can now bind a selector namespace prefix just like any other XML namespace 
prefix:  

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=smooks-2-config.xml"></script>

Furthermore, the new [_smooks-2.0.xsd_](https://raw.githubusercontent.com/smooks/smooks.github.io/v2.0.0-M2/xsd/smooks-2.0.xsd) removes the _default-selector-namespace_ and _selector-namespace_ XML attributes in favour of 
declaring namespaces within the standard _xmlns_ attribute from the _smooks-resource-list_ element. _smooks-2.0.xsd_ 
also removes the _default-selector_ attribute from the _smooks-resource-list_ element: some may not like this change but we think 
a config is easier to comprehend when selectors are explicit.


### Visitor Memento

Visitor [mementos](https://en.wikipedia.org/wiki/Memento_pattern) are a convenient way to store, track, and retrieve a visitor's 
state from the execution context. In the past, given that visitors are not thread-safe, the general approach to holding onto state 
between visits was to stash the state inside the execution context and retrieve it later on. This led to boilerplate code 
because oftentimes the state to be retrieved depended on the fragment being processed and the visitor instance. Visitor mementos were
introduced in order to eliminate this boilerplate code.

A great example showcasing mementos is accumulating character data in a visitor while SAX events are streamed:

<script src="https://gist.github.com/claudemamo/db790ce86c7ac65581f62c7b4cd4b2a1.js?file=MyTextAccumulatorVisitor.java"></script>

_getMementoCareTaker()_ is a new addition to the _ExecutionContext_ interface. A _MementoCareTaker_ manages mementos 
on behalf of visitors. In line 19, the _MementoCareTaker_ stashes each chunk of read character data. It then goes on to 
restore the collected character data in the visitor's _visitAfter(Element, ExecutionContext)_ method as shown in lines 29-30.
Observe that the element and visitor objects serve as keys for saving as well as restoring the memento's state. Consult 
the [Javadocs](https://github.com/smooks/smooks.github.io/tree/v2.0.0-M2/javadoc/v2.0.0-M2/smooks) to take a deep dive into mementos.