# eSO-hostable containerized FreeRADIUS filtering proxy for SP-only eduroam deployments
This is a containerized eduroam filtering proxy using FreeRADIUS, pre-configured and tailored for SP-only deployments in the US. The idea is to enable wireless gear to play nicely with the eduroam-US service even if the AP/controller can't make simple policy filtering decisions like dropping realm-less auth requests, which will always fail when they hit the national proxies, since they aren't routable.

We advise deploying one proxy container (two if you have redundant hosting capability) per constituent so any roaming stats delivered by the NRO aren't aggregated together with other constituents (use different ports for each constituent). Very small environments may wish to run this as a container on a SOHO-grade NAS appliance, AP/router combo, etc.

## Table of Contents

- [Setup](#setup)
  - [eduroam registration](#eduroam-registration)
  - [Container hosting](#container-hosting)
  - [Connecting it all](#connecting-it-all)
- [Building and running the container](#building-and-running-the-container)
  - [Environmental variables](#environtmental-variables)
  - [Logging](#logging)
- [Building and updating](#building-and-updating)
  - [Security updates and considerations](#security-updates-and-considerations)

## OK, but why?
One of our goals as an eSO is to remove as many barriers as possible for community partners to add eduroam to their public wi-fi offerings, even in relatively simple wi-fi deployments. The main drivers behind this stripped-down fork of [eso-tools](https://github.com/nshe-scs/public-eso-tools) are:
- It's hard to follow eduroam best practices when your wireless hardware vendor sells/rents products with 802.1x support, but forces you to buy additional products for fairly basic policy decisions like `if 'RADIUS User-Name' contains '@', then: forward to 'eduroam_proxies', else: reject`.
- Let's make it easy for an eSO or an SP-only constituent to point RADIUS/802.1x auth at this container, and get basic eduroam-compatible username filtering and proxy rate limiting to reduce auth waste. Because logging is built in, an eSO hosting this proxy can help a constituent meet eduroam logging requirements; see [Logging](#logging) below.
- Contribute our work back to the broader education community. While we're focused on spurring eduroam adoption in our neck of the woods, this is still broadly applicable (just not fine-tuned) to larger organizations and other countries.

# Setup
## eduroam registration
We assume you're a US-based [eduroam Support Organization (eSO)](https://incommon.org/eduroam/eduroam-k12-libraries-museums/), but maybe you're a municipality or community center wanting to add eduroam as a secure wi-fi option to support your local education community... In any case, you must be able to register an SP in the eduroam infrastructure. US-based organizations do this via Internet2's Federation Manager tool. If you don't know what that is, reach out to Internet2 or your state's eSO. 

## Container hosting
You'll need to be able to host a container, ideally two on separate hardware and/or locations for redundancy.

If you're an eSO, we suggest preparing a single DNS name like `proxy.roam.(your-eso-domain)`, e.g. `proxy.roam.example.org`, with _one IP per redundant hosting location_ and _one unique port number per constituent_. By using unique proxy containers for each constituent, you ensure their roaming stats aren't co-mingled or aggregated together. You also give yourself operational flexibility and reduce the risk of impacting multiple constituents if you need to test or change something down the road.

Redundant containers for the same constituent can share the same hostname; the combination of IP, port, and secret, not the hostname, uniquely identifies your proxy container to the eduroam-US proxies.

## Connecting it all
Once the proxy container is live, have your constituent point their wireless gear at the proxy container IP + port with the secret you assigned them.

A successful auth will flow as an encrypted EAP conversation from visitor device -> constituent wi-fi (SP) -> your proxy -> FLR (eduroam proxies) -> visitor's school/college (IdP) -> (back).

Malformed auth requests from eduroam visitors, i.e. non-EAP auths and realm-less usernames, will be dropped quickly, saving traffic from your proxy -> FLR -> (back).

Repeated auth requests are rate-limited via FreeRADIUS's `proxy_rate_limit` module, further mitigating aggressive device behavior. This is often the client OS/firmware's fault, not intentional behavior from your visitors.

# Building and running the container
Build and run it as you typically would with the supplied `Dockerfile` and/or `docker-compose.yml`. We tested with `docker-compose` for convenience, but you could use other methods.

## Environmental variables
However you choose to run it, whether manually or via an automation/orchestration tool, the container configuration is completely driven by environmental variables. The entrypoint script will put the important details in the correct places in the various freeradius config files during container startup, and link the correct sites/mods/etc. based on your needs. We recommend setting these variables via the supplied `custom.env` file. You'll need one per container.

`FR_MY_FQDN`: What the proxy should believe is its fully qualified hostname. If you plan to host proxies for multiple constituents, we recommend `(constituent).proxy.roam.example.org` if you registered `proxy.roam.example.org` in DNS (don't register unique records for each container).

`FR_FLR_SECRET`: RADIUS shared secret you configured in Federation Manager for this proxy, e.g. a random 63-character mixed-case alphanumeric string that is NOT the same as `FR_WAP_SECRET`. A quick method to generate one: `echo $(tr -dc 'A-Za-z0-9' </dev/urandom | head -c63)`.

`FR_FLR_IP_1`: IP address (or FQDN, only checked at startup) of the first federation level RADIUS proxy, e.g. `163.253.30.2` or `tlrs2.eduroam.us` (west coast).

`FR_FLR_IP_2`: IP address (or FQDN, only checked at startup) of the second federation level RADIUS proxy, e.g. `163.253.31.2` for `tlrs1.eduroam.us` (east coast).

`FR_WAP_SECRET`: RADIUS shared secret configured on the wireless APs / controller to talk to this container, e.g. a random 63-character mixed-case alphanumeric string that is NOT the same as `FR_FLR_REALM`.

`FR_WAP_IP`: Comma-delimited list of IP addresses or CIDR subnets of the constituent's wireless APs / controller that will send authentication requests to this container, e.g. `192.168.0.1,192.168.1.1/24`. These are the only IPs the container will treat as legitimate RADIUS clients. Be aware of whether NAT is at play; your constituent may not realize that their `192.168.0.1` is really your `1.2.3.4`.

Omit the following if your wireless gear doesn't use/support VLANs or if it can assign VLANs without help from a RADIUS server:

`FR_VLAN_VISITORS`: VLAN number to assign to eduroam visitors when authentication succeeds.

## Logging
Anyone operating an eduroam IdP or SP is required to keep sufficient logs for troubleshooting. FreeRADIUS, particularly in debug mode, can be _very_ chatty, and this is a good thing. But you probably don't have infinite space for logs, so we provide two options that are less verbose than debug level console output, but still very useful for investigating and troubleshooting. We also include a `Correlation-ID` field in these log entries to help you grep for individual EAP conversations.

**Remember, this container only handles authentication, so the constituent is still on the hook for logging which IP their DHCP server assigned to a given MAC at a given time on their network.**

`FR_LOG_DESTINATION`: either `syslog` (recommended) or `file`.

### Remote syslog (recommended)
If you have a remote syslog host or SIEM, take advantage of its collection, searching, and compression capabilities. The following must be set:

`FR_LOG_SYSLOG_HOST`: the FQDN or IP of your syslog host.

`FR_LOG_SYSLOG_PORT`: the port number your syslog host is listening on.

`FR_LOG_SYSLOG_PROTO`: either `tcp` (recommended) or `udp`.

`FR_LOG_SYSLOG_FAC`: the syslog facility name FreeRADIUS logs should use. We recommend `local1` but this is arbitrary.

`FR_LOG_SYSLOG_SEV`: the syslog severity level FreeRADIUS logs should use. We recommend `notice` but this is arbitrary.

### File based logging with persistent storage
If you don't specify the syslog details above, we assume you don't have a syslog receiver. In that case, the container will log to a file instead: `/var/log/freeradius/eduroam.log`. You must persistently map / bind this file to a volume using your container host's recommended method. It's up to you to rotate and trim the log with a tool of your choice from your container host; the container won't do it for you.

# Building and updating
We use the latest official Alpine-based FreeRADIUS container, add the syslog-ng package, and copy our eduroam-US friendly config files over at container build time. We then apply actual configuration details during run time. Thus, you can destroy and rebuild the container every day and it will work the same way every time, as long as your config is the same (e.g. the set of environmental variables you feed your automation/orchestration tools). 

At container startup, `entrypoint.sh` will update the relevant FreeRADIUS conf files based on the environmental variables you made available to the container. It will also split your comma-delimited `FR_WAP_IP` into individual `client.d/*.conf` files for FreeRADIUS.

If you want to add/modify/delete clients once the container is running, you'll need to update the container's environment, e.g. via `custom.env`, and restart the container. Don't forget to update firewall rules if applicable.

## Security updates and considerations
Security updates take the form of pulling and rebuilding the container image, thereby starting from scratch with the latest patched Alpine + FreeRADIUS; destroying the currently-running container; and starting the new one.

This is a RADIUS UDP proxy, no TCP/TLS/RadSec support at this time. While eduroam authentication is protected by the use of EAP, you should control access to your proxy by configuring your firewall(s) to allow only the constituent's wireless gear and the upstream eduroam proxies to communicate over your chosen port.

You can read [more about eduroam security here](https://eduroam.org/eduroam-security/).
