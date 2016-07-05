+++
title = "Safely Prevent Template Caching in AngularJS"
date = 2014-02-19T15:21:00Z
updated = 2014-02-23T06:55:14Z
tags = ["AngularJS"]
blogimport = true 
[author]
	name = "Claude"
	uri = "https://www.blogger.com/profile/06379003436141860057"
+++

AngularJS's <i><a href="http://docs.angularjs.org/api/ng/service/$templateCache" target="_blank">$templateCache</a></i> can be a pain in the ass. Sometimes we don't want templates to be cached. A quick Internet search to disable caching gives the following workaround:<br /><br /><script src="https://gist.github.com/claudemamo/9092047.js?file=app(1).js"></script> But as I have learnt with the <a href="http://angular-ui.github.io/bootstrap/" target="_blank">UI Bootsrap module</a>, this may cause AngularJS modules that use <i>$templateCache</i> to break. A solution is to tweak the above workaround so that new cache entries are removed on route change instead of indiscriminately removing all entries:<br /><br /><script src="https://gist.github.com/claudemamo/9092047.js?file=app(2).js"></script>
