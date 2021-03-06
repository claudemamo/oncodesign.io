+++
title = "Posting & Processing JSON with jQuery & Spring MVC"
date = 2013-03-02T12:37:00Z
updated = 2013-08-19T19:39:52Z
tags = ["JQuery", "JSON", "Spring MVC"]
blogimport = true
[author]
	name = "Claude"
	uri = "https://www.blogger.com/profile/06379003436141860057"
+++

Consider an HTML form containing numerous input fields. When the user clicks on the form's submit button, the fields need to be sent as JSON to a service that under the hood is implemented in Spring MVC. A jQuery function transforms and posts the form's inputs:<br />
<script src="https://gist.github.com/claudemamo/5066797.js?file=NewClientForm.jsp"></script>
Through the <i>$('form').submit(...)</i> function (line 61), jQuery intercepts any click on the submit button and posts the form with the <i>$.ajax(...)</i> function (line 63). Before sending the data, jQuery transforms the form's inputs to JSON using&nbsp;<i>JSON.stringify($(this).serializeArray())</i> (line 66). This function outputs a JSON string consisting of a list of key-value pairs:<br />
<script src="https://gist.github.com/claudemamo/5066797.js?file=output.json"></script>
On the service side, I have the controller to process the form:<br /><br /><script src="https://gist.github.com/claudemamo/5066797.js?file=ApplicationController.java"></script><i>getCreateClientForm()</i> returns the HTML form to the user. The more interesting method is&nbsp;<i>processSubmitCreateClient(...)</i>: the <i>headers</i> annotation attribute tells Spring that&nbsp;<i>processSubmitCreateClient(...)</i>&nbsp;should be invoked only if the HTTP request header <i>Content-Type</i> is set to <i>application/json</i>. Furthermore,&nbsp;<i>@RequestBody</i> tells Spring to bind the JSON data to the <i>client</i> paramater which is a list of maps.&nbsp;<i>processSubmitCreateClient(...)</i>&nbsp;iterates through each element to merge the individuals maps into a single map to facilitate processing.<br /><br />I &nbsp;added the <a href="https://github.com/FasterXML/jackson" target="_blank">Jackson</a> library to the project's POM since Spring requires a JSON deserializer to perform the data binding:<br /><br /><script src="https://gist.github.com/claudemamo/5066797.js?file=pom.xml"></script>You can <a href="https://github.com/claudemamo/jq-springmvc-json" target="_blank">grab the complete example</a> from GitHub and run it from the project root directory using the following command:<br /><br /><script src="https://gist.github.com/claudemamo/5066797.js?file=run.sh"></script>From your browser, enter "<a href="http://localhost:8080/jq-springmvc-json/create-client.html">http://localhost:8080/jq-springmvc-json/create-client.html</a>" to access the form.
