# Module 6: Zero Touch Provisioning (ZTP)

In this module, you will verify and confirm the prerequisites for ZTP, the Zero Touch Provisioning feature of IOS XE on the Catalyst 9300 switch. At the end of this module, you will issue the 'write erase' command, reload the switch, and watch as the ZTP process completes, and the switch is configured programmatically and automatically.

What is ZTP? When a device that supports Zero-Touch Provisioning boots up, and does not find the startup configuration (during initial installation), the device enters the Zero-Touch Provisioning mode. The device searches for an IP from a DHCP server and bootstraps itself by enabling the Guest Shell. The device then obtains the IP address or URL of an HTTP/TFTP server, and downloads a Python script from a server to configure the device.



## *Step 1:*

**ZTP Python File:** Review the ztp-simple.py file on the Ubuntu VM which is located at /var/www/html. This file uses the Python API to set the interface IP address, configure credentials and enables access to the device over the programmatic interfaces, as well as to configure some additional device features. From the Windows Jump Host desktop, onen a SSH session to the **Ubuntu** server and review the ztp-simple.py script:

```
auto@automation:~$ less /var/www/html/ztp-simple.py
```

**Note** : The Python script with the POD environment may differ slightly

```
# Importing cli module
import cli

print("Configure vlan interface, gateway, aaa, and enable netconf-yang\n\n")
cli.configurep(["int gi1/0/24","no switchport", "ip address 10.1.1.5 255.255.255.0", "no shut", "end"])
cli.configurep(["ip route 0.0.0.0 0.0.0.0 10.1.1.3", "end"])
cli.configurep(["int vlan 1", "no ip address", "end"])
cli.configurep(["username admin privilege 15 secret 0 Cisco123"])
cli.configurep(["ip tftp blocksize 8192", "end"])
cli.configurep(["ip tcp window-size 65535", "end"])
cli.configurep(["ip http client source-interface GigabitEthernet1/0/24", "end"])
cli.configurep(["interface Loopback0", "ip address 192.168.12.1 255.255.255.0", "end"])
cli.configurep(["aaa new-model", "aaa authentication login default local", "end"])
cli.configurep(["aaa authorization exec default local", "aaa session-id common", "end"])
cli.configurep(["netconf-yang", "end"])
#cli.configurep(["restconf", "end"])
cli.configurep(["gnxi", "gnxi secure-init", "gnxi secure-server", "end"])
cli.configurep(["line vty 0 32", "transport input all", "exec-timeout 0 0", "no ip domain lookup", "end"])
cli.configurep(["ip scp server enable", "end"])
cli.configurep(["hostname C9300", "ntp server 10.1.1.3", "clock timezone Pacific -7", "end"])
cli.configurep(["telemetry ietf subscription 101","encoding encode-kvgpb","filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds","stream yang-push","update-policy periodic 30000","receiver ip address 10.1.1.3 57500 protocol grpc-tcp","end"])
#cli.configurep(["iox", "end"])

print("\n\n *** Executing show ip interface brief  *** \n\n")
cli_command = "sh ip int brief | exclude unassigned"

cli.executep(cli_command)
print("\n\n *** ZTP Day0 Python Script Execution Complete *** \n\n")

cli.configurep(["int gi1/0/24","no switchport", "ip address 10.1.1.5 255.255.255.0", "no shut", "end"])
cli.configurep(["ip route 0.0.0.0 0.0.0.0 10.1.1.3", "end"])
cli.configurep(["int vlan 1", "no ip address", "end"])

print("\n\n *** Executing show ip interface brief  *** \n\n")
cli_command = "sh ip int brief | exclude unassigned"

cli.executep(cli_command)
print("\n\n *** ZTP Day0 Python Script Execution Complete *** \n\n")
```

press ‘q’ to quit.

## *Step 2:*

**IP:** The IP address on the Ubuntu VM is is 10.1.1.3 and can be confirmed with the following commands. It is important to know the IP as it is used for setting the DHCP option 67 - this is how the IOS XE devices know where to find the Python file to execute.

Check the interface IP assignments:

```
auto@automation:~$ ip a
auto@automation:~$ ip a | grep 10.1.1.3
```

<img src="imgs/ipa.png" style="zoom:67%;" />



## Step 3:

**DHCP Server** : ZTP works when the DHCP server replies to the IOS XE device with DHCP option 67 in the DHCP Response. The DHCP server's configuration file is called dhcpd.conf and it is located in the /etc/dhcp/ folder. It specifies the IP range to serve DHCP leases to, as well as the Python file that the device will download and executed as part of the option bootfile-name which is also known as option 67.

Examine the DHCP server configuration:

```
auto@automation:~$ cat /etc/dhcp/dhcpd.conf
```

<img src="imgs/dhcpd.png" style="zoom:65%;" />

Check the status of the DHCP service to ensure it is running correctly

```
auto@automation:~$ sudo /etc/init.d/isc-dhcp-server status
```

<img src="imgs/dhcpdstatus.png" style="zoom:70%;" />



If any changes are made to the configure file or if it is not running it can be restarted with the commands below:

```
auto@automation:~$ sudo /etc/init.d/isc-dhcp-server restart
```

## *Step 4:*

**Webserver** : **NGINX** is open source software for web serving, reverse proxying, caching, load balancing, media streaming, and more. It started out as a web server designed for maximum performance and stability. The nginx webserver is used to serve the Python file to the IOS XE device as part of the ZTP process

Check the status of the nginx webserver to ensure it is running:

```
auto@automation:~$ sudo /etc/init.d/nginx status
```

<img src="imgs/nginxstatus.png" style="zoom:75%;" />

If any changes are required to the nginx webserver configuration file, or if the service needs to be restart then run the following commands:

```
auto@automation:~$ sudo /etc/init.d/nginx restart
```

## *Step 5:*

**IOS XE Device:** Now the prerequisites for ZTP are met and the device is ready to be reloaded once the previous configuration is removed – this is to ensure that the Day0 ZTP process is initialized once the switch boots. This emulates a new, un-configured device that is ready to be provisioned.

Connect to the Serial Console of the C9300 using the **~/console-helper.sh** script

 Once connected to the serial console the next step is to erase the configuration and reload the device

This process will take about 5 minutes to successfully complete. Once completed, ICMP pings from the device will begin responding.

```
C9300# wr
C9300# wr erase
C9300# reload
```

If prompted to save the configuration, enter no

Press enter to confirm reloading

<img src="imgs/consolehelper.png" style="zoom:75%;" />

The device will reload and once IOS XE boots up, the ZTP process will start. The device will obtain the ztp.py configuration file from the Ubuntu server that it receives in the DHCP response.

During this time, the DHCP and HTTP server logs can be followed, and progress can be tracked as the device boots completes the ZTP process. In this case, we see first the DHCP server providing the DHCPOFFER to the device, and next the GET requests to the HTTP server (from the device at IP 10.9.1.154) that accesses and executes the ztp-simply.py script:

```
auto@automation:~$ sh ~/watch_ztp_logs.sh
```

<img src="imgs/watchlogs.png" style="zoom:67%;" />



You may also start a ping to watch when the interface on the devices comes back online with the configuration applied. Start a ping to 10.1.1.5 which is the IP address specified in the ztp-simple.py file.

```
auto@automation:~$ ping 10.1.1.5
```

Once the ICMP ping replies start then the device has been successfully provisioned - You can now log back in using SSH or Telnet.

<img src="imgs/ping.png" style="zoom:50%;" />

Example output from the serial console that shows successful ZTP and Python file execution:

<img src="imgs/execution.gif" style="zoom:75%;" />

## *Conclusion*

The Cisco IOS XE Catalyst 9300 switch has now successfully completed the Zero Touch Provisioning process and is fully configured on the network. Because of the pre-configuration within the ztp-simple.py file, all use cases for the related IOS XE Programmability Lab have been enabled. Specifically, the switch has an IP, username/password, SSH access enabled, and the programmatic NETCONF and RESTCONF interfaces have also been configured and enabled.