---
layout: post
title: "What the =?UTF-8?B?ZnVjayDwn5CO?=!"
modified: 2015-07-25 10:05:56 -0400
category: posts
tags: [encoded word syntax, email, mime, python, base64, rfc 20822, rfc 2047, rfc 2045, character set, encoding, ascii, iana, charset]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

## Encoded-word Syntax ##

Email originated as an ASCII-only communications medium. Multipurpose Internet Mail Extensions extended email to, among many other things, display body text and header information in character sets other than ASCII. As of ​[RFC 2822](https://tools.ietf.org/html/rfc2822), this is accomplished by using encoded-word syntax defined in [RFC 2047](https://tools.ietf.org/html/rfc2047). This syntax encodes non-ASCII text as an ASCII text string as follows:

    =?<charset>?<encoding>?<encoded-text>?=

where:

  * `charset` is one of the character sets registered with IANA for use with the MIME text/plain content-type
  * `encoding` is either the character Q or B, where B represents base64 encoding and Q represents an encoding similar to the "quoted-printable" content transfer encoding defined in ​[RFC 2045](https://tools.ietf.org/html/rfc2045)
  * `encoded-text` is the text encoded according to the defined `encoding`

It is the email clients' responsibility to take non-ASCII strings, encode them properly, and then decode them for display. Unfortunately, this does not always seem to be the case. Issues can be as simple as character sets or encodings specified in lower case ([RFC 2047](https://tools.ietf.org/html/rfc2047) specifies uppercase characters, only) to character sets that don't correctly decode the encoded text. Several character sets are supersets of others. These larger sets should be used to decode encoded text where possible to reduce the chance of decoding errors. This [GitHub issue comment](https://github.com/buriy/python-readability/issues/42#issuecomment-26580084) lists a few of these character sets:

  * 'big5' should be decoded with 'big5hkscs'
  * 'gb2312' should be decoded with 'gb18030'
  * 'ascii' should be decoded with 'utf-8'
  * 'iso-8859-1' should be decoded with 'cp1252'

## Encoding Example ##
The following Python code make use of the `base64` and `quopri` modules to translate text into encoded-word syntax:

{% highlight python %}
import base64, quopri

def text_to_encoded_words(text, charset, encoding):
    """
    text: text to be transmitted
    charset: the character set for text
    encoding: either 'q' for quoted-printable or 'b' for base64
    """
    byte_string = text.encode(charset)
    if encoding.lower() is 'b':
        encoded_text = base64.b64encode(byte_string)
    elif encoding.lower() is 'q':
        encoded_text = quopri.encodestring(byte_string)
    return "=?{charset}?{encoding}?{encoded_text}?=".format(
        charset=charset.upper(),
        encoding=encoding.upper(),
        encoded_text=encoded_text.decode('ascii'))
{% endhighlight %}

Now we can encode our favorite emoji 🐎 (or more formally, the U+1F40E Unicode code point) to send it in an email:

{% highlight python %}
In [1]: text_to_encoded_words('This is a horsey: \U0001F40E', "utf-8", "b")
Out[1]: '=?UTF-8?B?VGhpcyBpcyBhIGhvcnNleTog8J+Qjg==?='

In [2]: text_to_encoded_words('This is a horsey: \U0001F40E', "utf-8", "q")
Out[2]: '=?UTF-8?Q?This is a horsey: =F0=9F=90=8E?='
{% endhighlight %}

Note: This output is from Python 3. You will need to change `"\U0001F40E"` to `u"\U0001F40E"` to see the same results using Python 2.

Now our horse is ready to email! This output shows a benefit of using the 'Q' encoding for strings that are primarily composed of ASCII characters. One can read most of the message in the second output without first decoding the encoded words.

## Decoding Example ##

{% highlight python %}
def encoded_words_to_text(encoded_words):
    encoded_word_regex = r'=\?{1}(.+)\?{1}([B|Q])\?{1}(.+)\?{1}='
    charset, encoding, encoded_text = re.match(encoded_word_regex, encoded_words).groups()
    if encoding is 'B':
        byte_string = base64.b64decode(encoded_text)
    elif encoding is 'Q':
        byte_string = quopri.decodestring(encoded_text)
    return byte_string.decode(charset)
{% endhighlight %}

This code applies a regular expression to pull out the character set, encoding, and encoded text from the encoded words. Next, it decodes the encoded words into a byte string, using either the `quopri` module or `base64` module as determined by the encoding. Finally, it decodes the byte string using the character set and returns the result.

Running this function on our encoded words returns the Unicode code point for our  horsey:

{% highlight python %}
In [3]: encoded_words_to_text('=?UTF-8?B?VGhpcyBpcyBhIGhvcnNleTog8J+Qjg==?=')
Out[3]: 'This is a horsey: \U0001f40e'

In [4]: encoded_words_to_text('=?UTF-8?Q?This is a horsey: =F0=9F=90=8E?=')
Out[4]: 'This is a horsey: \U0001f40e'
{% endhighlight %}

We get our original string back in this output. In addition to these functions, email headers with encoded words can be manipulated with the `email.headers` Python module.
