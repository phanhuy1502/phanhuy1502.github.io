---
layout: post
title: Stream processing with Flink
subtitle: A hand-on tutorial with stream processing on Flink
tags: [code, flink, tutorial]
comments: true
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
---

## What is a stream?
Think of data stream as continuous, immutable flow of sequential data record. Three properties of data stream from my observation are (1) continuous, (2) sequential and (3) immutable. 

Your Facebook feed is a data stream, with posts coming continunously. The posts are sequential, meaning they have a time order and new posts are appended to the end of the feed (top of your screen). The posts are also immutable in a sense that post won't be updated once they are in the stream. (Strictly speaking, Facebook post can be edited, and can be moved up/down in the feed, but let's assume that they aren't so we can have a simple example of data stream.)

Another example of stream is your application log. Think of each log line as a record in the stream. So we have continuous flow of lines comming continuously in sequential order, each line is immutable (log won't be changed once they have been written to the file system).

## Creating a (trivial) stream
Let's create a stream which writes a random number to the end of a file every 500 miliseconds. Create a new file named `stream.sh` with the content:
```bash
#!/bin/bash
while true 
do
	echo $((1 + RANDOM % 10))
	sleep 0.5
done
```
Run the script:
> sh stream.sh






