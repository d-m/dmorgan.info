---
layout: post
title: "Mac OS X Network Proxy Settings in Terminal"
excerpt: "Mac OS X doesn't configure the command-line network proxy automatically when switching between wired and wireless networks. This post shows you how."
modified: 2015-11-30 19:59:52 -0500
category: posts
tags: [terminal, bash, network, proxy, mac os x, scripts, configuration, shell, system preferences, scutil, awk]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

Mac OS X does a good job of juggling proxy configurations for graphical applications while moving between wired and wireless network connections. However, this functionality doesn't extend to command-line work in Terminal or iTerm and can be a pain when using git or package managers like npm, apm, pip, or homebrew while switching between environments. This post describes a method for programmatically setting the command-line network proxy environment variables based on the configured proxy in the Network System Preferences pane.

## Mac OS X Proxy Behavior ##

Mac OS X maintains individual network proxy settings for each network adapter. For example, a Thunderbolt ethernet adapter has its own proxy configuration associated with it that is separate from a wireless adapter. The operating system uses the proxy configuration for the currently-connected adapter, updating the system proxy as adapter connection states change. If more than one adapter is connected, the operating system uses the proxy configuration for the connected adapter highest on the adapter list in the Network System Preferences pane. The adapter order can be changed by clicking on the gear icon at the bottom of the list and clicking the "Set Service Order..." menu item.

<figure class="half">
    <a href="/images/connected-network-adapters.png"><img src="/images/connected-network-adapters.png" alt="two connected network adapters in Mac OS X Network System Preference pane"></a>
    <a href="/images/set-service-order.png"><img src="/images/set-service-order.png" alt="the network adapter configuration menu with 'Set Service Order...' highlighted"></a>
    <figcaption>On the left, two network adapters are active. Because the "Display Ethernet" network adapter is higher on the list, its proxy configuration takes precedence. On the right, the "Set Service Order..."" menu item can be used to change the precedence of the configured network adapters.</figcaption>
</figure>

## Access System Proxy settings in Terminal ##

The `scutil` command-line utility allows for management for a variety of system configuration parameters and can be used to access system proxy configuration with the `--proxy` flag.

Here is the output of `scutil --proxy` without a configured proxy:

{% highlight console %}
$ scutil --proxy
<dictionary> {
  FTPPassive : 1
  HTTPEnable : 0
  HTTPSEnable : 0
}
{% endhighlight %}

and here is the output of `scutil --proxy` with `example.proxy` set as the system proxy for the HTTP and HTTPS protocols:

{% highlight console %}
$ scutil --proxy
<dictionary> {
  FTPPassive : 1
  HTTPEnable : 1
  HTTPPort : 80
  HTTPProxy : example.proxy
  HTTPSEnable : 1
  HTTPSPort : 80
  HTTPSProxy : example.proxy
}
{% endhighlight %}

## Parse `scutil` output ##

We can use `awk` to parse the output of `scutil` and extract the proxy configuration. The following snippet does the trick:

{% highlight console %}
$ export http_proxy=`scutil --proxy | awk '\
  /HTTPEnable/ { enabled = $3; } \
  /HTTPProxy/ { server = $3; } \
  /HTTPPort/ { port = $3; } \
  END { if (enabled == "1") { print "http://" server ":" port; } }'`
$ export HTTP_PROXY="${http_proxy}"
{% endhighlight %}

This script looks for `HTTPEnable`, `HTTPProxy`, and `HTTPPort` in the output of `scutil`. If the proxy is enabled, the script prints out the proxy URL and sets it as the `http_proxy` environment variable. If the proxy is not enabled, the script sets `http_proxy` to an empty string. The final line sets the `HTTP_PROXY` environment variable as well since some command-line applications use that instead.

Placing this snippet in your `.bash_profile` ensures that your proxy will stay configured automatically while switching between wired and wireless networks.
