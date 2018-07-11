---
layout: post
title: "Using Apache Flume-NG to move binaries around"
date: 2014-03-26 16:32:19 -0400
comments: true
categories: [linux, hadoop, flume-ng]
---

Need to move small binary files from one server to the other? Flume-NG to the
rescue! It supports complex topologies and is quite fast about it.

1. Install flume-ng on your server (Cloudera's CDH4 packages are nice).

2. Set up a source and a destination directories and a configuration
   file, in our case flume-ng-agent.conf.

```
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = spooldir
a1.sources.r1.spoolDir = ~/source
a1.sources.r1.fileHeader = true
a1.sources.r1.basenameHeader = true
a1.sources.r1.batchSize = 1
a1.sources.r1.deletePolicy = immediate
a1.sources.r1.deserializer = org.apache.flume.sink.solr.morphline.BlobDeserializer$Builder

# Describe the sink
a1.sinks.k1.type = file_roll
a1.sinks.k1.batchSize = 1
a1.sinks.k1.sink.directory = ~/dest
a1.sinks.k1.rollInterval = 0

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

3. Launch a flume agent

```
flume-ng agent --conf-file flume-ng-agent.conf --name a1 -Dflume.root.logger=INFO,console
```

4. Drop some binaries into the ~/source folder and watch them appear in
   the ~/dest folder. Originals are by default renamed with a .COMPLETE
suffix but could also be removed.

The only downside is the the default serializer for file_roll sink does
not preserve the original filename.

Always a good idea to look at the docs for more info -
[http://archive-primary.cloudera.com/cdh4/cdh/4/flume-ng/](http://archive-primary.cloudera.com/cdh4/cdh/4/flume-ng/).
