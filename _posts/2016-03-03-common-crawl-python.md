---
layout: post
title: "Exploring the Common Crawl with Python"
excerpt: "The Common Crawl is an organization that crawls the web. Here's how to explore their archives with Python."
modified: 2016-03-03 11:20:24 -0500
category: posts
tags: [python, common crawl, warc, requests, wat, wet, s3, datasets, stringio, contextlib]
image:
  feature: common_crawl_header.jpg
  thumb: common_crawl_thumb.jpg
  credit:
  creditlink:
comments:
share:
---

[Common Crawl](http://commoncrawl.org) is a nonprofit organization that crawls the web and provides the contents to the public free of charge and under [few restrictions](http://commoncrawl.org/terms-of-use/). The organization began crawling the web in 2008 and its corpus consists of billions of web pages crawled several times a year. The data is hosted on Amazon S3 as part of the [Amazon Public Datasets](http://aws.amazon.com/public-data-sets/) program, making it easy and affordable to scan and analyze large portions of the internet.

This post will give a quick introduction to Common Crawl and some ways to explore a crawl archive with Python.

## Navigating a Crawl Archive

Common Crawl performs several crawls a year. All crawls since 2013 have been stored in the [Web ARChive (WARC)](https://en.wikipedia.org/wiki/Web_ARChive) format. This format is a general archive format that can be used both for storing the raw content of a web crawl and associated metadata. One of Common Crawl's archives contains three types of files, all in the WARC format:

 - WARC files containing the raw crawl data ([example](https://gist.github.com/Smerity/e750f0ef0ab9aa366558#file-bbc-warc))
 - WAT files containing metadata for the WARC records in a corresponding WARC file ([example](https://gist.github.com/Smerity/e750f0ef0ab9aa366558#file-bbc-pretty-wat))
 - WET files containing plaintext contained in the WARC records in a corresponding WARC file ([example](https://gist.github.com/Smerity/e750f0ef0ab9aa366558#file-bbc-wet))

The data from each crawl archive is divided into several segments with each segment getting its own directory. Within each segment are additional 'warc', 'wat', and 'wet' directories containing the WARC, WAT, and WET files, respectively.

Each archive comes with four files that aid in exploring the dataset. These files list all segments, all WARC files, all WAT files, and all WET files. We'll use the [February 2016](http://blog.commoncrawl.org/2016/02/february-2016-crawl-archive-now-available/) crawl archive for the following examples. The output of the first few lines of each file is below:

[segments.paths file](https://aws-publicdatasets.s3.amazonaws.com/common-crawl/crawl-data/CC-MAIN-2016-07/segment.paths.gz) (all segements):
{% highlight text %}
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145578.23/
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145751.1/
...
{% endhighlight %}

[warc.paths file](https://aws-publicdatasets.s3.amazonaws.com/common-crawl/crawl-data/CC-MAIN-2016-07/warc.paths.gz) (all raw data files):
{% highlight text %}
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/warc/CC-MAIN-20160205193905-00000-ip-10-236-182-209.ec2.internal.warc.gz
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/warc/CC-MAIN-20160205193905-00001-ip-10-236-182-209.ec2.internal.warc.gz
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/warc/CC-MAIN-20160205193905-00002-ip-10-236-182-209.ec2.internal.warc.gz
...
{% endhighlight %}

[wat.paths file](https://aws-publicdatasets.s3.amazonaws.com/common-crawl/crawl-data/CC-MAIN-2016-07/warc.paths.gz)
{% highlight text %}
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/wat/CC-MAIN-20160205193905-00000-ip-10-236-182-209.ec2.internal.warc.wat.gz
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/wat/CC-MAIN-20160205193905-00001-ip-10-236-182-209.ec2.internal.warc.wat.gz
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/wat/CC-MAIN-20160205193905-00002-ip-10-236-182-209.ec2.internal.warc.wat.gz
...
{% endhighlight %}

[wet.paths file](https://aws-publicdatasets.s3.amazonaws.com/common-crawl/crawl-data/CC-MAIN-2016-07/wet.paths.gz)
{% highlight text %}
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/wet/CC-MAIN-20160205193905-00000-ip-10-236-182-209.ec2.internal.warc.wet.gz
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/wet/CC-MAIN-20160205193905-00001-ip-10-236-182-209.ec2.internal.warc.wet.gz
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/wet/CC-MAIN-20160205193905-00002-ip-10-236-182-209.ec2.internal.warc.wet.gz
...
{% endhighlight %}

The full contents of each listed WARC, WAT, or WET file can be downloaded by prepending s3://aws-publicdatasets/ or https://aws-publicdatasets.s3.amazonaws.com/ to the line.

## The relationship between WARC, WAT, and WET files

The paths for WARC, WAT, and WET files adhere to a pattern. The path

{% highlight text %}
common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/warc/CC-MAIN-20160205193905-00000-ip-10-236-182-209.ec2.internal.warc.gz
{% endhighlight %}

can be deconstructed like so:

<table>
    <tr>
        <td>1454701145519.33</td>
        <td>warc</td>
        <td>20160205193905</td>
        <td>00000</td>
        <td>10-236-182-209.ec2.internal</td>
        <td>.warc.gz</td>
    </tr>
    <tr>
        <td>segment timestamp in microseconds since epoch</td>
        <td>Common Crawl file type</td>
        <td>segment timestamp in YYYYmmddHHMMSS format</td>
        <td>zero-indexed file index in segment</td>
        <td>crawler hostname</td>
        <td>file extension</td>
    </tr>
</table>

These files are also related; each WARC, WAT, and WET file in the same segment with the same index describe the same capture events.

We can investigate this relationship by using the [`warc`](http://warc.readthedocs.org/en/latest/) Python module. The `warc` module simplifies working with WARC files and can be installed with `pip` (you'll also want to make sure you have the [`requests`](http://docs.python-requests.org/en/latest/) module for the examples):

{% highlight bash %}
pip install warc requests
{% endhighlight %}

Note that as of this writing the `warc` module does not work with Python 3.

The `warc` module has an `warc.open` helper function that can be used to load a locally-downloaded, gzipped WARC-formatted file. However, since these files can be several gigabytes, we will try to be a little bit more clever and only download the first 10KB of the file using the `requests` and [`StringIO`](https://docs.python.org/2/library/stringio.html) modules.

Lets download the first 10KB of the first WARC, WAT, and WET files in the 2016 Common Crawl archive:

{% highlight python %}
import warc
import requests
from contextlib import closing
from StringIO import StringIO

def get_partial_warc_file(url, num_bytes=1024 * 10):
    """
    Download the first part of a WARC file and return a warc.WARCFile instance.

    url: the url of a gzipped WARC file
    num_bytes: the number of bytes to download. Default is 10KB

    return: warc.WARCFile instance
    """
    with closing(requests.get(url, stream=True)) as r:
        buf = StringIO(r.raw.read(num_bytes))
    return warc.WARCFile(fileobj=buf, compress=True)

urls = {
    'warc': 'https://aws-publicdatasets.s3.amazonaws.com/common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/warc/CC-MAIN-20160205193905-00000-ip-10-236-182-209.ec2.internal.warc.gz',
    'wat':  'https://aws-publicdatasets.s3.amazonaws.com/common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/wat/CC-MAIN-20160205193905-00000-ip-10-236-182-209.ec2.internal.warc.wat.gz',
    'wet':  'https://aws-publicdatasets.s3.amazonaws.com/common-crawl/crawl-data/CC-MAIN-2016-07/segments/1454701145519.33/wet/CC-MAIN-20160205193905-00000-ip-10-236-182-209.ec2.internal.warc.wet.gz'
}

files = {file_type: get_partial_warc_file(url=url) for file_type, url in urls.items()}
# this line can be used if you want to download the whole file
# files = {file_type: warc.open(url) for file_type, url in urls.items()}
{% endhighlight %}

The `get_partial_warc_file` function takes the url of a gzipped WARC-formatted file and partially downloads it. A context manager is used to make sure that the `requests` file object is closed when we're done reading. The `warc.open` function takes a local file path; use it instead if you've already downloaded the files.

Since the WET file only contains extracted plaintext, it is easiest to show the relationship between these files by looking at a raw 'response' record and finding the related WAT and WET records:

{% highlight python %}
def get_record_with_header(warc_file, header, value):
    for record, _, _ in warc_file.browse():
        if record.header.get(header) == value:
            return record

warc_record = get_record_with_header(
    files['warc'],
    header='WARC-Type',
    value='response'
)
wat_record = get_record_with_header(
    files['wat'],
    header='WARC-Refers-To',
    value=warc_record.header['WARC-Record-ID']
)
wet_record = get_record_with_header(
    files['wet'],
    header='WARC-Refers-To',
    value=warc_record.header['WARC-Record-ID']
)
{% endhighlight %}

The `get_record_header` function returns the first record with a header matching `value`. Internally, it uses the `warc.WARCFile.browse` method which returns a `(record, offset, size)` tuple for each record. If the file is compressed, the offset and size correspond to the compressed file.

We can check out the headers to verify that these records are indeed related:

{% highlight python %}
print(warc_record.header)
print(wat_record.header)
print(wet_record.header)
{% endhighlight %}

{% highlight text %}
WARC/1.0
Content-Length: 3086
WARC-Concurrent-To: <urn:uuid:d801b36f-2f5c-4c61-bb10-5e01971a81b1>
WARC-Date: 2016-02-05T21:52:39Z
WARC-Payload-Digest: sha1:GXJXSGMV3AHYX4TETSBFFQSCVDJW7QNW
WARC-IP-Address: 212.27.63.107
WARC-Block-Digest: sha1:EOJZD4P2Y6RZCWOB3HLXEE24RCFUGP3D
WARC-Record-ID: <urn:uuid:d9d0e6e8-c58c-40ae-91ea-fc80d6651f3e>
WARC-Target-URI: http://01marocain.online.fr/
WARC-Truncated: length
WARC-Warcinfo-ID: <urn:uuid:a855e809-16cd-4868-9f49-18db4f004e9e>
Content-Type: application/http; msgtype=response
WARC-Type: response

{% endhighlight %}

{% highlight text %}
WARC/1.0
Content-Length: 2262
WARC-Target-URI: http://01marocain.online.fr/
WARC-Refers-To: <urn:uuid:d9d0e6e8-c58c-40ae-91ea-fc80d6651f3e>
WARC-Record-ID: <urn:uuid:8a0f17c9-c963-4ebd-bacf-a1f52b0062e2>
WARC-Date: 2016-02-05T21:52:39Z
Content-Type: application/json
WARC-Type: metadata

{% endhighlight %}

{% highlight text %}
WARC/1.0
Content-Length: 132
WARC-Target-URI: http://01marocain.online.fr/
WARC-Block-Digest: sha1:UH7XO7N2JXTCVZJCJ7HCMX2LMSBFFJ6M
WARC-Refers-To: <urn:uuid:d9d0e6e8-c58c-40ae-91ea-fc80d6651f3e>
WARC-Record-ID: <urn:uuid:34969cd1-9555-4238-84e6-080a70a17342>
WARC-Date: 2016-02-05T21:52:39Z
Content-Type: text/plain
WARC-Type: conversion
{% endhighlight %}

The WAT, and WET record `WARC-Refers-To` header points back to the WARC record. Additionally, the WAT record has `WARC-Type` 'metadata' and the WET record has `WARC-Type` 'conversion', as expected.

Finally, lets print out the content:

{% highlight python %}
import json

print(warc_record.payload.read())
print(json.dumps(json.loads(wat_record.payload.read()), indent=2))
print(wet_record.payload.read())
{% endhighlight %}

The contents of `warc_record`, truncated for display purposes only:
{% highlight html %}
HTTP/1.1 200 OK
Date: Fri, 05 Feb 2016 21:52:35 GMT
Server: Apache/ProXad [Jul 22 2015 14:50:04]
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Pragma: no-cache
X-Powered-By: PHP/5.1.3RC4-dev
Set-Cookie: PHPSESSID=e97e61e9446fe75b7889096e659000e2; path=/
Connection: close
Content-Type: text/html

...TRUNCATED...
  <table cellspacing="1" align="center" width="80%" border="0" cellpadding="10px;">
    <tr>
      <td align="center"><div style="background-color: #DDFFDF; color: #136C99; text-align: center; border-top: 1px solid #DDDDFF; border-left: 1px solid #DDDDFF; border-right: 1px solid #AAAAAA; border-bottom: 1px solid #AAAAAA; font-weight: bold; padding: 10px;">Le site est actuellement ferm� pour maintenance. Merci de r�essayer ult�rieurement.</div></td>
    </tr>
  </table>
  <form action="http://01marocain.online.fr/user.php" method="post">
    <table cellspacing="0" align="center" style="border: 1px solid silver; width: 200px;">
      <tr>
        <th style="background-color: #2F5376; color: #FFFFFF; padding : 2px; vertical-align : middle;" colspan="2">User Login</th>
      </tr>
      <tr>
        <td style="padding: 2px;">Username: </td><td style="padding: 2px;"><input type="text" name="uname" size="12" value="" /></td>
      </tr>
      <tr>
        <td style="padding: 2px;">Password: </td><td style="padding: 2px;"><input type="password" name="pass" size="12" /></td>
      </tr>
      <tr>
        <td style="padding: 2px;">&nbsp;</td>
        <td style="padding: 2px;">
        	<input type="hidden" name="xoops_redirect" value="/" />
        	<input type="hidden" name="xoops_login" value="1" />
        	<input type="submit" value="User Login" /></td>
      </tr>
    </table>
  </form>
...TRUNCATED...
{% endhighlight %}

The contents of `wat_record`:
{% highlight json %}
{
  "Container": {
    "Filename": "CC-MAIN-20160205193905-00000-ip-10-236-182-209.ec2.internal.warc.gz",
    "Offset": "846",
    "Compressed": true,
    "Gzip-Metadata": {
      "Header-Length": "10",
      "Inflated-CRC": "2065955525",
      "Inflated-Length": "3650",
      "Deflate-Length": "1550",
      "Footer-Length": "8"
    }
  },
  "Envelope": {
    "WARC-Header-Length": "560",
    "Format": "WARC",
    "Block-Digest": "sha1:EOJZD4P2Y6RZCWOB3HLXEE24RCFUGP3D",
    "WARC-Header-Metadata": {
      "Content-Length": "3086",
      "WARC-Warcinfo-ID": "<urn:uuid:a855e809-16cd-4868-9f49-18db4f004e9e>",
      "WARC-Payload-Digest": "sha1:GXJXSGMV3AHYX4TETSBFFQSCVDJW7QNW",
      "WARC-IP-Address": "212.27.63.107",
      "WARC-Block-Digest": "sha1:EOJZD4P2Y6RZCWOB3HLXEE24RCFUGP3D",
      "WARC-Type": "response",
      "WARC-Concurrent-To": "<urn:uuid:d801b36f-2f5c-4c61-bb10-5e01971a81b1>",
      "WARC-Target-URI": "http://01marocain.online.fr/",
      "WARC-Truncated": "length",
      "WARC-Date": "2016-02-05T21:52:39Z",
      "Content-Type": "application/http; msgtype=response",
      "WARC-Record-ID": "<urn:uuid:d9d0e6e8-c58c-40ae-91ea-fc80d6651f3e>"
    },
    "Payload-Metadata": {
      "Actual-Content-Type": "application/http; msgtype=response",
      "HTTP-Response-Metadata": {
        "Headers-Length": "379",
        "HTML-Metadata": {
          "Head": {
            "Metas": [
              {
                "content": "text/html; charset=ISO-8859-1",
                "http-equiv": "content-type"
              },
              {
                "content": "en",
                "http-equiv": "content-language"
              }
            ],
            "Link": [
              {
                "url": "http://01marocain.online.fr/xoops.css",
                "path": "LINK@/href",
                "type": "text/css",
                "rel": "stylesheet"
              }
            ],
            "Title": "01marocain"
          },
          "Links": [
            {
              "url": "http://01marocain.online.fr/themes/02/logo.gif",
              "path": "IMG@/src",
              "alt": ""
            },
            {
              "url": "http://01marocain.online.fr/",
              "path": "A@/href"
            },
            {
              "url": "http://01marocain.online.fr/user.php",
              "path": "FORM@/action",
              "method": "post"
            }
          ]
        },
        "Entity-Digest": "sha1:GXJXSGMV3AHYX4TETSBFFQSCVDJW7QNW",
        "Headers": {
          "X-Powered-By": "PHP/5.1.3RC4-dev",
          "Set-Cookie": "PHPSESSID=e97e61e9446fe75b7889096e659000e2; path=/",
          "Expires": "Thu, 19 Nov 1981 08:52:00 GMT",
          "Server": "Apache/ProXad [Jul 22 2015 14:50:04]",
          "Connection": "close",
          "Pragma": "no-cache",
          "Cache-Control": "no-store, no-cache, must-revalidate, post-check=0, pre-check=0",
          "Date": "Fri, 05 Feb 2016 21:52:35 GMT",
          "Content-Type": "text/html"
        },
        "Entity-Trailing-Slop-Bytes": "0",
        "Response-Message": {
          "Status": "200",
          "Reason": "OK",
          "Version": "HTTP/1.1"
        },
        "Entity-Length": "2707"
      },
      "Trailing-Slop-Length": "4"
    },
    "Actual-Content-Length": "3086"
  }
}
{% endhighlight %}

The contents of `wet_record`:
{% highlight text %}
Le site est actuellement ferm� pour maintenance. Merci de r�essayer ult�rieurement.
User Login
Username: Password:
{% endhighlight %}

The WARC record content block contains the complete server response and contains the headers and response body received by the crawler.

The WAT record contains metadata pertaining to its associated WARC record. This includes the file that contains the record and the record's offset and the full set of WARC record headers in JSON format, including content length. This makes it possible to work with the smaller WAT file while still making it easy to find and download only the record containing the raw data if needed.

Finally, the WET record is much smaller and contains just the plaintext of the original WARC record. If all you need is the text, this file is much smaller and obviates parsing the full HTML response.
