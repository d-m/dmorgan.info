---
layout: post
title: "Getting Started with AOL Moloch"
description: "Get started with Moloch, an open source PCAP capturing system
maintained by AOL."
modified: 2014-05-17 17:37:53 -0400
category: posts 
tags: [aol, moloch, pcap, open source]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---

[Moloch](https://github.com/aol/moloch) is an open source PCAP capturing,
indexing, and database system maintained by AOL with source hosted on GitHub.

## Overview

With Moloch you can capture full PCAPs of traffic sessions on your network,
search through and filter the resultant session metadata, and export PCAPs based
on session, time period, or both. In addition, Moloch enriches session metadata
by utilizing [MaxMind GeoIP](http://www.maxmind.com/app/c) to gelocate IP
addresses.

## Components

Moloch consists of three main components:

 -  A single-threaded capture process which runs per network interface. One
machine can run multiple processes if monitoring multiple interfaces.
 -  A viewer application built using [node.js](http://nodejs.org) containing the
user interface and PCAP transfer functionality.
 -  [Elasticsearch](http://elasticsearch.org)

These components can be deployed on a single machine or across multiple hosts.
The GitHub repo contains an `easybutton-config.sh` script for quickly getting
started.

## Single Host Install

To get Moloch up and running, perform the following steps on a Linux system:

 1.  Run `git clone https://github.com/aol/moloch.git` to clone the Moloch repo.
 2.  Execute `sudo moloch/easybutton-singlehost.sh` to kick off the
 installation. The script will use `yum` or `apt-get` and `wget` to get its
 dependencies.
 3. Start Elasticsearch: `sudo /data/moloch/bin/run_es.sh`
 4. Start the capture process: `sudo nohup /data/moloch/bin/run_capture.sh &`
 5. Start the viewer process: `sudo nohup /data/moloch/bin/run_viewer.sh &`

The web interface will be avaialable on port 8005 after starting the viewer
process.  If the capture process on your Moloch host is listening on the same
interface that the viewer is bound to then you should begin to see traffic
sessions appear in the "Sessions" tab.

## Troubleshooting

If traffic does not begin to appear in the "Sessions" tab of the web interface
then you can check the logs located at `/data/moloch/logs/`. When I first
started Elasticsearch, the viewer process, and the capture process my
`capture.log` contained the following:

~~~
May  1 23:14:07 http.c:237 moloch_http_connect(): Connecting 0x7f29208a6010 localhost:9200
May  1 23:14:07 http.c:277 moloch_http_connect(): 0x7f29208a6010: Error: Address family not supported by protocol
May  1 23:14:07 http.c:553 moloch_http_create(): Couldn't connect to 'localhost:9200'
~~~

Elasticsearch uses port 9200, so this log message indicates that the capture
process could not connect to the Elasticsearch instance.  To fix this, I edited
the Elasticsearch config line in `/data/moloch/etc/config.ini` from
`elasticsearch=localhost:9200` to `elasticsearch=127.0.0.1:9200` and started
the capture process again.
