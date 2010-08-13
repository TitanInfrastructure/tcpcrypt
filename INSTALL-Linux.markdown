Installing tcpcrypt on Linux
============================

Tcpcrypt has 2 separate implementations: kernel and userland. These
instructions cover only the userland tcpcrypt, which is easier to set up.


Dependencies
============

 * OpenSSL >= 0.9.8
 * libnfnetlink >= 0.0.40
 * libnetfilter_queue >= 0.0.16
 * libcap

Ubuntu packages
---------------
apt-get install iptables libcap-dev libssl-dev \
                libnfnetlink-dev libnetfilter-queue-dev

{TODO: add packages for other distros}

Compiling
---------

In the tcpcrypt/user directory, run:

   make

Running `make install` will install `libtcpcrypt.so` in `/usr/lib`.


Try it out
----------

See the included README.markdown file for ways to try out tcpcrypt.

Troubleshooting
---------------

If it's not working, the most likely causes are the following.

   * There's a conflict with your existing iptables firewall rules. See the
     "Firewall setup" section below.

   * Your browser already had an open, non-tcpcrypted TCP connection to
     tcpcrypt.org before you ran the launch script. Quit and reopen your
     browser, wait 30 seconds, or use a different browser to retrieve the
     tcpcrypt.org URL.

Visit http://wiki.github.com/sorbo/tcpcrypt/troubleshooting if you're still
unable to make it work.


iptables firewall setup
=======================

The included launch_tcpcryptd.sh script adds iptable rules to divert Web and local port 7777 traffic to tcpcryptd. Read on only for more complex firewall setups.

The naive way to use tcpcryptd:

iptables -A OUTPUT -p tcp -j NFQUEUE --queue-num 666
iptables -A INPUT -p tcp -j NFQUEUE --queue-num 666

This will apply tcpcrypt to all locally destined (or generated) TCP packets.
This will work, but you'll run into problems #1 and #2, which may not be
problems if you don't have a firewall or nat setup.

For testing on your local machine, you can restrict tcpcrypt to the loopback interface:

iptables -A OUTPUT -p tcp -o lo -j NFQUEUE --queue-num 666
iptables -A INPUT -p tcp -i lo -j NFQUEUE --queue-num 666

Linux firewall setup is more challenging than on FreeBSD for two reasons.

   1. In FreeBSD, after a packet is diverted, the divert daemon can drop the
      packet, or accept it. In the latter case, firewall processing continues
      from the next rule. So basically natd will get a chance to run, and other
      firewall rules. It's a pipeline. On Linux, you can either accept or drop
      the packet, which ignores the rest of the firewall.

   2. In FreeBSD, you can easily order tcpcryptd, then natd, because they're
      both in userland, and both use divert, and the whole firewall is a
      pipeline. On Linux natd is IP connection tracking in the kernel, which is
      used for stateful firewalls too. We gotta make tcpcryptd run BEFORE
      conntrack.

To make tcpcrypt work the "proper" way, making sure that nat and stateful
firewalls (e.g., -m state --state ESTABLISHED) work:

iptables -t raw -A PREROUTING -p tcp -j NFQUEUE --queue-num 666
iptables -t mangle -A POSTROUTING -p tcp -j NFQUEUE --queue-num 666

This will apply tcpcrypt to all TCP packets entering and exiting the box,
including forwarded packets.  Note that this setup will respect firewall
rules in other tables but terminate those in the raw and mangle tables.  In
short, your firewall rules in the filter table and nat table (those that you
probably care about most) will work.  You'll get caught by problem #1 though.

To make tcpcrypt work the elite way, making sure that all firewall rules are
obeyed and conntrack isn't confused:

iptables -t raw -N tcpcrypt
iptables -t raw -A tcpcrypt -p tcp -m mark --mark 0x0/0x10 -j NFQUEUE --queue-num 666
iptables -t raw -I PREROUTING -j tcpcrypt

iptables -t mangle -N tcpcrypt
iptables -t mangle -A tcpcrypt -p tcp -m mark --mark 0x0/0x10 -j NFQUEUE --queue-num 666
iptables -t mangle -I POSTROUTING -j tcpcrypt
 
And launch tcpcryptd with -x 0x10

This example is like before, but will create a chain with only the tcpcrypt
rule, which will run only if a packet is unmarked.  When tcpcryptd needs to
accept a packet, rather than passing a verdict of ACCEPT, which terminates
all rule processing, it will pass a verdict of REPEAT, which restarts
processing at the current chain.  To avoid loops, it will also mark the
packet so that the rule to divert will be matched only once.  Effectively the
first time round real work will be done, and the second time round we
"return" to process the other rules.

Note that you can make tcpcryptd work transparently on forwarded traffic, and
even in conjunction with NAT.  You can pretend that the Internet is
tcpcrypted.  Lets say eth0 is your LAN.  You can do something like:

[create the tcpcrypt chains as explained earlier.]

iptables -t raw -A PREROUTING -i eth0 -j tcpcrypt
iptables -t mangle -A POSTROUTING -o eth0 -j tcpcrypt

tcpcryptd will see all incoming traffic from eth0 and make it look like
standard TCP to the outside world, and will then tcpcrypt all the responses
coming back to eth0.  There's one caveat though when using it in conjunction
with NAT (conntrack).  tcpcryptd forges a packet (the INIT2) and this
confuses conntrack as it thinks it's a new connection and it changes the
source port.  You therefore need to add:

iptables -t raw -A OUTPUT -o eth0 -j NOTRACK

i.e., all locally generated traffic (the forged packet from tcpcryptd) should
not be natted.  In fact I don't even know why it is being natted (maybe a
bug).  Of course you need to setup nat with something like:

iptables -t nat -A POSTROUTING -o eth1 -j SNAT --to-source 1.2.3.4

where eth1 is your Internet interface and 1.2.3.4 your Internet static IP.