Configure TURN
----------

This document covers how to setup a TURN server for BigBlueButton to allow users behind restrictive firewalls to connect.

You can also use [bbb-install.sh](https://github.com/bigbluebutton/bbb-install#install-a-turn-server) to automate the steps in this document.

Setup a TURN server
==========

BigBlueButton normally requires a wide range of UDP ports to be available for WebRTC communication. In some network restricted sites or development environments, such as those behind NAT or a firewall that restricts outgoing UDP connections, users may be unable to make outgoing UDP connections to your BigBlueButton server.

The [TURN protocol](https://en.wikipedia.org/wiki/Traversal_Using_Relays_around_NAT) is designed to allow UDP-based communication flows like WebRTC to bypass NAT or firewalls by having the client connect to the TURN server, and then have the TURN server connect to the destination on their behalf.

In addition, the TURN server implements the STUN protocol as well, used to allow direct UDP connections through certain types of firewalls which otherwise might not work.

Using a TURN server under your control improves the success of connections to BigBlueButton and also improves user privacy, since they will no longer be sending IP address information to a public STUN server.

Required Hardware
----------

The TURN protocol is not CPU or memory intensive. Additionally, since it’s only used during connection setup (for STUN) and as a fallback for users who would otherwise be unable to connect, the bandwidth requirements aren’t particularly high. For a moderate number of BigBlueButton servers, a single small VPS is usually sufficient.

Having multiple IP addresses may improve the results when using STUN with certain types of firewalls, but is not usually necessary.

Having the server behind NAT (for example, on Amazon EC2) is OK, but all incoming UDP and TCP connections on any port must be forwarded and not firewalled.

Required Software
----------

We recommend using a minimal server installation of Ubuntu 20.04. The [coturn](https://github.com/coturn/coturn) software requires port 443 for its exclusive use in our recommended configuration, which means the server cannot have any dashboard software or other web applications running.

Stable versions of coturn are already available in the Ubuntu packaging repositories for version 20.04 and later, and it can be installed with apt-get:

```
$ sudo apt-get update
$ sudo apt-get install coturn

```

Note: coturn will not automatically start until configuration is applied (see below).

Required DNS Entry
----------

You need to setup a fully qualified domain name that resolves to the external IP address of your turn server. You’ll use this domain name to generate a TLS certificate using Let’s Encrypt (next section).

Required Ports
----------

On the coturn server, you need to have the following ports (in addition port 22) available for BigBlueButton clients to connect (port 3478 and 443) and for coturn to connect to your BigBlueButton server (32769 - 65535).

|   Ports   |Protocol|     Description     |
|-----------|--------|---------------------|
|   3478    |TCP/UDP |coturn listening port|
|    443    |TCP/UDP | TLS listening port  |
|32769-65535|  UDP   |  relay ports range  |

Generating TLS certificates
----------

You can use `certbot` from [Let’s Encrypt](https://letsencrypt.org/) to easily generate free TLS certificates. To setup `certbot` enter the following commands on your TURN server (not your BigBlueButton server).

```
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install certbot

```

You can then run a `certbot` command like the following to generate the certificate, replacing `` with the domain name of your TURN server:

```
$ sudo certbot certonly --standalone --preferred-challenges http \
    -d

```

Be aware that TCP 80 needs to be open temporarily to get it to work.

Current versions of the certbot command set up automatic renewal by default. To ensure that the certificates are readable by `coturn`, which runs as the `turnserver` user, add the following renewal-hook to Let’s Encrypt. First, create the directory `/etc/letsencrypt/renewal-hooks/deploy`.

```
$ sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy

```

Next, create the file `/etc/letsencrypt/renewal-hooks/deploy/coturn` with the following contents. Replace  with the hostname of your TURN server.

```
#!/bin/bash -e
for certfile in fullchain.pem privkey.pem ; do
	cp -L /etc/letsencrypt/live//"${certfile}" /etc/turnserver/"${certfile}".new
	chown turnserver:turnserver /etc/turnserver/"${certfile}".new
	mv /etc/turnserver/"${certfile}".new /etc/turnserver/"${certfile}"
done
systemctl kill -sUSR2 coturn.service

```

Make this file executable.

```
$ sudo chmod 0755 /etc/letsencrypt/renewal-hooks/deploy/coturn

```

Configure coturn
----------

`coturn` configuration is stored in the file `/etc/turnserver.conf`. There are a lot of options available, all documented in comments in the default configuration file. We include a sample configuration below with the recommended settings (refer to the default configuration file for more information on the settings)..

Use the file below for `/etc/turnserver.conf` and make the following changes:

* Replace `` with the hostname of your TURN server, and
* Replace `` with the realm of your TURN server, and
* Replace `` to a random value for a shared secret (you can generate one by running `openssl rand -hex 16`)
* Replace `` with the external IP of your TURN server

This configuration file assumes your TURN server is not behind NAT and has a public IP address.

```
listening-port=3478
tls-listening-port=443

listening-ip=
relay-ip=

# If the server is behind NAT, you need to specify the external IP address.
# If there is only one external address, specify it like this:
#external-ip=172.17.19.120
# If you have multiple external addresses, you have to specify which
# internal address each corresponds to, like this. The first address is the
# external ip, and the second address is the corresponding internal IP.
#external-ip=172.17.19.131/10.0.0.11
#external-ip=172.17.18.132/10.0.0.12

min-port=32769
max-port=65535
verbose

fingerprint
lt-cred-mech
use-auth-secret
static-auth-secret=
realm=

cert=/etc/turnserver/fullchain.pem
pkey=/etc/turnserver/privkey.pem
# From https://ssl-config.mozilla.org/ Intermediate, openssl 1.1.0g, 2020-01
cipher-list="ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384"
dh-file=/etc/turnserver/dhp.pem

keep-address-family
no-cli
no-tlsv1
no-tlsv1_1

# Block connections to IP ranges which shouldn't be reachable
no-loopback-peers
no-multicast-peers
# CVE-2020-26262
# If running coturn version older than 4.5.2, uncomment these rules and ensure
# that you have listening-ip set to ipv4 addresses only.
#denied-peer-ip=0.0.0.0-0.255.255.255
#denied-peer-ip=127.0.0.0-127.255.255.255
#denied-peer-ip=::1
# Private (LAN) addresses
# If you are running BigBlueButton within a LAN, you might need to add an "allow" rule for your address range.
# IPv4 Private-Use
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=192.168.0.0-192.168.255.255
# Other IPv4 Special-Purpose addresses
denied-peer-ip=100.64.0.0-100.127.255.255
denied-peer-ip=169.254.0.0-169.254.255.255
denied-peer-ip=192.0.0.0-192.0.0.255
denied-peer-ip=192.0.2.0-192.0.2.255
denied-peer-ip=198.18.0.0-198.19.255.255
denied-peer-ip=198.51.100.0-198.51.100.255
denied-peer-ip=203.0.113.0-203.0.113.255
# IPv6 Unique-Local
denied-peer-ip=fc00::-fdff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
# IPv6 Link-Local Unicast
denied-peer-ip=fe80::-febf:ffff:ffff:ffff:ffff:ffff:ffff:ffff
# Other IPv6 Special-Purpose assignments
denied-peer-ip=::ffff:0:0-::ffff:ffff:ffff
denied-peer-ip=64:ff9b::-64:ff9b::ffff:ffff
denied-peer-ip=64:ff9b:1::-64:ff9b:1:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=2001::-2001:1ff:ffff:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=2001:db8::-2001:db8:ffff:ffff:ffff:ffff:ffff:ffff
denied-peer-ip=2002::-2002:ffff:ffff:ffff:ffff:ffff:ffff:ffff

```

We need to create `dph.pem` file,

```
$ sudo mkdir -p /etc/turnserver
$ sudo openssl dhparam -dsaparam  -out /etc/turnserver/dhp.pem 2048

```

To increase the file handle limit for the TURN server and to give it the ability to bind to port 443, add the following systemd override file. First, create the directory.

```
$ sudo mkdir -p /etc/systemd/system/coturn.service.d

```

and then then create `/etc/systemd/system/coturn.service.d/override.conf` with the following contents

```
[Service]
LimitNOFILE=1048576
AmbientCapabilities=CAP_NET_BIND_SERVICE
ExecStart=
ExecStart=/usr/bin/turnserver --daemon -c /etc/turnserver.conf --pidfile /run/turnserver/turnserver.pid --no-stdout-log --simple-log --log-file /var/log/turnserver/turnserver.log
Restart=always

```

Configure Log Rotation
----------

To rotate the logs for `coturn`, install the following configuration file to `/etc/logrotate.d/coturn`

```
/var/log/turnserver/*.log
{
	rotate 7
	daily
	missingok
	notifempty
	compress
	postrotate
		/bin/systemctl kill -s HUP coturn.service
	endscript
}

```

And create the associated log directory

```
$ sudo mkdir -p /var/log/turnserver
$ sudo chown turnserver:turnserver /var/log/turnserver

```

Restart coturn
----------

With the above steps completed, restart the TURN server

```
$ sudo /etc/letsencrypt/renewal-hooks/deploy/coturn    # Initial copy of certificates
$ sudo systemctl daemon-reload                         # Ensure the override file is loaded
$ sudo systemctl restart coturn                        # Restart

```

Ensure that the `coturn` has binded to port 443 with `netstat -antp | grep 443`. Also restart your TURN server and ensure that `coturn` is running (and binding to port 443 after restart).

Configure BigBlueButton to use your TURN server
==========

You must configure bbb-web so that it will provide the list of turn servers to the web browser. Edit the file `/usr/share/bbb-web/WEB-INF/classes/spring/turn-stun-servers.xml` using the contents below and make edits:

* replace both instances of `` with the hostname of the TURN server, and
* replace `` with the secret you configured in `turnserver.conf`.

```

 xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">

     id="stun0" class="org.bigbluebutton.web.services.turn.StunServer">
         index="0" value="stun:"/>

     id="turn0" class="org.bigbluebutton.web.services.turn.TurnServer">
         index="0" value=""/>
         index="1" value="turns::443?transport=tcp"/>
         index="2" value="86400"/>

     id="turn1" class="org.bigbluebutton.web.services.turn.TurnServer">
         index="0" value=""/>
         index="1" value="turn::443?transport=tcp"/>
         index="2" value="86400"/>

     id="stunTurnService"
            class="org.bigbluebutton.web.services.turn.StunTurnService">
         name="stunServers">

                 bean="stun0"/>

         name="turnServers">

                 bean="turn0"/>
                 bean="turn1"/>

```

Restart your BigBlueButton server to apply the changes.

Going forward, when users connect behind a restrictive firewall that prevents outgoing UDP connections, the TURN server will enable BigBlueButton to connect to FreeSWITCH and Kurento via the TURN server through port 443 on their firewall.

Test your TURN server
==========

By default, your browser will try to connect directly to Kurento or FreeSWITCH using WebRTC. If it is unable to make a direct connection, it will fall back to using the TURN server as one of the interconnectivity connectivity exchange (ICE) candidates to relay the media.

Use FireFox to test your TURN server. FireFox allows you to disable direct connections and require fallback to your TURN server. Launch FireFox, open `about:config`, and search for ‘relay`. You should see a parameter `media.peerconnection.ice.relay\_only`. Set this value to `true`.

With FireFox configured to only use a TURN server, open a new tab and join a BigBlueButton session, and share your webcam. If your webcam appears, you can verify that FireFox is using your TURN server by opening a new tab and choosing `about:webrtc`. Click `show details` and you’ll see a table for ICE Stats. The successful connection, shown at the top of the table, should have `(relay-tcp)` in the Local Candidate column. This means the video connection was successfully relayed through your TURN server.

If, however, you received a 1020 (unable to establish connection) when sharing a webcam the browser may not be able to connect to the TURN server or the TURN server is not running or configured correctly. Check the browser console in FireFox. If you see

```
WebRTC: ICE failed, your TURN server appears to be broken, see about:webrtc for more details

```

then FireFox was unable to communicate with your TURN server, or your TURN server was not running or configured correctly.

To ensure that your firewall is not blocking UDP connections over port 443, open a new tad visit <https://test.bigbluebutton.org/>, launch a test session, and try sharing your webcam.
the browser may not be able to connect to the TURN server or the TURN server is not running or configured correctly.

The TURN server also acts as a STUN server, so you can first check that the STUN portion is working using the `stunclient`. Run the following commands below and substitute `` with the hostname of your TURN server.

```
sudo apt-get install -y stuntman-client
stunclient --mode full --localport 30000  3478

```

If successful, you should see output for `stunclient` should be similar to the following.

```
Binding test: success
Local address: xxx.xxx.xxx.xxx:30000
Mapped address: xxx.xxx.xxx.xxx:30000
Behavior test: success
Nat behavior: Direct Mapping
Filtering test: success
Nat filtering: Endpoint Independent Filtering

```

If you get an error, check that `coturn` is running on the TURN server using `systemctl status coturn.service`. Check the logs by doing `tail -f /var/log/turnserver/coturn.log`. You can get verbose logs by adding `verbose` to `/etc/turnserver.conf` and restarting the TURN server `systemctl restart coturn.service`

You can test your TURN server using the [trickle ICE](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/) page. This gives you a log of the relay candidates as they are returned from ICE gathering. To test using this page, you need to generate some test credentials. Run the following BASH script and substitute `` with the hostname of your TURN server and `` with the password for your TURN server.

```
#!/bin/bash

HOST=
SECRET=

time=$(date +%s)
expiry=8400
username=$(( $time + $expiry ))

echo
echo "          https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/"
echo
echo      URI : turn:$HOST:443
echo username : $username
echo password : $(echo -n $username | openssl dgst -binary -sha1 -hmac $SECRET | openssl base64)
echo

```

Enter the values into URI, username, and password into the trickle ICE page and click ‘Gather candidates’. You should see a list of relay candidates. If you don’t, again check that your TURN server is running and tail the logs TURN server logs via `tail -f /var/log/turnserver/coturn.log` or `journalctl -f -u coturn.service`.

You can get verbose logs by adding `verbose` to `/etc/turnserver.conf` and then restarting the TURN server `systemctl restart coturn.service`, and try testing again from FireFox or the above Tricke ICE page.