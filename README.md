# local-host DNS

When developing with Docker Desktop for Mac (and minikube on Linux in non-bare modes) one challenge
can be that services running inside and outside of the containers cannot rely on `localhost` to
resolve to the same service. Yes, you can make use of `host.docker.internal` to reach out of the
containers, but this doesn't work outside of the Docker environment, and is not representative
of an operational setting.

This guide details how to:

1. Set up a new loop back alias on an IP in a private block (`172.173.174.175`),
2. Configure [dnsmasq](http://example.com) so that a specific domain routes to IP address of the loopback
3. Update iptables within the VM running inside Docker Desktop so that the IP Address for the loopback routes to the host.

Generally documentation for [dnsmasq](http://example.com) will suggest routing to `127.0.0.1` but
this will not work inside and outside the containers as `127.0.0.1` are different places. With this
set up dnsmasq will alway route to `172.173.174.175` outside of the VM that resolves to the
loopback while inside the VM it is routed to the Mac.

## Set up a loopback alias

You can set up a loopback alias with the following command.

```
sudo ifconfig lo0 alias 172.173.174.175
```

However, this will not persist through restarts. If you wish to do that you need to configure a
`plist` file (e.g. `/Library/LaunchDaemons/localhost.docker.kubernetes.loopback.plist`) with the
following content:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>localhost.docker.kubernetes.loopback.plist</string>
    <key>ProgramArguments</key>
    <array>
        <string>ifconfig</string>
        <string>lo0</string>
        <string>alias</string>
        <string>172.173.174.175</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

## Configure dnsmasq

Assuming you've installed [brew](https://brew.sh) then you can install dnsmasq by running:

```
brew install dnsmasq
```

You then need to ensure the following line is uncommented in `/usr/local/etc/dnsmasq.conf`

```
# Include all files in a directory which end in .conf
conf-dir=/usr/local/etc/dnsmasq.d/,*.conf
```

And then create new file in `/usr/local/etc/dnsmasq.d/` that ensures that `*.localhost` will
be resolved to the IP address we configured before.

```
address=/.localhost/172.173.174.175
```

And restart the service so the new settings take affect:

```
sudo brew services restart dnsmasq
```

(**Note** The brew instructions advise against running with sudo and having run the command with
sudo it wants about changing permissions on some paths. However, running without sudo did not work
for me - possibly something for more investigation.)

The last step required is to get your mac to use the new `dnsmasq` configuration to resolve
`*.localhost` addresses. To do this, create a new file `/etc/resolver/localhost` with the
necessary content.

```
sudo echo "nameserver 127.0.0.1" > /etc/resolver/localhost
```

## Configure iptables

For this to work inside the container it is necessary to route the IP address that `dnsmasq` will
return to the the IP address that maps back to the host `192.168.65.2`. To do this we can use
a container in privileged mode

```
docker run -it --privileged --pid=host justincormack/nsenter1 /bin/bash -c "iptables -t nat -A OUTPUT -d 172.173.174.175 -j DNAT --to-destination 192.168.65.2"
```

## Test it out

First start a service running on the host - for example an [`echo-server`](https://github.com/rpmiskin/echo-server).

```
git clone https://github.com/rpmiskin/echo-server.git
cd echo-server
npm i
npm start
```

Now check that the DNS resolves on the host:

```
curl dev.localhost:3000
```

this should return something like:

```
{"method":"GET","url":"/","headers":{"host":"dev.localhost:3000","user-agent":"curl/7.64.1","accept":"*/*"},"query":{},"body":{}}
```

Finally, check this works inside a container

```
docker run -it curlimages/curl dev.localhost:3000
```

And if you want to be _really_ sure that it is working because of the settings here, try changing
`dev.localhost` to just `localhost`. You should find it works fine on the host, but not in the
container.

## Credits

This guide is _very_ heavily based on
[this blog post](https://revelry.co/resources/development/kubernetes-docker-desktop/) from
[Revelry](https://revelry.co). Mostly duplicated to ensure that I can find it in future if needs be,
and that the blog post currently (17th Oct 2021) seems to break MacOS Safari...
