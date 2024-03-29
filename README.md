# OpenVPN-Server
<p>Step-by-step guide to setting up an OpenVPN server on Amazon AWS EC2 T4g using Ubuntu 22.04 arm64.</p>

<p>Since you're looking for instructions on how to build your own OpenVPN server, I'll assume you have an Amazon AWS account and some experience with it, so I can skip the part about creating an EC2 instance.</p>

<h2><br>Installing and configuring your OpenVPN server</h2>

<p><br>Step 1: ssh into the EC2 instance you set up just now</p>

<pre>$ ssh -i ovpn.pem ubuntu@ec2-1-2-3-4.region.compute.amazonaws.com</pre>

<p><br>Step 2: Install openvpn and easy-rsa</p>

<pre>
$ sudo apt install openvpn easy-rsa
</pre>

<p><br>Step 3: Set Certificate Authority (CA) and generate certificates and keys for an OpenVPN server and clients</p>

<pre>
$ mkdir ~/easy-rsa
$ ln -s /usr/share/easy-rsa/* ~/easy-rsa/
$ cd ~/easy-rsa
$ ./easyrsa init-pki
$ ./easyrsa build-ca
$ ./easyrsa build-server-full server   ### for openvpn server 
$ ./easyrsa build-client-full client   ### for openvpn client
$ ./easyrsa gen-dh
</pre>

<p>Note: You will be asked to set one passphrase for each certificate and key. Choose one you like and you will need it later.</p>

<p><br>Step 4: Have certificates and keys in place for OpenVPN server</p>

<pre>$ cd /etc/openvpn/
$ sudo mkdir keys
$ cd keys
$ sudo cp ~/easy-rsa/pki/ca.crt .
$ sudo cp ~/easy-rsa/pki/issued/server.crt .
$ sudo cp ~/easy-rsa/pki/dh.pem .
$ sudo cp ~/easy-rsa/pki/private/server.key .
</pre>

<p><br>Step 5: Have an OpenVPN server config ovpn.conf ready under /etc/openvpn/. Following is mine as an example.</p>

<pre>
# Which TCP/UDP port should OpenVPN listen on?
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

# Pushing DNS to clients
push "dhcp-option DNS 1.1.1.1"  #Cloudflare
push "dhcp-option DNS 1.0.0.1"  #Cloudflare

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
#cipher AES-256-CBC

# Enable compression on the VPN link and push the
# option to the client (v2.4+ only, for earlier
# versions see below)
;compress lz4-v2
;push "compress lz4-v2"

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

# It's a good idea to reduce the OpenVPN
# daemon's privileges after initialization.
#
# You can uncomment this out on
# non-Windows systems.
;user nobody
;group nobody

# CRL-VERIFY - for revoking users
;crl-verify keys/crl.pem
</pre>

<p><br>Step 6: Have a text file named pass.txt under /etc/openvpn/, and have the passphrase you chose earlier on it. And then set the permission to 400 for security.</p>

<pre>
$ sudo echo {your passphrase} &gt; /etc/openvpn/pass.txt
$ sudo chmod 400 /etc/openvpn/pass.txt
</pre>

<p><br>Step 7: Firewall & Routing Configurations</p>

<pre>$ (ip -4 route ls | grep default | grep -Po '(?<=dev )(\S+)' | head -1)</pre> 
My output is `ens5`, which is the public network interface of my EC2.
<pre>sudo vim /etc/ufw/before.rules</pre>
Adding the following lines before `# Don't delete these required lines, otherwise there will be errors`
<pre># START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to ens5. Change ens5 to your interface here. Mine is ens5.
-A POSTROUTING -s 10.8.0.0/8 -o ens5 -j MASQUERADE
COMMIT
# END OPENVPN RULES</pre>
<pre>$ sudo vim /etc/default/ufw</pre>
Change the value of `DEFAULT_FORWARD_POLICY` from `DROP` to `ACCEPT`.
<pre>DEFAULT_FORWARD_POLICY="ACCEPT"</pre>
<pre>$ sudo ufw disable;sudo ufw enable</pre>

<p><br>Step 8: Congratulations! It’s time to launch your first OpenVPN server on Amazon EC2!</p>

<pre>$ sudo service openvpn start</pre>

<h2><br>How to revoke an access to the VPN service?</h2>
<p><br>Step 1: Navigate to ~/easy-rsa</p>
<pre>$ cd ~/easy-rsa</pre>
<p><br>Step 2: Revoke the access (Let's assume I want to revoke the access of a client that I previously granted</p>
<pre>$ ./easyrsa revoke client</pre>
<pre>Type '<b>yes</b>' manually when you see the following message: Type the word 'yes' to continue, or any other input to abort. Continue with revocation:</pre>
<pre>Enter passphrase you set for your ca.key</pre>
<p><br>Step 3: Run gen-crl</p>
<pre>$ ./easyrsa gen-crl</pre>
<p><br>Step 4: Upload a CRL (if you have done it before you can skip the step)</p>
<pre>$ cd /etc/openvpn/keys/</pre>
<pre>$ sudo ln -s ~/easy-rsa/pki/crl.pem crl.pem</pre>
<p><br>Step 5: Update your ovpn.conf and uncomment ;crl-verify keys/crl.pem  (if you have done it before you can skip the step)</p>
<pre>;crl-verify keys/crl.pem ==> crl-verify keys/crl.pem</pre>
<p><br>Step 6: Restart OpenVPN server</p>
<pre>sudo service openvpn restart</pre>
<p><br>Step 7: Now the ios device has lost the access to the VPN service!</p>
