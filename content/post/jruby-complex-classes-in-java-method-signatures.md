+++
title = "JRuby Complex Classes in Java Method Signatures"
date = 2013-08-19T11:41:00Z
updated = 2013-12-27T08:57:10Z
tags = ["Java", "JRuby"]
blogimport = true 
[author]
	name = "Claude"
	uri = "https://www.blogger.com/profile/06379003436141860057"
+++

As documented in the JRuby wiki,&nbsp;<i><a href="https://github.com/jruby/jruby/wiki/GeneratingJavaClasses#generating-java-classes-ahead-of-time" target="_blank">java_signature</a></i>&nbsp;changes a method's signature to match the signature string passed to it:<br /><br /><script src="https://gist.github.com/claudemamo/6262775.js?file=example(1).rb"></script>Observe that the classes in the method signature&nbsp;are primitive. What if we use a complex class as a parameter type?<br /><br /><script src="https://gist.github.com/claudemamo/6262775.js?file=example(2).rb"></script>Running the above code will give you the following <i>NoMethodError</i> message:<br /><br /><script src="https://gist.github.com/claudemamo/6262775.js?file=error"></script> The way I went about using complex classes in signatures is to utilise&nbsp;<i>add_method_signature</i>&nbsp;instead of&nbsp;<i>java_signature</i>:<br /><br /><script src="https://gist.github.com/claudemamo/6262775.js?file=example(3).rb"></script><i>add_method_signature</i>&nbsp;expects the first argument to be the name of the method that will have its signature changed. For the second argument, it expects it to be a list of classes. The list's first item is the return class (e.g.,&nbsp;<i>void</i>) while the subsequent items are the signature's parameter classes (e.g.,&nbsp;<i>int</i> and <i>MyClass</i>). Note that I invoke&nbsp;<i>become_java!</i>&nbsp;on the complex class. This tells <i>MyClass</i> to materialize itself into a Java class. The <i>false</i> flag is needed so that JRuby's main class loader is used to load the class. Without it, you'll be greeted by a&nbsp;<i>java.lang.ClassNotFoundException</i>.<br /><br />
