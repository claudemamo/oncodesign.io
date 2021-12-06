+++
title = "Smooks DFDL Filter in a Cross Domain Solution"
date = 2021-12-05T20:45:00+02:00
tags = ["Smooks", "DFDL", "Apache Daffodil", "Smooks 2", "cross domain solution", "cyber security"]
+++

In the cyber security space, a [cross domain solution](https://www.cyber.gov.au/acsc/view-all-content/publications/fundamentals-cross-domain-solutions) is a bridge connecting two different security domains, permitting data to flow from one domain into another while minimising the associated security risks.
A filter, or more formally a [verification engine](https://www.ncsc.gov.uk/collection/cross-domain-solutions/using-the-principles/content-based-attack-protection), is a suggested component in a cross domain solution.

<br/>
<img src="/images/cross-domain-solution.png" alt="Cross Domain Solution"/>
<br/>

A filter inspects for anomalies the content flowing through the bridge. Data failing inspection
is captured for investigation by the security team. Given this brief description, I argue for the following properties in a verification engine:

* Validation: syntactically and semantically validates complex data formats

* Content-based routing: routes valid data to its destination while invalid data is routed to a different channel

* Data streaming: filters data whatever the size which implies parsing the data and then reassembling it

* Open to scrutiny: the filter's source code should be available for detailed evaluation. Open source software, by definition, is a prime example of this property

[Smooks](https://www.smooks.org/v2/), with its strong support for SAX event streams and XPath-driven routing, alongside [DFDL](https://daffodil.apache.org/)'s transformation and validation features, manifest the above properties. Picture a situation where [NITF](https://en.wikipedia.org/wiki/National_Imagery_Transmission_Format) (National Imagery Transmission Format) files need to be imported from an untrusted system into a trusted one. Widely used in national security systems, NITF is a binary file format that encapsulates imagery (e.g., JPEG) and its metadata. As part of the import, a filter is needed to unpack the NITF stream, ensure it's as expected, and repack it before being routed to its destination. Should verification fail, the bad data is put aside for human intervention. This is how such a filter is described in a Smooks config:

<script src="https://gist.github.com/claudemamo/56d73cf6af94a6eae4928beaf60a0e92.js?file=smooks-config.xml"></script>

[`dfdl:parser`](https://gist.github.com/claudemamo/56d73cf6af94a6eae4928beaf60a0e92#file-smooks-config-xml-L6) ingests the binary content from the input source (i.e., the untrusted system) and validates the produced event stream. Driving the transformation and validation behaviour is the XML schema `nitf.dfdl.xsd` which is copied from the [public DFDL schema repository](https://github.com/DFDLSchemas/NITF) and tweaked in order to route the data depending on its correctness. The first tweak is to turn invalid NITF data into hex with an `InvalidData` element wrapped around it like so:

<script src="https://gist.github.com/claudemamo/56d73cf6af94a6eae4928beaf60a0e92.js?file=bad-data.xml"></script>

The second tweak is to nest legit data within a `ValidData` element:

<script src="https://gist.github.com/claudemamo/56d73cf6af94a6eae4928beaf60a0e92.js?file=good-data.xml"></script>

Following ingestion, Smooks will either:

&nbsp;&nbsp; a. [Fire the happy path](https://gist.github.com/claudemamo/56d73cf6af94a6eae4928beaf60a0e92#file-smooks-config-xml-L8-L20) on encountering events that are not descendants of the `InvalidData` node [^1]. A pipeline executes [`dfdl:unparser`](https://gist.github.com/claudemamo/56d73cf6af94a6eae4928beaf60a0e92#file-smooks-config-xml-L16-L17) to reassemble the data in its original format, to then go on and replace the XML execution result stream with the reassembled binary data which will be delivered to the destination (i.e., the trusted sytem).

&nbsp;&nbsp; b. [Fire the unhappy path](https://gist.github.com/claudemamo/56d73cf6af94a6eae4928beaf60a0e92#file-smooks-config-xml-L22-L33). A different pipeline streams `InvalidData` to an output resource named `deadLetterStream`.

Voilà, we've implemented a low-cost efficient verification engine with a few lines of XML. The complete source code of this [example is available online](https://github.com/smooks/smooks-examples/tree/master/cross-domain-solution).

[^1]: Smooks 2 release candidate 1 will feature support for selector negation.
