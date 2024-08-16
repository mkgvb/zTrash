## zTrash Setup Guide

### Prerequisites
- **Machine**: Ensure the system can connect to Z and has VMware installed.

### Step 1: Create a Virtual Machine
- **Operating System**: Use Linux (Fedora 40 in this example).
- **Network Interfaces**:
  - `ens160`: BRIDGE interface.
  - `ens192`: NAT interface.

### Step 2: Initial Setup
1. **Disable NAT Interface**: After installation, turn off the NAT interface. It won't be needed until later due to potential issues with Z certificate and blocking almost every url.
2. **Install Dante SOCKS Server**:
   - Fedora: `dnf install danted`
   - Debian-based: `apt install sockd` (may also be `libsockd?`).

### Step 3: Configure Networking
1. **Enable NAT Interface**: Once setup is complete.
2. **Check Routing**:
   - Ensure the NAT interface is the default (lower metric).
   - Run `route -n` (Fedora) to verify:
     ```bash
     Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
     0.0.0.0         192.168.159.2   0.0.0.0         UG    100    0        0 ens192
     0.0.0.0         192.168.8.1     0.0.0.0         UG    199    0        0 ens160
     172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
     192.168.8.0     0.0.0.0         255.255.255.0   U     199    0        0 ens160
     192.168.159.0   0.0.0.0         255.255.255.0   U     100    0        0 ens192
     ```
   - If not correct: Temporarily turn off the BRIDGE interface and then turn it back on. You may need to adjust the metric for `ens160` (set it higher).

### Step 4: Configure Dante SOCKS Server
Edit `/etc/sockd.conf`:

    ```bash
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
    }
    ```

### Step 5: Enable and Start Dante SOCKS Server
    ```bash
    systemctl enable danted
    systemctl start danted
    ```

### Step 6: Firefox Proxy Setup
1. Go to Firefox settings and search for `Proxy`.
2. Configure the following:
   - **SOCKS HOST**: `<YOUR BRIDGE INTERFACE IP>`
   - **PORT**: `1080`
   - **NO PROXY**: `*discord.com,*discord.gg,*.discordapp.net,*discordapp.com,*spotify.com,*.spotify.com`
     - Add more domains as needed that you don't want proxied.

### Step 7: (Optional) Use BRIDGE Interface as Default Gateway with Routing Table
1. **Create and Configure a Dispatcher Script**:
   - Create the file `/etc/NetworkManager/dispatcher.d/10-danted-routing`.
   - Make it executable and add the following content:
     ```bash
     if [ "$1" == "ens192" ] && [ "$2" == "up" ]; then
         # Add the custom routing table
         echo "100 danted" >> /etc/iproute2/rt_tables
    
         # Add the policy rule
         ip rule add from 192.168.159.0/24 lookup danted
    
         # Add the route to the danted table
         ip route add default via 192.168.159.2 dev ens192 table danted
     fi
     ```

2. **Restart and Verify**:
   - After restarting, check that the `danted` routing table is active:
     ```bash
     ip route show table danted
     ```
   - You should see:
     ```bash
     default via 192.168.159.2 dev ens192
     ```
