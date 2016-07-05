+++
title = "Building ZeroMQ for Android"
date = 2012-08-12T10:16:00Z
updated = 2013-07-19T13:58:02Z
tags = ["ØMQ", "ZeroMQ", "Android", "ZMQ"]
blogimport = true 
[author]
	name = "Claude"
	uri = "https://www.blogger.com/profile/06379003436141860057"
+++

This weekend I ran ZeroMQ on the Android platform. The ZeroMQ website gives <a href="http://www.zeromq.org/build:android" target="_blank">instructions</a> on how to build ZeroMQ for Android and (surprise, surprise) I got errors following them. At least I wasn't alone. Fellow blogger <a href="http://iso3103.blogspot.com/2012/05/java-zeromq-for-android.html" target="_blank">Victor was just as lucky as me</a>. He solved these errors, and what's even better, he created a <a href="https://github.com/vperron/android-jzeromq" target="_blank">set of scripts</a> which correctly build ZeroMQ for Android. Alas, running "<i>make all</i>" on the project gave me the following error:<br /><br /><script src="https://gist.github.com/claudemamo/3330498.js"> </script> The compiler complained because it couldn't find some <a href="http://developer.android.com/tools/sdk/ndk/index.html" target="_blank">Android NDK</a> header files which had their location changed in the latest&nbsp;release of the NDK (at the time of the writing the version is R8b). This was an easy fix. I just passed some flags to the compiler specifying the new header file locations. On my GitHub page, you'll find a <a href="https://github.com/claudemamo/android-jzeromq" target="_blank">fork</a> of Victor's project which applies the fix. Enjoy!
