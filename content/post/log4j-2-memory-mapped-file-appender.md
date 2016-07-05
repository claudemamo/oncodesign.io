+++
title = "Log4j 2 Memory-Mapped File Appender"
date = 2013-12-23T11:38:00Z
updated = 2013-12-23T14:12:11Z
tags = ["Java", "Log4j 2", "Java NIO"]
blogimport = true 
[author]
	name = "Claude"
	uri = "https://www.blogger.com/profile/06379003436141860057"
+++

During the weekend I dug into Java NIO, specifically, mapping files to memory to reduce I/O time. What's more, since I had a lot of free time on my hands, I developed a&nbsp;<a href="http://logging.apache.org/log4j/2.x/" target="_blank">Log4j 2</a>&nbsp;<a href="https://issues.apache.org/jira/browse/LOG4J2-431" target="_blank">memory-mapped file appender</a>. On my machine, performance tests running on a single thread using the MemoryMappedFile appender show an improvement by a factor of 3 when compared to the&nbsp;<a href="http://logging.apache.org/log4j/2.x/manual/appenders.html#RandomAccessFileAppender" target="_blank">RandomAccessFile</a>&nbsp;appender.
