# SSH Authentication with Github on Fly Fi

*Disclaimer*: JetBlue is awesome for offering Fly-Fi free of charge on domestic
flights in the USA.  They are miles ahead of any other airline in this area, so
this should not be taken as a slight towards their service.

I was recently disappointed to find the following behavior when using JetBlue's in flight WiFi:

```bash
$ ssh git@github.com
Connection reset by 192.30.253.112 port 22
```

After some digging, it became clear that for some reason Fly-Fi blocks all
outgoing traffic on port 22. Using a nifty tool called
[outPorts](https://github.com/nhooyr/outPorts), the list of open ports was in
fact limited to these few:

```bash
$ outPorts -c -t 5 -w 700 alls
success on port 443
success on port 80
success on port 20
success on port 21
success on port 3128
success on port 8080
success on port 8443
```

Determined to still be able to use SSH authentication to authorize with Github,
I began exploring to see if I could find a way around these restrictions. One
obvious way to achieve this would be to use a traditional VPN that would be
accessible on one of the following ports to tunnel traffic through.
Unfortunately, I did not have access to such a VPN at the time of my flight.

After some searching, one specific `ssh_config` option that caught my eye was:

```
ProxyJump
			 Specifies one or more jump proxies as [user@]host[:port].  Multiple proxies may be separated by
			 comma characters and will be visited sequentially.  Setting this option will cause ssh(1) to con‐
			 nect to the target host by first making a ssh(1) connection to the specified ProxyJump host and
			 then establishing a TCP forwarding to the ultimate target from there.

			 Note that this option will compete with the ProxyCommand option - whichever is specified first
			 will prevent later instances of the other from taking effect.
```

Essentially, this option allows a user to proxy all SSH traffic through a
separate SSH connection.

Knowing this, I fired up my non traditional VPN which would only allow me SSH
access to machines on the private network, and quickly validated that in fact I
was able to ssh into some of my personal machines. I was pleasantly surprised
to eventually arrive at following:

```bash
$ ssh -J jfmyers9@10.0.0.1 git@github.com
PTY allocation request failed on channel 0
Hi jfmyers9! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
Killed by signal 1.
```

Lastly to get this configuration to apply globally, I added the following to my `~/.ssh/config`:

```
Host github.com
  ForwardAgent yes
  ProxyJump jfmyers9@10.0.0.1
```

and was able to successfully push this blog to Github using SSH authentication
on Fly-Fi.

This experience has once again reaffirmed my belief that SSH is one of the most
powerful tools available.

[Comments](https://github.com/jfmyers9/blogs/issues/4)
