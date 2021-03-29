+++
title = "Smooks 2 Final Milestone Release"
date = 2021-03-29
tags = ["Smooks", "DFDL", "Performance", "2.0.0-M3", "Pipelines"]
+++

<img src="/images/smooks-logo.png" alt="Smooks logo" style="max-width:70%"/>
<br/>

The [last milestone umbrella release](https://github.com/smooks/smooks/releases/tag/v2.0.0-M3) of Smooks 2 is out. Among the things we’ve focused on in this release is performance. 
In particular, the [DFDL](https://www.ibm.com/support/knowledgecenter/SSMKHH_10.0.0/com.ibm.etools.mft.doc/df20060_.htm) cartridge was upgraded to [Apache Daffodil 3](https://daffodil.apache.org/releases/3.0.0/) in order to leverage Daffodil’s new streaming capabilities. 
This means that the DFDL cartridge, including the specialised EDI and EDIFACT cartridges, are no longer memory-bound which 
allows developers to unleash Smooks’s full potential when processing flat files. Other noteworthy performance improvements are:

* Changing the default SAX parser implementation from Apache Xerces to [FasterXML's Woodstox](https://github.com/FasterXML/woodstox): benchmarks consistently showed Woodstox outperforming Xerces

* Eliminating CPU and memory hot spots

* Replacing synchronised data structures with thread-safe lock-free alternatives

* Forking [Apache Xerces](https://en.wikipedia.org/wiki/Apache_Xerces) and modify it to provide a more optimal DOM implementation

All these optimisations have translated to a massive jump in throughput as demonstrated by our benchmarks. In one benchmark 
consisting of a non-trivial config, Smooks churned through a 1.5 GB [real-world XML dataset](https://datahub.io/collections/bibliographic-data#the-dblp-computer-science-bibliography) in under 20 minutes! Not bad 
considering that the benchmark ran in a desktop environment. This definitely doesn’t mark the end of our performance tuning 
exercise. To keep us on our toes, the latest Smooks snapshot is benchmarked every time code is checked in, and the results [archived for review](https://github.com/smooks/smooks/releases/download/v2.0.0-M3/recording.zip).

Aside from better performance, pipelines have made their grand debut in 2.0.0-M3. As discussed in an [earlier blog post](/2021/02/16/a-sneak-peek-at-smooks-2-pipelines/), 
a pipeline is a flexible, yet simple, Smooks construct that isolates the processing of a targeted event from its main processing 
as well as from the processing of other pipelines. With pipelines, you can enrich data, rename/remove elements or attributes, 
and much more. Read [2.0.0-M3's docs](https://github.com/smooks/smooks.github.io/blob/v2.0/docs.markdown#pipeline) to learn more about pipelines.

Last but not least, Smooks’s Java namespaces were re-organised to provide a cleaner and more intuitive package structure. 
Broadly speaking, Smooks classes now fall under two top-level packages:

* `org.smooks.api`: represents the Java contract between the developer and Smooks. Developers can safely assume that referencing 
  interfaces within this package will not lead to breakage in their applications when upgrading to minor or patch versions of Smooks.

* `org.smooks.engine`: represents Smooks’s internals. Whenever possible, developers should avoid referencing this package's classes 
  since no guarantee is given about their backwards compatibility between Smooks releases.

Milestone 3 concludes the final major changes of Smooks 2. Future releases prior to GA will be release candidates addressing 
any shortcomings or bugs. As always, feedback is more than welcome on [Smooks’s user forum](https://groups.google.com/g/smooks-user).
