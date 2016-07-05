+++
title = "Revisiting Dynamic Ports in Mule 3"
date = 2012-05-28T20:43:00Z
updated = 2013-02-05T19:51:31Z
tags = ["Mule 3", "Dynamic Port", "JUnit 4"]
blogimport = true 
[author]
	name = "Claude"
	uri = "https://www.blogger.com/profile/06379003436141860057"
+++

Daniel Zapata wrote an interesting&nbsp;<a href="http://blogs.mulesoft.org/dynamic-port-testing-in-mule-3" target="_blank">post</a> about using dynamic ports when testing your Mule 3 application. Since then, subsequent releases of Mule included support for JUnit 4 which meant improved flexibility in terms of dynamic ports.<br /><br />Before JUnit 4, an annoying problem with dynamic ports was that you were limited to property placeholder names having the following pattern: 'port' +&nbsp;<i>n</i> where n is an integer. For example:<br /><br /><script src="https://gist.github.com/claudemamo/2820059.js?file=mule-config.xml"></script>Using JUnit 4, this problem is solved by leveraging the <i>Rule</i> annotation and the <i>DynamicPort</i> class. We'll see how this is done. Let's create a simple Mule config for testing dynamic ports out:<br /><br /><script src="https://gist.github.com/claudemamo/2820059.js?file=dynamic-port-junit4-functional-test-config.xml"></script>Notice the property placeholder "${foo}" in the <i>port</i> attribute. The next step is to create a test case for the config. The test case <b>must</b> extend <i>org.mule.tck.junit4.FunctionalTestCase</i> for this to work:<br /><br /><script src="https://gist.github.com/claudemamo/2820059.js?file=DynamicPortJunit4TestCase.java"></script>The <i>port</i> variable declaration is what we're interested in. <i>@Rule</i> instructs JUnit to execute code in the&nbsp;<i>DynamicPort</i>&nbsp;object before running the test. The code will:<br /><ol><li>Find an unused port,</li><li>Search the Mule config for a placeholder with the name 'foo', and then</li><li>Replace the placeholder with the unused port number.</li></ol><div>The newly assigned port no. is retrieved using the <i>getNumber()</i> method of the <i>DynamicPort</i> class. The complete example can be found on <a href="https://github.com/claudemamo/dynamic-port-junit4" target="_blank">GitHub</a>.</div>
