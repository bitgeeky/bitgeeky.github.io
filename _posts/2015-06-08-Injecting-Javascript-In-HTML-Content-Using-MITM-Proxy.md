---
layout: post
title: Injecting Javascript In HTML Content Using MITM Proxy
date: 2015-06-08 21:43:10 UTC
updated: 2015-06-08 21:43:10 UTC
comments: true
categories: python proxy mitm https server javascript html homepage
---

Its been a few days that I have been learning about Man In The Middle MITM proxy, my use case was to inject a simple javascript to html pages that users open behind proxy server.

It actually involves a bit insight on how to do this, and a few small challenges. Here I describe the solutions I found to do this:

What is MITM proxy ?
----
An interactive console program that allows traffic flows to be intercepted, inspected, modified and replayed.
So basically it gives the proxy administartor the power to modify any traffic that goes through the proxy. You can play with html content, inject elements, get header data, modify headers, dns spoofing, traffic filteration, redirection and a lot more things you can do with mitmproxy.
<br/>
[Example scripts for doing experimentation](https://github.com/mitmproxy/mitmproxy/tree/master/examples)
<br/>
For more information visit [mitmproxy official website](https://mitmproxy.org/)

Installation
----
For OS X its distributed as a homebrew package and is really simple to install, just do `$ brew install mitmproxy`.
For troubleshooting and setting environment variables have a look at the [installation guide](https://mitmproxy.org/doc/install.html)

Make sure you have all the certificates used by mitm proxy in `~/.mitmproxy/`. Install certificates on browser by browsing to `mitm.it`. If the traffic is passing through the proxy server this page will show you options to install certificates, just select the platform you are browsing on.  For more information about generating and installing certificates see [about cerificates](https://mitmproxy.org/doc/certinstall.html)

Running
----
```
$ mitmproxy -p port_number
```
The default port on which proxy server runs is `8080` but you can specify the `port_number` by using `-p` flag. It will open up a window showing traffic passing through proxy.

Configure browser to use proxy server by specifying host as `localhost` and the port on which proxy server is listening.
You will start seeing requests though the proxy and traffic passing through it.
<img src="/public/images/mitmproxy.png" alt="MITM Console" style="margin-right:15px; width: 100%; height: 100%" />

Modifying traffic passing through proxy
----
You can pass scripts as arguments while running proxy server which will modify the traffic according to the logic you specify in script.
The script will operate on each request passing through proxy, and will make the desired modification.

###Inline Scripts
Basic Structure of an Inline script is:

```
def on_event(context, flow):
    some_logic
    another_logic
    do_something
```

The first argument to each event method is an instance of ScriptContext that lets the script interact with the global mitmproxy state. 
`on_event` can be an event like request, response, start, clientconnect, serverconnect or any other event from this [list](https://mitmproxy.org/doc/scripting/inlinescripts.html)

Script to inject javascript to html content:

```
# Usage: mitmdump -s "js_injector.py src"
# (this script works best with --anticache)
from bs4 import BeautifulSoup
from libmproxy.protocol.http import decoded

# On start of proxy server ask for src as an argument
def start(context, argv):
    if len(argv) != 2:
        raise ValueError('Usage: -s "js_injector.py src"')
    context.src_url = argv[1]


def response(context, flow):
    if flow.request.host in context.src_url:
        return # Make sure JS isn't injected to itself
    with decoded(flow.response):  # Remove content encoding (gzip, ...)
        html = BeautifulSoup(flow.response.content)
        if html.body:
            script = html.new_tag(
                "script",
                src=context.src_url,
                type='application/javascript')
            html.body.insert(0, script)
            flow.response.content = str(html)
            context.log("Script injected.")
```

Run the mitm proxy using:

```
$ mitmdump -p 8888 -s "js_injector.py http://path/to/src.js"
```

Note: that we are using `mitmdump` istead of `mitmproxy` for getting more detailed output.

Note: If you want to inject javascript in an `https` website, your javascript file must also be hosted on `https` server.


So far so good:
Now open the url in browser configured to use proxy server, you should see a broken website.

### Any guess why the website is broken ?

The website is broken because in previous script our javascript is getting injected on every resource on page having `html.body` which should not be the case.

Replace:

```
-- if html.body:
with
++ if html.body and ('text/html' in flow.response.headers["content-type"][0]):
```

to make sure only requests with `text/html` in their header get js injection.

Try to open the page again, it should now open fine with _almost_ all resources loading properly.

### So why _almost_ all and not _ALL_ ? Try to figure it out and read further:

The reason is that an html page has multiple resources that can be of type text/html for example an iframe loading content from an external resource. But we want to inject our js only to the page that is opened by user at current time i.e the main page.

Its not possible to do this without knowing what page is the main page and what are the elements within it. Therefore to overcome this situation we come up with a filter.

filter.js

```
if(parent.document.URL!=document.location.href)
       throw new Error("Not the main page");

(function(e){e.setAttribute("src","http://path/to/script.js");
document.getElementsByTagName("body")[0].appendChild(e);})
(document.createElement("script"));void(0);
```

So basically injecting this filter on proxy server side by replacing path/to/script.js with the path to actual js that we want to inject will do a client side verification to identify the main page and allow execution of js only on main page otherwise it will throw an error "Not the main page".

This solves our problem of js getting executed multiple times on a single page. Now the js we inject will execute only once but still on the proxy server side our js filter will get inject to all valid html resources i.e which have `text/html` as their `content-type` in header response.

Final Steps:
----
Inject JS filter on html resources:

Final version of `js_injector.py`

```
# Usage: mitmdump -s "js_injector.py src"
# (this script works best with --anticache)
from bs4 import BeautifulSoup
from libmproxy.protocol.http import decoded

# On start of proxy server ask for src as an argument
def start(context, argv):
    if len(argv) != 2:
        raise ValueError('Usage: -s "js_injector.py src"')
    context.src_url = argv[1]


def response(context, flow):
    with decoded(flow.response):  # Remove content encoding (gzip, ...)
        html = BeautifulSoup(flow.response.content)
        """
        # To Allow CORS
        if "Content-Security-Policy" in flow.response.headers:
            del flow.response.headers["Content-Security-Policy"]
        """
        if html.body and ('text/html' in flow.response.headers["content-type"][0]):
            script = html.new_tag(
                "script",
                src=context.src_url)
            html.body.insert(0, script)
            flow.response.content = str(html)
            context.log("******* Filter Injected *******")
```


```
$ mitmdump -p 8888 -s "js_injector.py http://path/to/filter.js"
```

Final version of `filter.js`:

```
if(parent.document.URL!=document.location.href)
       throw new Error("Not the main page");

(function(e){e.setAttribute("src","http://path/to/script.js");
document.getElementsByTagName("body")[0].appendChild(e);})
(document.createElement("script"));void(0);

console.log("******* Script Injected *******")
```

Replace `/path/to/script.js` in `filter.js` with path to the script you want to actually inject.

That's it ! Now the JS you inject will be executed only one time per page i.e the actual page and not the resources in it.
