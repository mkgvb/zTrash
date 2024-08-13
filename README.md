## zTrash


0. Find a company brick capable of connecting to Z and has vmware
1. Create a VM, Linux, I used fedora 40
2. Setup 2 interfaces one NAT, one BRIDGE
    - ens160 is my BRIDGE
    - ens192 is my NAT
3. Once booted/installed, Turn off NAT interface, its useless until you get everything setup as Z spyware is trash, nothing will install.
4. Install danted via apt/dnf (fedora) I believe in the Debian pkg its called sockd (possibly libsockd idgaf). Dont mix this up with dante, this is the client, you want the daemon(server)
5. Turn back on NAT interface

Check your routes make sure your NAT interface is default (metric lower).
`route -n` (fedora, ubuntu good luck) should look like this
```
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.159.2   0.0.0.0         UG    100    0        0 ens192
0.0.0.0         192.168.8.1     0.0.0.0         UG    199    0        0 ens160
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.8.0     0.0.0.0         255.255.255.0   U     199    0        0 ens160
192.168.159.0   0.0.0.0         255.255.255.0   U     100    0        0 ens192
```
- If its not Turn off bridge for a second and then turn it back on usually gets it right, I set my metric to higher always for ens160, cant remember how.


Here is my `/etc/sockd.conf`
```
# Logging configuration
logoutput: syslog stdout /var/log/sockd.log

# Server address specification
internal: ens160 port = 1080
external: ens192

#external.rotation: route

# Method to use for outgoing connections
socksmethod: none


user.privileged: root
user.unprivileged: nobody


# Client rules - specify which clients are allowed to connect
client pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect disconnect error
}

# Server rules - specify which destinations are allowed
socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: connect disconnect error
```

6.  enable and start danted(sockd)
    - `systemctl enable danted`
    - `systemctl start danted`

7.  Firefox setup
    - goto settings, search proxy
    - SOCKS HOST: <YOUR BRIDGE INTERACE IP>
    - PORT: 1080
    - NO PROXY. You might want to add stuff here that Z will block. Here is my list so far
*discord.com,*discord.gg, *.discordapp.net, *discordapp.com, *spotify.com, *.spotify.com

### Addendum
It would seem that danted is always uses the routing table of the host, regardless of how you set external in /etc/sockd.conf. Which sucks.
This means that your default route is the NAT interface which means that the VM cant get to the internet due to Z.
SSH via the BRIDGE interface will work.
Tailscale will not work due to the routing.
