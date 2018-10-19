+++
date = "2018-10-17T10:00:00+01:00"
title = "The Many Facets of EDI Integration"
tags = ["EDI", "B2B"]
+++

*This post was reproduced on [PortX's blog](https://portx.io/the-many-facets-of-edi-integration/)*.

The other day I read a simple yet thought-provoking thread on a message board. A company was about to embark on its first EDI integration journey but was pretty much clueless about EDI translators and what questions to ask to EDI providers. Though this might come as a surprise to the individual asking for guidance, the EDI translator, that is the program transforming the EDI file into a structure suitable for processing and vice-versa, however important, is only a small piece of the integration puzzle. EDI lives alongside a number of application layer protocols, tools, and processes which, together, make up the ecosystem of an EDI integration:

* A process documenting the trading agreement, known as “onboarding”, has to be implemented and followed for parties wanting to trade with you;

* EDI files must be transmitted over secure application protocols and the corresponding B2B transactions must be visible to the business;

* Enterprises need to transform and map EDI documents into formats their internal systems understand;

* Documents or parts of the document need to be routed to the correct systems.

<br/>
<img src="/images/the-many-facets-of-edi-integration.jpg" alt="Typical EDI integration solution"/>

Having given an idea of the complete picture, it’s understandable to assume that simply translating documents to and from EDI will likely be a far cry from making one’s first EDI integration a success.

EDI integration puzzle pieces can be acquired as a unified solution from a single vendor, purchased separately from different vendors, developed in-house, or outsourced. Whichever way it’s done, learning the many facets of an EDI integration, or rather, B2B integration, is a prerequisite to successfully architecting a solution. With this in mind, in the following sections, I briefly describe the main facets hoping to equip integrator newbies with the initial vocabulary, concepts, and pointers needed for solving the typical challenges encountered in such integrations.

### Trading Partner Onboarding

Partner onboarding is the first step to establishing an automated trading channel with your business partner. The partner’s identifiers (including aliases and the partner’s legal name) such as its [DUNS](https://www.dnb.com/duns-number.html) number, the type of documents to expect and over which transports they will be delivered, host addresses, passwords, TLS certificates, and FTP mailbox names are only a tiny sample of what you would want to record. Manually onboarding partners through the use of ad-hoc solutions like spreadsheets doesn’t scale beyond a few partners.

A fully-fledged Trading Partner Management solution manages this information in a safe and secure place for your perusal while also ensuring that B2B exchanges are actually respecting the parameters set by you: if you’re anticipating an EDIFACT Purchase Order message yet instead receive a Dispatch Advice, then the solution should reject the transaction while reporting the problem to the relevant users. A number of Trading Partner Management products also provide a portal to collaborate with partners and give them the ability to participate in the on-boarding process such that they can enter the parameters they have control over, say certificates and contact information.
<br/>
### Application Protocols

Your organisation and your trading partner’s requirements dictate the protocol over which documents are sent or received. For example, if the business requires non-repudiation of receipt, [AS2 over HTTPS](http://oncodesign.io/2015/05/15/a-primer-to-as2/) is often recommended for achieving this. Otherwise, you can use a simpler, less costly protocol like FTPS. Sometimes it’s not even up to you to decide on the protocol. Big players will mandate the protocol to use and you’ll have little option but to adhere to their rules in order to do business with them. In any case, make sure that the B2B vendor supports transports for the most frequently used protocols (i.e., AS2, SFTP, FTPS, and HTTPS) and easily allows you to add your own transports should the need arise. A solution dedicated to secure file transfers, with all the bells and whistles, is aptly named Managed File Transfer (MFT).

Consider where you will host the endpoint sending/receiving the document. Does the vendor allow you to stand-up an endpoint in the cloud only through configuration? Your management might not necessarily agree with a cloud strategy depending on their budget and the nature of the data to be transported, therefore ensure that the vendor has a solution for hosting the endpoints on-premise. This might come in the form of a private cloud, or it could be more rudimentary like libraries provided by the vendor which you have to install, configure, and invoke.
<br/>
### Transaction Visibility

Business divisions require micro level and macro level visibility into the transactions carried out with your trading partners. At a minimum, a summary ought to be kept for every transaction. The summary should contain basic information like the transaction’s initiator and recipient, a transaction identifier, direction (i.e., whether the document is inbound or outbound), attributes common to all transmitted files (e.g., format,  filename, etc...), the protocol over which the file was delivered, the overall status (i.e., whether the transaction was successful, failed, or is still pending), and the date/time the transaction took place.

Summaries will only go so far. Technical staff will likely need to inspect the document’s raw or transformed content to troubleshoot system errors. On the other hand, management will want a high-level view of documents such that they can easily read purchase order numbers, amounts, inventories, and so on. Consequently, it’s important to have documents visible at various levels of abstraction.

From a micro level point of view, solutions are expected to answer questions like “What happened to that purchase order?” or ”Did they use the old or new item number for that item back in January?”. Not only that, they should be able to correlate transactions so that they can answers queries such as “Did we send a PO Acknowledgement or an ASN back?” or “Need all ship notices relating to that order”. From a macro level standpoint, solutions should satisfy statements like “I want to view all failed transactions happening on a particular day grouped by error type” or “Show me the number of purchase orders received by partner”.

SaaS vendors probably don’t keep transactions ad infinitum so, if you’re talking to one, be sure to ask for the retention policy and what mechanisms it has for exporting the transaction history. Equally important, your organisation might not trust the vendor with sensitive data like credit card numbers. The vendor should offer ways to exclude such data from their storage and allow you to plug in your own storage provider.

Excluding sensitive data doesn’t absolve you from knowing where the vendor will save transactions and how will they be saved. The locality of data matters within many jurisdictions: verify that the vendor can host the transactions in geographic areas meeting your legal requirements.

How transactions are stored is a broad topic. Are your transactions sufficiently isolated from other customers’ transactions? Does the vendor offer customers the option to have their own dedicated data store? Are transactions encrypted? Is the storage provider HIPAA compliant? These are just some of the questions that can be raised during discussions with the vendor, but of course, which questions to bring up will largely depend on your domain.

Related to transaction visibility is the ability to retry transactions in case of errors, which is a common business requirement. The error may occur occur for reasons beyond your control, however, hitting the retry button should trigger the sequence of events that leads to the re-processing of the document. Ideally a facility would be available for editing the document before retrying the transaction should the fault lie in the document itself. Otherwise, repeatedly retrying a transaction will result the same error over and over again.
<br/>
### Transforming, Mapping & Routing

The life of an EDI document doesn’t end once it’s received over a transport and translated: it’s actually the beginning. Different IT systems in your organisation will be interested in different parts of the document. The accounting system might be interested in the document’s payment section, the order management system might be interested in the shipping details, items, and quantities ordered, while the ERP application could be interested in all these parts. Tools should be bought or implemented for slicing, dicing, and routing translated documents according to your organisation’s needs. Moreover, the internal systems might not necessarily know how to process the document in its parsed representation. Solving this problem means that transformers and mappers should be leveraged for converting documents from one format to another and mapping to the target structures expected by the back-end systems. Adding to the topic of transformation and mapping, the output produced from these stages should be visible to teams performing troubleshooting, generating reports, etc.

It’s no coincidence one often finds a class of software, known as Enterprise Service Bus (ESB), in B2B integrations. After all, the previously mentioned tools are supposed to be an ESB’s bread and butter. Generally speaking, ESBs have in-built features for batching, debatching, content-based routing, UI-based or script-based mapping, and transforming a dozen or so formats. They also come with a myriad of connectors for integrating with backend systems like ERPs and CRMs.
<br/>
### EDI Translator

EDI, in general, is difficult to translate (RosettaNet might be an exception due to the pervasiveness of XML readers/writers though it’s by no means trivial to follow the standard). Multiple levels of nesting, ever changing implementation guidelines, repeatable groups (i.e., loops), dynamic delimiters and terminators, variable length data fields, relational data field usage: this is only a glimpse of the things a properly built translator must handle.

Open source translators do exist for some of the standards but they’re far and between. Apart from their complexity, some standard committees, like ANSI X12, put a hefty price tag on their specs which makes building an open source translator even more challenging. It’s a given then that you shouldn’t try to develop your own translator unless you definitely know what you’re doing. If the B2B vendor doesn’t have its own EDI translator, then it’s less likely that an off-the-shelf translator will integrate well with the vendor’s B2B solution.

Besides standard features like schema validation, ensure that the chosen translator can handle files in the sizes you expect to receive or send without requiring unreasonable amounts of main memory. Files ranging in the hundreds of megabytes (even gigabytes) is more common than you think in the B2B world and the last thing you want to find out is that the shiny expensive parser you purchased doesn’t support streaming.

In addition to streaming, the parser should allow you to add your own schemas given how a many organisations amend the EDI schemas to suit their needs. Let’s not forget those pesky sequence numbers. Any self-respecting EDI translator ought to detect duplicate sequence numbers, report them, and allow you to customise the sequence according to the business’s needs. If the EDI translator is to be replicated across a cluster, coordination might be required between the instances in order to avoid sequence number collisions. A “cluster-aware” translator will be a bonus to the development team.

Support for automatic acknowledgments is another item in the feature checklist. Acknowledgements play a crucial role in EDI. It’s the way for parties in an exchange to determine whether the document was successfully processed (see transaction visibility), and if not, what the problem was (e.g., missing field). Your job will be much easier when you have a translator automatically generating the right acknowledgements on your behalf (e.g., TA1 ACK for an X12 interchange header and 997 ACK for every functional group).<br/><br/>


No doubt that the above sections merely touch the surface of B2B integrations. Feel free to add your own thoughts and let me know what I’ve missed in the comments section below.
