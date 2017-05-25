+++
title = "Scaling up Mule with Async Request Handling/Continuations"
date = 2014-01-07T14:01:00Z
updated = 2014-09-17T10:52:41Z
tags = ["Mule 3", "Mule"]
blogimport = true
aliases = [
  "/2014/01/07/scaling-up-mule-with-async-request-handling/continuations/"
]
[author]
	name = "Claude"
	uri = "https://www.blogger.com/profile/06379003436141860057"
+++

Non-blocking I/O servers such as Node.js are appealing because, when compared to blocking I/O servers, they utilise less threads to perform the same tasks under the same load. Less threads mean more efficient use of resources (e.g., smaller memory footprint) and better performance (e.g., reduced no. of context switches between threads). Let's take a stab at having non-blocking I/O behaviour in Mule. Consider the following Mule 3.4 application calling an HTTP service:<br /><br /><script src="https://gist.github.com/claudemamo/8289429.js?file=mule-config(1).xml"></script>Wrapping the&nbsp;<i>async</i>&nbsp;processor around&nbsp;<i>http:outbound-endpoint</i>&nbsp;prevents the receiver thread from blocking on the outgoing HTTP call. But this <b>kind</b> of asynchronous behaviour causes the service's reply to be ignored: certainly not what we want for the common case. Moreover, the&nbsp;<i>async</i>&nbsp;processor borrows a thread from some thread pool to carry out the blocking HTTP call, preventing the borrowed thread from doing any useful work while being blocked.<br /><br />The aforementioned problems can generally be solved by replacing the blocking I/O library with a non-blocking version and&nbsp;<b><a href="http://wiki.eclipse.org/Jetty/Feature/Continuations" target="_blank">Asynchronous Request Handling</a></b>&nbsp;(a.k.a continuations). Async request handling is a threading model where a thread serving a client request can be suspended and returned to its respective thread pool; free to serve other client requests. Typically the thread would be suspended after sending out a request to a remote service or kicking off a long-running computation. Although the suspended thread has forgotten about the client, the server has not. It knows the client is still waiting for a reply. For this reason, a thread can pick up where the suspended tread has left off and deliver the reply back to the client. Normally this would happen in the context of a callback.<br /><br />Awesome! Let's implement this in every place where blocking I/O is present. Not so fast. First, a library supporting a non-blocking alternative to what you already have in your solution must be available. Second, to my knowledge, the only Mule transport that provides async request handling&nbsp;is <a href="http://www.mulesoft.org/documentation/display/current/Jetty+Transport+Reference" target="_blank">Jetty</a>. So for this to work, the Jetty inbound-endpoint&nbsp;processor must be used as the message source:<br /><br /><script src="https://gist.github.com/claudemamo/8289429.js?file=mule-config(2).xml"></script>Furthermore, as shown above,&nbsp;async&nbsp;request handling&nbsp;must be turned on by setting <i>useContinuations</i> to true on the&nbsp;Jetty connector.<br /><br /><div style="text-align: justify;">Calling an HTTP service is a fine example where we can put async&nbsp;request handling to good use. The initial step is to find an HTTP client library implementing a non-blocking API [1]. I'll opt for <a href="http://hc.apache.org/httpcomponents-asyncclient-4.0.x/" target="_blank">Apache&nbsp;HttpAsyncClient</a>.</div><br /><div style="text-align: justify;">The next step is to develop a message processor that (1) uses HttpAsyncClient to call a service, (2) registers a callback to resume processing of the client request on receiving the HTTP service reply, and (3) immediately returns the thread to its thread pool upon sending asynchronously the HTTP request. Such a processor will require special abilities so I'll extend my processor from&nbsp;<i><a href="http://www.mulesoft.org/docs/site/current3/apidocs/org/mule/processor/AbstractInterceptingMessageProcessor.html" target="_blank">AbstractInterceptingMessageProcessor</a></i>:</div><br /><script src="https://gist.github.com/claudemamo/8289429.js?file=AhcProcessor(1).java"></script>By inheriting from <i>AbstractInterceptingMessageProcessor</i>, I can invoke the next processor in the flow from my callback. Speaking of callbacks, here is the snippet concerning the HTTP client:<br /><br /><script src="https://gist.github.com/claudemamo/8289429.js?file=AhcProcessor(2).java"></script>Lines 10-13 initialise the HTTP client and set the server address to wherever we're going to send the request to. Line 15 sends asynchronously the request, and registers the callback that will handle the reply. Other than the usual stuff of reading from the response stream (lines 19-22), observe that on line 23 the subsequent flow processor in invoked on a&nbsp;<b>different</b>&nbsp;thread. Line 24 tells Jetty that the flow's output message is to be used as the reply for the end-user.<br /><br />One additional item in the list is left: freeing the thread after invoking asynchronously the HTTP client's <i>execute(...)</i> method. Returning <i>null</i> from the <i>process(...)</i> method will do the job (line 40):<br /><br /><script src="https://gist.github.com/claudemamo/8289429.js?file=AhcProcessor(3).java"></script>Finally, we can hook up everything together:<br /><br /><script src="https://gist.github.com/claudemamo/8289429.js?file=mule-config(3).xml"></script>The <a href="https://github.com/claudemamo/asynchronous-request-handling" target="_blank">complete example</a> is found on GitHub.<br /><br />Hopefully async request handling will someday be <a href="https://www.mulesoft.org/jira/browse/MULE-7214" target="_blank">part of Mule's&nbsp;default behaviour</a>. Imagine how useful it would be to call almost any service (e.g., HTTP, JMS, VM) synchronously knowing fully well that behind the scenes Mule is taking care of making every remote call non-blocking.<br /><br /><span style="font-family: Times, Times New Roman, serif; font-size: small;"><span class="num">1:&nbsp;</span><span style="text-align: justify;">A client library implementation should be based on the <a href="http://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf" target="_blank">Reactor pattern</a> otherwise we would be going back to the original problem of many blocking threads.</span></span>