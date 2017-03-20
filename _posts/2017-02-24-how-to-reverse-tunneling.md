---
layout: post
title:  How to Reverse Tunneling
date:   2017-02-24T14:00:00+8:00
---

Recently I've been asking to expose one of the services on our partner's machines through tunneling so our teams can access to it.

Tunneling is a common way to connect to a service through ssh if you have restricted network access (i.e. all porst are blocked by the firewall).

I've tried forward ssh tunnling but never reverse tunning before.

After some trial and error, here's my final command to run a combination of foward and reverse ssh tunning in order to bridge a connection through three machines (illustrated as follow).

![]({{site.baseurl}}/images/ssh-tunneling.png)

```bash
# reverse tunneling on the service machine in the private network.
ssh -f -N -R 6006:0.0.0.0:6006 pocuser@10.108.1.109 

# forward tunneling on the proxy server so the inernal service on service machine is exposed on the port 8006.
ssh -f -N -L 0.0.0.0:8006:0.0.0.0:6006 -p 19999 pocuser@localhost  

#     -f      Requests ssh to go to background just before command execution.
#             This is useful if ssh is going to ask for passwords or
#             passphrases, but the user wants it in the background.  This
#             implies -n.  The recommended way to start X11 programs at a
#             remote site is with something like ssh -f host xterm.
#
#     -L [bind_address:]port:host:hostport
#             Specifies that the given port on the local (client) host is to be
#             forwarded to the given host and port on the remote side.  This
#             works by allocating a socket to listen to port on the local side,
#             optionally bound to the specified bind_address.  Whenever a con-
#             nection is made to this port, the connection is forwarded over
#             the secure channel, and a connection is made to host port
#             hostport from the remote machine.  Port forwardings can also be
#             specified in the configuration file.  IPv6 addresses can be spec-
#             ified with an alternative syntax:
#             [bind_address/]port/host/hostport or by enclosing the address in
#             square brackets.  Only the superuser can forward privileged
#             ports.  By default, the local port is bound in accordance with
#             the GatewayPorts setting.  However, an explicit bind_address may
#             be used to bind the connection to a specific address.  The
#             bind_address of “localhost” indicates that the listening port be
#             bound for local use only, while an empty address or ‘*’ indicates
#             that the port should be available from all interfaces.
#
#     -N      Do not execute a remote command.  This is useful for just for-
#             warding ports (protocol version 2 only).
#
#     -R [bind_address:]port:host:hostport
#             Specifies that the given port on the remote (server) host is to
#             be forwarded to the given host and port on the local side.  This
#             works by allocating a socket to listen to port on the remote
#             side, and whenever a connection is made to this port, the connec-
#             tion is forwarded over the secure channel, and a connection is
#             made to host port hostport from the local machine.
#
#             Port forwardings can also be specified in the configuration file.
#             Privileged ports can be forwarded only when logging in as root on
#             the remote machine.  IPv6 addresses can be specified by enclosing
#             the address in square braces or using an alternative syntax:
#             [bind_address/]host/port/hostport.
#
#             By default, the listening socket on the server will be bound to
#             the loopback interface only.  This may be overriden by specifying
#             a bind_address.  An empty bind_address, or the address ‘*’, indi-
#             cates that the remote socket should listen on all interfaces.
#             Specifying a remote bind_address will only succeed if the
#             server’s GatewayPorts option is enabled (see sshd_config(5)).

```

To prevent the connections from breaking accidentally (E.g. machine restart, etc.), we can leverage `autossh` to restart the connection automatically. Please find the comprehensive tutorial [here](https://www.everythingcli.org/ssh-tunnelling-for-fun-and-profit-autossh/).
