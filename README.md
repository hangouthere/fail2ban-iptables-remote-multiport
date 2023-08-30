# fail2ban - Remote IPTables

This small configuration is my approach for the case where you have `fail2ban` running on a host that *isn't* the gateway/proxy for your network, or your services are distributed across multiple hosts across a localized network. When an IP is banned, by default it is banned *only* on the service host. However, you may want the gateway/proxy to actually block on behalf of the entire network to avoid further intrusions allowable via services across the network.

### A note about this repo

In my particular use-case, I am running the [`linuxserver/swag` Docker image](https://hub.docker.com/r/linuxserver/swag) on my network that is routed and switched by a [Netgear R8000](https://www.amazon.com/NETGEAR-Nighthawk-X6S-Smart-Router-R8000P/dp/B01H53WZ20), running [FreshTomato](https://freshtomato.org/).

I'll cover specific configuration for those later, and this setup should work for all instances of `fail2ban` with `openssh-client` installed, but I wanted to make sure anyone reading this understood this information up-front.

## Pre-requisites

As mentioned, `openssh-client` must be installed in order to execute on the remote host, as well as an ssh key generated for secure fingerprint authentication.

### Setting up an SSH Key

I suggest generating a new key specifically for this use/user as you don't want this getting out:

```
ssh-keygen -q -t rsa -N '' -f ~/.ssh/ident_fail2ban
```

Now copy the contents of your new `ident_fail2ban` Identity to your gateway/proxy's `authorized_keys`:

```
ssh-copy-id iptablesUser@proxyHost -i ~/.ssh/ident_fail2ban
```

or, alternatively (older distros/alternate versions of ssh):
```
ssh iptablesUser@proxyHost "echo \"`cat ~/.ssh/ident_fail2ban.pub`\" >> .ssh/authorized_keys"
```

## Overview

As you can see, [`action.d/iptables-remote-multiport.conf`](./action.d/iptables-remote-multiport.conf) is very short and concise...
The whole point is to simply change the `iptables` command to one that prefixes with the proper `ssh` arguments.

By design, this will block locally as per standard `fail2ban` action, as well as remotely on a gateway/proxy system. 
I didn't want to fully rely on the remote system to be available (or configured properly, etc), such that at *minimum* it should also block locally.

Simply place it in your `action.d` folder and configure it within a Jail.

## Configuration

### Jails

Because actions can be applied per `jail`, we can configure them to include this custom action:

```
# Note: Use [DEFAULT] to apply globally

[some_jail]
action = %(action_)s
         iptables-remote-multiport
```

There's a good chance you'll need to also provide some parameters along with the action:

Example:

```
[some_jail]
action = iptables-remote-multiport[remote_proxy_user=fail2ban]
```

| Param             | Default                 | Description                                                                                                                                                               |
| ----------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| port              | 22                      | Defaults to the action this extends. Set to the port(s) you want to block  on once detected.                                                                              |
| remote_proxy_user | root                    | The user to ssh into the remote proxy (i.e., `remote_proxy_user@remote_proxy_host`).                                                                                      |
| remote_proxy_host | `192.168.1.1`           | The remote proxy host name/IP to ssh into (i.e., `remote_proxy_user@remote_proxy_host`).<br />This should include the port if the remote is running on an alternate port. |
| identity_file     | `~/.ssh/ident_fail2ban` | The local path to find the SSH Identity to use for remoting into the proxy.                                                                                               |

### Docker Container

Out of the box, the [`linuxserver/swag` image](https://hub.docker.com/r/linuxserver/swag) does not have `openssh-client` installed, so we have to tell it to set it up on run. Luckily, the `linuxserver` team already planned that one out, so you merely have to copy the contents of `lsio` to the root of your volume mount (i.e., `/config/custom-cont-init.d/`). This contains a simple script to install the client when the container is first ran.

To add to that, I chose to store my keys in `/config/keys/ssh/ident_fail2ban[.pub]`, so my configuration sets the `identity_file` parameter like so:

```
# jail.local

[DEFAULT]
action = %(action_)s
         iptables-remote-multiport[identity_file=/config/keys/ssh/ident_fail2ban,port="http,https"]
```

### FreshTomato Routing Software

As mentioned, I'm using [FreshTomato](https://freshtomato.org/), which is an amazing open source routing software that supports many popular routers on the consumer market.

I run my network on a CIDR of `192.168.1.0/24`, and my router is of course `.1`.

Since `iptables` is basically one of the fundamental commands behind how FreshTomato operates, setup is pretty much relegated to configuring SSH:

1. Ensure SSH is Enabled at Startup (usually on by default)
2. Paste the contents of `~/.ssh/ident_fail2ban.pub` into the **Authorized Keys** input, as a single line (it may word-wrap)
3. Disable **Limit Connection Attempts** to avoid triggering FreshTomato's ssh tarpit, and banning your own `fail2ban` instance!
4. (Optional) Ensure Remote SSH via IP Address access is disabled

See the [FreshTomato Setup](./ft_setup.png) UI for a better understanding.

## Testing / Operating

Once fully configured, you should be able to trigger using a fake entry into one of your configured Jails.

In this example, we'll trigger `nginx-http-auth` by adding the following to the end of our `error.log` for `nginx`:

```
2021/06/22 00:00:01 [error] 2865#0: *66647 user "xyz": password mismatch, client: 33.33.33.33, server: www.myhost.com, request: "GET / HTTP/1.1", host: "www.myhost.com"
2021/06/22 00:00:02 [error] 2865#0: *66647 user "xyz": password mismatch, client: 33.33.33.33, server: www.myhost.com, request: "GET / HTTP/1.1", host: "www.myhost.com"
2021/06/22 00:00:03 [error] 2865#0: *66647 user "xyz": password mismatch, client: 33.33.33.33, server: www.myhost.com, request: "GET / HTTP/1.1", host: "www.myhost.com"
2021/06/22 00:00:04 [error] 2865#0: *66647 user "xyz": password mismatch, client: 33.33.33.33, server: www.myhost.com, request: "GET / HTTP/1.1", host: "www.myhost.com"
2021/06/22 00:00:05 [error] 2865#0: *66647 user "xyz": password mismatch, client: 33.33.33.33, server: www.myhost.com, request: "GET / HTTP/1.1", host: "www.myhost.com"
```

> Note: You will need to change the time from `00:00:0*` to something near-recent (within seconds) for this test to work, or `fail2ban` will consider the entry too old!
 
On the host where `fail2ban` is running, you should see:

`fail2ban.log`
```
...
2021-06-22 00:00:05,810 fail2ban.actions        [466]: NOTICE  [nginx-http-auth] Ban 33.33.33.33
```

`fail2ban-client status nginx-http-auth`
```
Status for the jail: nginx-http-auth
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- File list:        /config/log/nginx/error.log
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   33.33.33.33
```

`iptables -L`
```
...
Chain f2b-nginx-http-auth (1 references)
target     prot opt source               destination
REJECT     all  --  33.33.33.33          anywhere             reject-with icmp-port-unreachable
RETURN     all  --  anywhere             anywhere
...
```

And on the remote gateway/proxy, we can see the `iptables` has propagated there as well:

`ssh root@192.168.1.1 -i ~/.ssh/ident_fail2ban iptables -L`
```
...
Chain f2b-nginx-http-auth (1 references)
target     prot opt source               destination
REJECT     all  --  33.33.33.33          anywhere             reject-with icmp-port-unreachable
RETURN     all  --  anywhere             anywhere
...
```

And of course, unbanning works as expected!

## Troubleshooting

For the most part, take a look at `fail2ban.log` and see what it says the error might be. Usually it's an auth problem so double check your key location, as well as that it's configured properly on the remote host as well. If need-be, attempt a manual ssh login using the identity to ensure it's properly configured.

Otherwise, the only issue I've come across in my short testing is doubling up of IP's in a Jail's chain. To resolve this, simply flush the chains on any/all of the systems, and restart `fail2ban`:

```
iptables --flush f2b-nginx-http-auth
ssh root@192.168.1.1 -i ~/.ssh/ident_fail2ban iptables --flush f2b-nginx-http-auth
restart_fail2ban #However you restart it in your env 
```