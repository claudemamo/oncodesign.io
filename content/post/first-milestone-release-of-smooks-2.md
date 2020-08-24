+++
title = "First Milestone Release of Smooks 2"
date = 2020-08-24T08:15:00+02:00
tags = ["Smooks", "DFDL", "Apache Daffodil", "2.0.0-M1"]
+++

<img src="/images/smooks-logo.png" alt="Smooks logo" style="max-width:70%"/>

Following my [short stint](https://oncodesign.io/2014/09/16/the-trials-of-smooks/) with [Smooks](https://www.smooks.org/), I joined its developer community to contribute back. As technologies like [Apache 
Daffodil](https://daffodil.apache.org/) mature, Smooks has become more relevant today than it ever was. Thanks to the guidance of Smooks creator [Tom Fennelly](https://github.com/tfennelly), 
the [code repositories](https://github.com/smooks/) are bursting again with commit activity in preparation for the release of Smooks 2! The first milestone 
was [published to Maven Central](https://search.maven.org/search?q=org.smooks) a few days ago and, among the several notable improvements, we are pleased to announce the 
new [DFDL](https://en.wikipedia.org/wiki/Data_Format_Description_Language) cartridge which forms the technological underpinnings of the overhauled EDI cartridge. As with any major release, 
there are breaking changes like the dropping of support for Java 7. What follows is a brief overview of the changes:

* DFDL cartridge
    * DFDL is a specification for describing file formats in XML. The DFDL cartridge leverages Apache Daffodil to parse 
    files and unparse XML. This opens up Smooks to an incredible number of file formats like SWIFT, ISO8583, HL7, and many more.
* Complete overhaul of the EDI cartridge
    * Rewritten to extend the DFDL cartridge and provide much better support for reading EDI documents.
    * New functionality for serialising EDI documents.
    * As in earlier versions, incorporated special support for EDIFACT.
* Independent release cycles for all cartridges and one [Maven BOM](https://github.com/smooks/smooks-bom/tree/v2.0.0-M1) (bill of materials) to track them all
* License change
    * After reaching consensus among our code contributors, we've dual-licensed Smooks under [LGPL v3.0](https://choosealicense.com/licenses/lgpl-3.0/) and [Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/). 
    This license change keeps Smooks open source while adopting a permissive stance to modifications.
* Numerous dependency updates
* Maven coordinates change
    * We are now publishing artifacts under Maven group IDs prefixed with "org.smooks".

Head over to Smooks’s extensive [catalogue of examples](https://github.com/smooks/smooks-examples/tree/v1.0.0) to get yourself started on 2.0.0-M1. The next milestone release 
will simplify Smooks’s API, restructure its internals in order to make it more extensible, as well as add nifty constructs 
for common use cases so stay tuned.
