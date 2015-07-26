---
layout: post
title: Skip SSL HostName Verification Java HttpsURLConnection
date: 2015-07-27 20:39:10 UTC
updated: 2015-07-27 20:39:10 UTC
comments: true
categories: ssl hostname https java homepage
---

There might arise a situation when you have to make a secure request to a server with certificates that do not have host name to which you are making a request.

For example a certificate generated for `https://example.com` might not support its ip address `https://ex.com.ip.add` as a valid host name.

PS: This situation is different from errors arrising from invalid or self signed certificates. This situation arrises when certificates are valid but the host name verification fails.

You might see an error like

```
java.io.IOException: Hostname '0.0.0.0' was not verified
```

Where in place of `0.0.0.0` its the server's ip address.

In such a situation all you need to do is to skip host name verification for the URL connection. You can Override the default `HostnameVerifier` with a custom verifier to add exception for the host you are making request to.

This should be done only if you are confident that the server you are sending request to doesn't has any kind of security issues, because when you skip host name verification there is actually no point of `HTTPS/SSL`.

For a particular host you can write a custom `HostnameVerifier` like:

```
HostnameVerifier hostnameVerifier = new HostnameVerifier() {
    @Override
    public boolean verify(String hostname, SSLSession session) {
        HostnameVerifier hv =
            HttpsURLConnection.getDefaultHostnameVerifier();
        return hv.verify("hostname", session);
    }
};
```

or just `return true` if you want to skip for all hosts on a particular URL connection.

A sample code that fetches JSON data using such request:

```
import android.util.Log;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.lang.Exception;
import java.net.URL;

import javax.net.ssl.HostnameVerifier;
import javax.net.ssl.HttpsURLConnection;
import javax.net.ssl.SSLSession;

public class SkipVerification implements Runnable {

    private static final String TAG = SkipVerification.class.toString();
    private String server_port;
    private String serverIP;

    public SkipVerification(String serverIP, String server_port){
        this.serverIP = serverIP;
        this.server_port = server_port;
    }

    public void run() {
        try {
            HostnameVerifier hostnameVerifier = new HostnameVerifier() {
                @Override
                public boolean verify(String hostname, SSLSession session) {
                    return true;
                }
            };

            URL url = new URL("https://" + serverIP + ":" + server_port + "/json");
            InputStream inStream = null;

            try {
                HttpsURLConnection urlConnection = (HttpsURLConnection)url.openConnection();
                urlConnection.setRequestProperty("Content-Type", "application/json");
                urlConnection.setHostnameVerifier(hostnameVerifier);
                inStream = urlConnection.getInputStream();
                BufferedReader reader = new BufferedReader(new InputStreamReader(inStream));
            } catch (Exception e) {
                Log.e(TAG, "error fetching data from server", e);
            } finally {
                if (inStream != null) {
                    inStream.close();
                }
            }
        } catch (Exception e) {
            Log.e(TAG, "error initializing SkipVerificationn thread", e);
        }
    }
}
```

The above code makes a `https` request to a server whose certificate doesn't have an entry of its ip address as a verified hostname. The custom `HostnameVerifier` skips any kind of hostname verification particular to a `HttpsURLConnection`.
