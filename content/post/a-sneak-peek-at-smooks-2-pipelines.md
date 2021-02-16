+++
title = "A Sneak Peek at Smooks 2 Pipelines"
date = 2021-02-16
tags = ["Smooks", "Pipelines", "2.0.0-M3", "EDI", "EDIFACT", "DFDL" ]
+++

<br/>

<figure>
    <img src="/images/pipeline.jpg" alt="pipeline" style="max-width:90%"/>
    <figcaption><sub><sup>*</sup></sub></figcaption>
</figure>


We’re inching ever closer to releasing Smooks 2 milestone 3. Featuring in this release will be the powerful ability to 
declare pipelines. A pipeline is a flexible, yet simple, Smooks construct that isolates the processing of a targeted event 
from its main processing as well as from the processing of other pipelines. In practice, this means being able to compose 
any series of transformations on an event outside the main execution context before, optionally, directing the pipeline 
output to the execution result stream or to other destinations.

Under the hood, a pipeline is just another instance of Smooks. This is self-evident from the Smooks config element declaring 
a pipeline:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.1.xml"></script>

_core:smooks_ fires a nested Smooks execution whenever an event in the stream matches the _filterSourceOn_ selector. The optional 
_core:action_ element tells the nested Smooks instance what to do with the pipeline’s output. Out-of-the-box actions include 
support for merging the output with the result stream, directing the output to a different stream, or binding the output 
to the execution context’s bean store.

_core:config_ expects a standard _smooks-resource-list_ defining the pipeline’s behaviour. It’s worth highlighting that the 
nested _smooks-resource-list_ element behaves identically to the outer one, and therefore, it accepts resources like visitors, 
readers, and even pipelines (a pipeline within a pipeline!). Moreover, a pipeline is transparent to its nested resources, 
so a resource’s behaviour will remain the same whether it’s declared inside a pipeline or outside it.

Let’s illustrate the power of pipelines in a real-world use case. Consider this classic data integration problem:

>An organisation is required at the end of day to churn through GBs of customer orders captured in a CSV file and hosted 
on an FTP server. Each individual order is to be communicated to the inventory system in JSON and uploaded as XML to a CRM service. 
Furthermore, an [EDIFACT](https://unece.org/trade/uncefact/introducing-unedifact) document aggregating the orders needs to be exchanged with the supplier.

A scalable solution powered by Smooks pipelines can be conceptualised to:

<img src="/images/smooks-2-pipelines.png" alt="Smooks 2 pipelines"/>

Every data integration point is implemented in its own pipeline. Smooks converts the CSV input into a stream of events, 
triggering the pipelines at different points in the stream. The document root event (i.e., _#document_ or _file_) triggers the 
EDI pipeline while _record_ (i.e., an order) events drive the inventory and CRM pipelines. Time to get cracking and implement 
the solution in a Smooks config.


### Reader

The first step to implementing the solution is to turn the CSV file stream into an event stream using the [DFDL](https://daffodil.apache.org/) parser added 
in Smooks 2.0.0-M1 (alternatively, a simpler but less flexible reader from the CSV cartridge can be used for this purpose):

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.2.xml"></script>

_csv.dfdl.xsd_ holds the DFDL schema for translating the CSV into XML:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=csv.dfdl.xsd"></script>

For the above schema, the _dfdl:parser_ turns a CSV stream such as the following:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=input.csv"></script>

into the SAX event stream:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=event-stream.xml"></script>

The events of interest are the _file_, _record_, and _item_ events. Coming up next are the pipeline configurations.


### Inventory Pipeline

The inventory pipeline maps each order to JSON and writes it to an HTTP stream where the organisation’s inventory system 
is reading on the other end:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.3.xml"></script>

The _filterSourceOn_ XPath expression selects the event/s for _core:smooks_ to visit. In this snippet, the pipeline visits 
_record_ events, including their child _item_ events. The root event in the pipeline’s context is _record_; not _file_. 
Although the parent of the _item_ event is _record_, the latter has no parent node given that it’s the root event in the 
inventory pipeline. It follows then that the _#document_ selector in the inventory pipeline is equivalent to the _record_ 
selector.

_maxNodeDepth_ is set to 0 (i.e., infinite) so as to append the item events/nodes to the record tree instead of discarding 
them. By default, Smooks never accumulates child events in order to keep a low-memory footprint but in this instance we 
assume the number of item events within a record node is manageable within main memory.

_InventoryVisitor_ visits record events and writes its output to a stream declared within the _core:action_ element 
(the output stream is registered programmatically). Drilling down to the _InventoryVisitor_ class will yield the Java code:  

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=InventoryVisitor.java"></script>

The _AfterVisitor_ implementation leverages the popular Jackson library to serialise the record element into JSON which 
is then transparently written out to the inventoryOutputStream with `Stream.out(executionContext).write(...)`.

### CRM Pipeline

Like the inventory pipeline, the CRM pipeline visits each _record_ event:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.4.xml"></script>

Spot the differences between the inventory and CRM pipelines. This pipeline omits _core:action_ because the _CrmVisitor_ resource 
HTTP POSTs the result directly to the CRM service. Another notable difference is the appearance of a new Smooks 2 reader. 

_core:node-reader_ delegates the pipeline event stream to an enclosed _ftl:freemarker_ visitor which instantiates the underneath 
template with the selected _record_ event as a parameter:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=purchaseOrder.xml.ftl"></script>

_core:node-reader_ goes on to feed Freemarker’s instantiated template to _CrmVisitor_, in other words, _core:node-reader_ converts 
the pipeline event stream into one _CrmVisitor_ can visit:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=CrmVisitor.java"></script>

_CrmVisitor_’s code should be self-explanatory. Observe _org.asynchttpclient.AsyncHttpClient_ is referenced in `visitAfter(...)` 
to perform a non-blocking HTTP POST. All framework execution in Smooks happens in a single thread so blocking calls should 
be avoided to keep the throughput rate acceptable.


### EDI Pipeline

The EDI pipeline requires more thought than the earlier pipelines because (a) it needs to aggregate the orders, (b) wrap 
a header and footer around the aggregated orders, and (c) convert the event stream into one that the _edifact:unparser_ visitor 
can digest. After which it needs to write _edifact:unparser_’s EDI output to the result stream, overwriting the XML stream 
produced from the _dfdl:parser_ reader. The latter is accomplished with the _replace_ pipeline action:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.5.xml"></script>

The selector for this pipeline is set to _#document_; not _record_. _#document_, which denotes the opening root tag, leads to the 
pipeline firing only once, necessary for creating a single EDIFACT document header and footer. We’ll worry later about how 
to enumerate the _record_ events.

The subsequent pipeline config leverages _core:node-reader_, introduced in the previous pipeline, to convert the event stream 
into a stream _edifact:unparser_ (covered furthered on) can understand:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.6.xml"></script>

_core:node-reader_ delivers the pipeline event stream to its child visitors. Triggered FreeMarker visitors proceed to materialise 
their templates and have their output fed to the _edifact:unparser_ for serialisation. The previous snippet has a lot to unpack 
therefore a brief explanation of each enclosed visitor’s role is in order.


* ##### Header Visitor

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.7.xml"></script>

On encountering the opening root tag, this FreeMarker visitor feeds the XML header from the _header.xml.ftl_ template to the 
_edifact:unparser_. The content of _header.xml.ftl_, shown next, is static for illustration purposes. In the real world, one 
would want to generate dynamically data elements like sequence numbers.

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=header.xml.ftl"></script>


* ##### Body Visitor

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.8.xml"></script>

The pipeline within a pipeline collects the item events and appends them to the record tree (`maxNodeDepth="0"`) before 
pushing the record tree down to the enclosed FreeMarker visitor. As a side note, this config could be simplified by unwrapping 
the FreeMarker visitor and setting the _maxNodeDepth_ attribute to 0 in the _#document_ pipeline but then that would unfortunately 
lead to the pipeline reading the entire event stream into memory.

The _body.xml.ftl_ template warrants a closer look:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=body.xml.ftl"></script>

FreeMarker materialises and feeds the above segment group to the _edifact:unparser_ for each visited _item_ event.

* ##### Footer Visitor

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.9.xml"></script>

This FreeMarker visitor is fired on the closing root tag, following the serialisation of the header and body. The footer 
residing in _footer.xml.ftl_ is also fed to the _edifact:unparser_:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=footer.xml.ftl"></script>


The final piece to the solution is to configure the _edifact:unparser_:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.10.xml"></script>

As per the _unparseOnElement_ wildcard selector, the pipeline delivers all events generated from the _core:node-reader_ visitors 
to _edifact:unparser_ to be serialised into EDIFACT before the pipeline merges the serialised events with the result stream.

Let’s take a step back to view the complete Smooks config:

<script src="https://gist.github.com/claudemamo/a65e33b1ee62984cb507b77baea75100.js?file=smooks-config.xml"></script>

A functional version of the [solution is available on GitHub](https://github.com/smooks/smooks-examples/tree/master/pipelines). Take it for a spin. Your feedback on [Smook’s user forum](https://groups.google.com/g/smooks-user) is most welcome.

<br/>

<sub><sup>* ["Pembroke Reverse Osmosis Plant pipeline"](https://www.flickr.com/photos/54758495@N00/11431784985) by [Rrrodrigo](https://www.flickr.com/photos/54758495@N00) is licensed under [CC BY-NC 2.0](https://creativecommons.org/licenses/by-nc/2.0/?ref=ccsearch&atype=rich)</sup></sub>