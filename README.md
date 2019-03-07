# OpenVPN-Server
<p>This is where you’ll find step-by-step instructions regarding how to build an <b>OpenVPN Server</b> on the Amazon AWS EC2.</p>

<p>Since you are here and seeking instructions to build your own OpenVPN server, so I would assume that you should have an Amazon AWS account already and some basic knowledge about AWS.</p>

<h2>Launch EC2 Instance in Preferred Region</h2>

<p>6 easy steps here in total.<br><br><b>Step 1</b>: Choose a region based on your location, needs and budget</p><figure data-orig-height="424" data-orig-width="200"><img src="https://66.media.tumblr.com/04bb483eaf5ac764310f63c1cd3e04d5/tumblr_inline_onl8kw50v61rmhyfa_540.png" data-orig-height="424" data-orig-width="200"></figure><p><br><b>Step 2</b>: Choose Amazon Linux AMI</p><figure class="tmblr-full" data-orig-height="205" data-orig-width="546"><img src="https://66.media.tumblr.com/34e93a5df493fda2f17452fd6e09b59c/tumblr_inline_onl8l3MD2O1rmhyfa_540.png" data-orig-height="205" data-orig-width="546"></figure><p><br><b>Step 3</b>: Choose an instance type based on your needs</p><figure class="tmblr-full" data-orig-height="243" data-orig-width="696"><img src="https://66.media.tumblr.com/3e735870828ff621c8d5868c75c42fc9/tumblr_inline_onl8vn9Ukb1rmhyfa_540.png" data-orig-height="243" data-orig-width="696"></figure><p><br><b>Step 4</b>: Configure Security Group to meet your requirements. Here is my settings, and UDP 1194 is for my OpenVPN server</p><figure class="tmblr-full" data-orig-height="228" data-orig-width="484"><img src="https://66.media.tumblr.com/4da184b9ccf4894eeb1ebcfbd5e80f70/tumblr_inline_onl8zeKK9y1rmhyfa_540.png" data-orig-height="228" data-orig-width="484"></figure><p><br><b>Step 5</b>: Select an existing key pair or create a new key pair i.e. oven.pem</p><p><br><b>Step 6</b>: Launch the instance!<br></p>


<h2><br>Set up OpenVPN server on AWS EC2</h2>

<p>Okay! The following part is going to be a bit technical and requires you to have some Unix knowledge. If you follow everything I said here, you should be able to get a running OpenVPN server in 20 minutes.</p>

<p>10 steps in total.</p>


<p><br>Step 1: ssh into the EC2 instance you set up just now</p>

<pre>$ ssh -i ovpn.pem ec2-user@ec2-1-2-3-4.x.compute.amazonaws.com</pre>

<p>Note: replace ovpn.pem and ec2-1-2-3-4.x.compute.amazonaws.com above with your actual filename and hostname</p>

<p><br>Step 2: Install openvpn and git</p>

<pre>$ sudo yum install -y openvpn
$ sudo yum install -y git
</pre>

<p><br>Step 3: Follow instructions to generate ssh key and add to GitHub account</p>

<p>
<a href="https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/" target="_blank">https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/</a><br><a href="https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/" target="_blank">https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/</a></p>

<p><br>Step 4: Get OpenVPN/easy-rsa</p>

<pre>$ git clone git@github.com:OpenVPN/easy-rsa.git</pre>

<p><br>Step 5: Set Certificate Authority (CA) and generate certificates and keys for an OpenVPN server and clients</p>

<pre>$ sudo mv ~/easy-rsa /etc
$ cd /etc/easy-rsa/easyrsa3
$ ./easyrsa init-pki
$ ./easyrsa build-ca
$ ./easyrsa build-server-full server   ### for openvpn server 
$ ./easyrsa build-client-full macbook   ### for openvpn client on mac
$ ./easyrsa build-client-full ios.  ### for openvpn client on iphone
$ ./easyrsa build-client-full android   ### for openvpn client on android phone
$ ./easyrsa gen-dh
</pre>

<p>Note: You will be asked to set one passphrase for each certificate and key. Choose one you like and you will need it later.</p>

<p><br>Step 6: Have certificates and keys in place for OpenVPN server</p>

<pre>$ cd /etc/openvpn/
$ sudo mkdir keys
$ cd keys
$ sudo cp /etc/easy-rsa/easyrsa3/pki/ca.crt ca.crt
$ sudo cp /etc/easy-rsa/easyrsa3/pki/issued/server.crt server.crt
$ sudo cp /etc/easy-rsa/easyrsa3/pki/dh.pem dh.pem
$ sudo cp /etc/easy-rsa/easyrsa3/pki/private/server.key server.key
</pre>

<p><br>Step 7: Have an OpenVPN server config ovpn.conf ready under /etc/openvpn/. Following is mine as an example.</p>

<pre># Which TCP/UDP port should OpenVPN listen on?
# If you want to run multiple OpenVPN instances
# on the same machine, use a different port
# number for each one.  You will need to
# open up this port on your firewall.
port 1194

# TCP or UDP server?
proto udp

# "dev tun" will create a routed IP tunnel,
# "dev tap" will create an ethernet tunnel.
# Use "dev tap0" if you are ethernet bridging
# and have precreated a tap0 virtual interface
# and bridged it with your ethernet interface.
# If you want to control access policies
# over the VPN, you must create firewall
# rules for the the TUN/TAP interface.
# On non-Windows systems, you can give
# an explicit unit number, such as tun0.
# On Windows, use "dev-node" for this.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
dev tun1194

# SSL/TLS root certificate (ca), certificate
# (cert), and private key (key).  Each client
# and the server must have their own cert and
# key file.  The server and all clients will
# use the same ca file.
#
# See the "easy-rsa" directory for a series
# of scripts for generating RSA certificates
# and private keys.  Remember to use
# a unique Common Name for the server
# and each of the client certificates.
#
# Any X509 key management system can be used.
# OpenVPN can also use a PKCS #12 formatted key file
# (see "pkcs12" directive in man page).
ca keys/ca.crt
cert keys/server.crt
key keys/server.key  # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh1024.pem 1024
# Substitute 2048 for 1024 if you are using
# 2048 bit keys. 
dh keys/dh.pem

# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 10.8.0.0 255.255.255.0

# If enabled, this directive will configure
# all clients to redirect their default
# network gateway through the VPN, causing
# all IP traffic such as web browsing and
# and DNS lookups to go through the VPN
# (The OpenVPN server machine may need to NAT
# the TUN/TAP interface to the internet in
# order for this to work properly).
# CAVEAT: May break client's network config if
# client's local DHCP server packets get routed
# through the tunnel.  Solution: make sure
# client's local DHCP server is reachable via
# a more specific route than the default route
# of 0.0.0.0/0.0.0.0.
push "redirect-gateway def1 bypass-dhcp"

# The keepalive directive causes ping-like
# messages to be sent back and forth over
# the link so that each side knows when
# the other side has gone down.
# Ping every 10 seconds, assume that remote
# peer is down if no ping received during
# a 120 second time period.
keepalive 10 120

# Select a cryptographic cipher.
# This config item must be copied to
# the client config file as well.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-CBC

# Enable compression on the VPN link and push the
# option to the client (v2.4+ only, for earlier
# versions see below)
compress lz4-v2
push "compress lz4-v2"

# For compression compatible with older clients use comp-lzo
# If you enable it here, you must also
# enable it in the client config file.
;comp-lzo


# Notify the client that when the server restarts so it
# can automatically reconnect.
explicit-exit-notify 1


# The maximum number of concurrently connected
# clients we want to allow.
max-clients 10

# The persist options will try to avoid
# accessing certain resources on restart
# that may no longer be accessible because
# of the privilege downgrade.
persist-key
persist-tun

# Output a short status file showing
# current connections, truncated
# and rewritten every minute.
status /var/log/openvpn-status.log

# By default, log messages will go to the syslog (or
# on Windows, if running as a service, they will go to
# the "\Program Files\OpenVPN\log" directory).
# Use log or log-append to override this default.
# "log" will truncate the log file on OpenVPN startup,
# while "log-append" will append to it.  Use one
# or the other (but not both).
log-append  /var/log/openvpn.log

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 3

# Silence repeating messages.  At most 20
# sequential messages of the same message
# category will be output to the log.
;mute 20

# Misc
askpass pass.txt
</pre>

<p><br>Step 8: Have a text file named pass.txt under /etc/openvpn/, and have the passphrase you chose earlier on it. And then set the permission to 400 for security.</p>

<pre>
$ sudo echo {your passphrase} &gt; /etc/openvpn/pass.txt
$ sudo chmod 400 /etc/openvpn/pass.txt
</pre>

<p><br>Step 9: Enable a forwarding rule for your iptables firewall so that traffic on your 10.8.0.0 network used on your VPN connection gets routed through from the tun interface to the eth0 interface</p>

<pre>$ echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
$ sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE</pre>

<p><br>Step 10: Congratulations! It’s time to launch your first OpenVPN server on Amazon EC2!</p>

<pre>$ sudo service openvpn start</pre>
