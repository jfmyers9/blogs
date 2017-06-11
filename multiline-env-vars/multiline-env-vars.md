# Multiline Environment Variables

A few days ago I found myself in a situation in which I needed to provide
credentials to an application through environment variables.  One set of
credentials needed were certificate and key files to be used for mutual auth
TLS communication with another service.  These credentials were delivered in
the usual format of:
```
-----BEGIN CERTIFICATE-----
MIIDtzCCAp+gAwIBAgIJAMZ/qRdRamluMA0GCSqGSIb3DQEBBQUAMEUxCzAJBgNV
...
-----END CERTIFICATE-----
```

Naively I passed these credentials as anonymous file descriptors to my
application as so:

```
$ ./application --ca-cert <(echo $CA_CERT) --client-cert <(echo $CLIENT_CERT) --client-key <(echo $CLIENT_KEY)
```

Yet continually I found that my application was unable to parse the
certificates and keys provided.  At first I thought that maybe the contents of
the environment variable were incorrect, but after inspecting the processes
environment through the `/proc` file system, I found that everything looked in
order.  After some investigation, I noticed the surprising behavior from `echo`
that eventually cleared the issue up:

```
$ echo $VAR
this is a multiline environement variable

$ echo "$VAR"
this is a multiline
environement
variable
```

Alas, it was clear. `echo` was outputting each line separated with whitespace,
rather than keeping the original structure.  It seems painfully obvious now,
but one should always be careful about using environment variables in bash as
the actual contents of the environment variable can cause wildly different
behavior.  Hopefully this insight will save someone the time that I spent
debugging this mysterious issue in the future.

[Comments](https://github.com/jfmyers9/blogs/issues/3)
