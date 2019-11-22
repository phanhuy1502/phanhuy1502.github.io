---
layout: post
title: Stream processing with Flink (Part 1)
subtitle: Part 1 explains what is stream processing with an example built from scratch
tags: [code, flink, tutorial]
comments: true
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
---

This post is a part of a series

- [Part 1]({% post_url 2019-05-10-stream-processing-flink %})
- [Part 2]({% post_url 2019-06-21-stream-processing-flink-2 %})

The first part of the series introduces the concepts of data stream, data stream processing, data pipeline, while building a minimal example data pipeline.

## What is a stream?

Think of data stream as continuous, immutable flow of sequential data record.
Those data stream are usually continuous, sequential and immutable.

Your Facebook feed is a data stream, with posts coming continunously.
The posts are sequential, meaning they have a time order and new posts are appended to the end of the feed (top of your screen).
The posts are also immutable in a sense that post won't be updated once they are in the stream.
(Strictly speaking, Facebook post can be edited, and can be moved up/down in the feed, but let's assume that they aren't so we can have a simple example of data stream.)

Another example of stream is your application log.
Think of each log line as a record in the stream.
The log is flow of lines comming continuously in sequential order, each line is immutable
(log won't be changed once they have been written to the file system).

## Create a (trivial) stream

Let's create a stream which writes a random number (from 1 to 5) together with a timestamp to the end of a file (`ids.txt`) every 500 miliseconds.
Create a new file named `stream.sh` with the following content:

> Scripts are tested in MacOS Mojave 10.14.4, it should also work on a Linux distribution.
> Send me a message if the script fails on your machine.

```bash
#!/bin/bash
while true
do
    echo $(date +"%s") " " $((1 + RANDOM % 5)) >> ids.txt
    sleep 0.5
done
```

Run the script on a terminal:

```sh
sh stream.sh
```

The content of the file `ids.txt` now is a stream.
And we can view the new records (in this case, a line with a single number) on another terminal:

```sh
tail -f ids.txt
```

You should observe a new number printing out every 0.5 second, like

```txt
1557394952   1
1557394952   4
1557394953   3
1557394953   2
...
```

This is a trivial example of a data stream. The random number can be replaced with a log line, a content of a Facebook post or a changes in database. The storage here is a file system, which can be replaced with a message queue like RabbitMQ or Kafka. The data volume can be a lot larger than just 2 messages/second. Here, the lines/messages come in time order, in other cases, messages might come out-of-order (a line with a later timestamp comes before a line in a previous timestamp).

## Process a stream

Image we have a website selling clothes, and the stream in our previous example are the IDs of the items which user clicks on (our shop has 5 items, so the item ID ranges from 1 to 5). We want to convert the stream of item IDs to a stream of item names, and write it to another stream. The conversion from item IDs to item names is done by a process unit:

```txt
 _____________     __________     ______________     __________
|  stream.sh  |-> |  ids.txt |-> |  process.sh  |-> |names.txt |
|(data source)|   |(stream 1)|   |(process unit)|   |(stream 2)|
 -------------     ----------     --------------     ----------
```

The content of names.txt should look like

```txt
1557394952   jeans
1557394952   t-shirt
1557394953   shorts
1557394953   skirt
...
```

Let's create our processing unit, create a file named `process.sh` with the content:

```bash
#!/bin/bash
while read line
do
    # parse message
    timestamp=$(echo $line | cut -d' ' -f1)
    id=$(echo $line | cut -d' ' -f2)

    # map id to name
    case $id in
        "1") name="jeans"
        ;;
        "2") name="skirt"
        ;;
        "3") name="short"
        ;;
        "4") name="t-shirt"
        ;;
        "5") name="hoodie"
        ;;
    esac

    # output
    echo $timestamp " " $name >> names.txt
done
```

The script reads from the stdin, maps the IDs to names and appends the processed message to another stream (in this case, appending to the file `names.txt`).

While running the `stream.sh`, run the `process.sh` on another terminal window:

```sh
tail -f ids.txt | sh process.sh
```

And we can see the processed stream by:

```sh
tail -f names.sh
```

The output should look like:

```
1557394952   jeans
1557394952   t-shirt
1557394953   shorts
1557394953   skirt
...
```

## Conclusion

Congrats, you've just built a simple data pipeline to process a data stream.
In the example above, we construct a simple stream with a processing unit to convert it into another stream.
The processing unit does a simple 1-to-1 mapping.
In reality, there are many concerns while dealing with stream processing, such as:

- More complicated processing requirements such as aggregation. For instance, counting number of clicks by item ID in every minute in our example. This requires more complicated processing unit in which we need to keep a `state` (a count of each item ID) and a time window checking (to know when is the start and end of a minute)

- Huge data volumes, which can't be processed on a single machine. This will requires us to design a distributed system. Distributed system, in turn, will introduce the problem of out-of-order and duplicated messages.

- Storage system: how to we effectively store the streams?

The next post will cover an example with `Kafka` and `Flink`, which aims to address those issues.
