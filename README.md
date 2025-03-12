## zTrash Setup Guide

### Prerequisites
- **Machine**: Ensure the system can connect to Z and has VMware installed.
- Mouse jiggler recommended, but probably not required
  - https://github.com/arkane-systems/mousejiggler use the portable version, which the developer obviously hates but is your only option due to IT treating you like a 4 year old

### Step 1: Create a Virtual Machine
- **Operating System**: Use Linux (Fedora 40 in this example).
- **Network Interfaces**:
  - `ens160`: BRIDGE interface.
  - `ens192`: NAT interface.

### Step 2: Initial Setup
1. **Disable NAT Interface**: After installation, turn off the NAT interface. It won't be needed until later due to potential issues with Z certificate and blocking almost every url.
2. **Install Dante SOCKS Server**:
   - Fedora: `dnf install dante-server`
   - Debian-based: `apt install sockd` (may also be `libsockd?` or `sockd-server`).

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
    internal: 0.0.0.0 port = 1080
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
    systemctl enable sockd
    systemctl start sockd
    ```

### Step 6: Firefox Proxy Setup
1. Go to Firefox settings and search for `Proxy`.
2. Configure the following:
   - **SOCKS HOST**: `<YOUR BRIDGE INTERFACE IP>`
   - **PORT**: `1080`
   - **NO PROXY**: `*discord.com,*discord.gg, *.discordapp.net, *discordapp.com, *spotify.com, *.spotify.com, *mkgvb.com, *.mkgvb.com,*discord*, *.googleapis.com`
     - Add more domains as needed that you don't want proxied.

### Step 7: (Optional) Use BRIDGE Interface as Default Gateway with Routing Table - needed for tailscale
0. Modifiy your routing table default gateway to always use your internet interface as a lower metric, im using 99
   Find your network interface guid with:
`nmcli connection show`
```
internet ens160  **525774e7-5840-3f4f-9bc1-9b41067c6784**  ethernet  ens160
z ens192    87031035-5c31-3902-9a05-2c2bf3d1edf4  ethernet  ens192
tailscale0       84df967e-b537-41bb-8108-2c8406a950f2  tun       tailscale0
lo               29114100-cadb-443f-9224-2da5444b366a  loopback  lo
docker0          6e68c769-ef86-49a0-b80b-4ee6dc0d4bf0  bridge    docker0
```

1. Run this command but change the guid to your internet interface `nmcli connection modify **525774e7-5840-3f4f-9bc1-9b41067c6784** ipv4.route-metric 99 `
Then bounce the interface and check its metric is less than the z interface with `route -n`
```
0.0.0.0         192.168.8.1     0.0.0.0         UG    99     0        0 ens160
0.0.0.0         192.168.159.2   0.0.0.0         UG    100    0        0 ens192
```


3. **Create and Configure a Dispatcher Script**:
   - Create the file `/etc/NetworkManager/dispatcher.d/10-danted-routing`.
   - Make it executable and add the following content:
     ```bash
     #!/bin/bash
     
     if [ "$1" == "ens192" ] && [ "$2" == "up" ]; then
         # Add the custom routing table
         echo "100 danted" >> /etc/iproute2/rt_tables
    
         # Add the policy rule
         ip rule add from 192.168.159.0/24 lookup danted
    
         # Add the route to the danted table
         ip route add default via 192.168.159.2 dev ens192 table danted
     fi
     ```

4. **Restart and Verify**:
   - After restarting, check that the `danted` routing table is active:
     ```bash
     ip route show table danted
     ```
   - You should see:
     ```bash
     default via 192.168.159.2 dev ens192
     ```
   - Harden sockd so it restarts if it fails on boot
     ```bash
     cat /usr/lib/systemd/system/sockd.service
      [Unit]
      Description=Dante SOCKS server
      After=syslog.target network.target network-online.target
      Documentation=man:sockd(8) man:sockd.conf(5)
      After=network-online.target
      Wants=network-online.target
      
      [Service]
      Type=forking
      #EnvironmentFile=/etc/sockd.conf
      PIDFile=/run/sockd.pid
      ExecStart=/usr/sbin/sockd -D -p /run/sockd.pid
      Restart=on-failure
      RestartSec=5s
      StartLimitBurst=3  # Maximum number of start attempts within the StartLimitIntervalSec
      StartLimitIntervalSec=60s  # Time window for counting retries
      
      [Install]
      WantedBy=multi-user.target
      ```
  ### Step 8: (Optional) Auto start the vm on windows login
  1. **Create a batch script and add it to shell::startup**
   ```batch
    @echo off
    REM Set the path to your vmrun executable (if necessary, otherwise skip this part)
    set vmrun_path="C:\Program Files (x86)\VMware\VMware Workstation\vmrun.exe"
    
    REM Set the path to your VM's .vmx file
    set vmx_path="C:\Users\USERNAME_REPLACE_ME\Documents\Virtual Machines\Fedora 64-bit\Fedora 64-bit.vmx"
    
    REM Run the vmrun command in a minimized window without the GUI
    start /min "" %vmrun_path% start %vmx_path% nogui
    
    REM Optional: Check if the VM was started successfully
    if %errorlevel% == 0 (
        echo %vmx_path% started successfully in the background without GUI.
    ) else (
        echo Failed to start the VM.
    )
    pause
   ```
### Step 9: (Optional) Make proxied applications into apps
- Windows
  - For example this is outlook. Make a file on your desktop named outlook.cmd. Stick this in it and double click when done
  ```batch
  @echo off
  start chrome --proxy-server=socks5://{PROXY_ADDRESS}:1080 -app=https://outlook.office365.us/mail/
  ```

     
