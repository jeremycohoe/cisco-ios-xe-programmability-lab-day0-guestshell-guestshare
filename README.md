## [IOS XE Programmability Lab](https://github.com/jeremycohoe/cisco-ios-xe-programmability-lab)

## Module: Guestshell and NETCONF

## Version: 17.6

## Topics Covered:
[Guestshell](#guestshell)

[Enable Guestshell](#enable-guestshell)

[Guestshell with NETCONF](#guestshell-with-NETCONF)

[Conclusion](#conclusion)


## Guestshell

In this section we will look at IOS XE's on-box Linux container and its capabilities. We will see how to enable it, how to use it to run Python scripts, and how to integrate it NETCONF API

## Enable Guestshell

Guestshell Python runs in an LXC container. This container is managed by IOX, which is a container manager specifically for IOS XE which is similar in function to Docker. Before using the guestshell, we must enable IOX and then enable guestshell.

Step 1.  Login to the RDP from the POD access sheet provided by the proctor and connect to the C9300 switch using terminal

```
auto@programmability:~$ ssh admin@10.1.1.5
Password: Cisco123
C9300# 
```

Step 2.  Enter the following command to check and enable **IOX** on the
device

```
C9300# show iox-service

IOx Infrastructure Summary:
---------------------------
IOx service (CAF)              : Running
IOx service (HA)               : Running
IOx service (IOxman)           : Running
IOx service (Sec storage)      : Not Running
Libvirtd 5.5.0                 : Running
Dockerd 18.03.0                : Running
Application DB Sync Info       : Available
Sync Status                    : Disabled
```

If the IOX services are not in the correct state as per above then the IOX service can be restarted by ending the "no iox, iox" CLI commands if needed:

```
C9300#conf t
Enter configuration commands, one per line. End with CNTL/Z.
C9300(config)# no iox
...
C9300(config)# iox
```

The **show iox-service** and the **show app-hosting list** is seen below, as well as the running configuration for the guest-shell section. This configuration may not yet be present in your switch, and can be added in Step3. **show app-hosting list** command may take up to 30 seconds to execute.

![](imgs/showiox.png)


Additionally the **show app-hosting list** CLI can be used to show the state of the guestshell container:


```
C9300#show app-hosting list
No App found

or

C9300#show app-hosting list
App id                                   State
---------------------------------------------------------
guestshell                               RUNNING

C9300#
```


Step 3. Configure and enable guestshell with the following commands:

```
C9300#

conf t
iox
ip nat inside source list NAT_ACL interface vlan 1 overload
ip access-list standard NAT_ACL
permit 192.168.0.0 0.0.255.255
exit
vlan 4094
exit
int vlan 4094
ip address 192.168.2.1 255.255.255.0
ip nat inside
ip routing
ip route 0.0.0.0 0.0.0.0 10.1.1.3

app-hosting appid guestshell
 app-vnic AppGigabitEthernet trunk
  vlan 4094 guest-interface 0
   guest-ipaddress 192.168.2.2 netmask 255.255.255.0
   exit
  exit
 app-default-gateway 192.168.2.1 guest-interface 0
 name-server0 128.107.212.175
 name-server1 64.102.6.247
 exit
interface AppGigabitEthernet1/0/1
 switchport mode trunk
end
```

**NOTE** If you see errors with **ip nat** commands then check the license level as DNA Advantage is required. Your switch has already been configured with the DNA Advantage license.

![](imgs/enablegs.png)

Step 4.  Start the Guest Shell container to enter into the Bash shell by sending the **guestshell enable** followed by the **guestshell** CLI - Note that it may take up to 1 minute to enable and enter the container.

```
C9300#guestshell enable

<< wait about 30 seconds >>

Interface will be selected if configured in app-hosting
Please wait for completion
guestshell installed successfully
Current state is: DEPLOYED
guestshell activated successfully
Current state is: ACTIVATED
guestshell started successfully
Current state is: RUNNING
Guestshell enabled successfully

C9300#guestshell

<< wait about 1 minute >>

[guestshell@guestshell ~]$

```

![](./imgs/enableenterls.png)

Step 5. Enter the guestshell CLI. This guestshell container can access the device bootflash **guest-share** directory only.

```
c9300# guestshell

[guestshell@guestshell ~]$ 

```

You are now in the Guest Shell Linux container. Check the python version with **python3 --version** command

```
[guestshell@guestshell ~]$ python3 --version
Python 3.6.8
[guestshell@guestshell ~]$
```

In the example above we show that Python3 is installed. Next use the NETCONF API.



## Guestshell with NETCONF

The below python was generated from YANG Suite's NETCONF plugin. This payload is used to read the hostname from the Cisco-IOS-XE-native.YANG - this is the YANG data model where most of the IOS XE configuration is modelled. 

This script has already been saved onto the Ubuntu VM in /tftpboot/get_hostname.py and is available from http://10.1.1.3/get_hostname.py or from tftp://10.1.1.3/get_hostname.py. From within the Guestshell copy this file from the Linux VM into the container with the curl command:

**curl http://10.1.1.3/get_hostname.py -o ~/get_hostname.py**

```
from ncclient import manager
import sys
import xml.dom.minidom

HOST = '127.0.0.1'
# use the NETCONF port for your device
PORT = 830
# use the user credentials for your  device
USER = 'admin'
PASS = 'Cisco123'

FILTER = '''
                <filter xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
                  <native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
                    <hostname></hostname>
                  </native>
                </filter>
            '''

def main():
    """
    Main method that prints netconf capabilities of remote device.
    """
    # Create a NETCONF session to the router with ncclient
    with manager.connect(host=HOST, port=PORT, username=USER,
                         password=PASS, hostkey_verify=False,
                         device_params={'name': 'default'},
                         allow_agent=False, look_for_keys=False) as m:

        # Retrieve the configuraiton
        results = m.get_config('running', FILTER)
        # Print the output in a readable format
        print(xml.dom.minidom.parseString(results.xml).toprettyxml())


if __name__ == '__main__':
    sys.exit(main())
```


Run the python file with the following command: **python3 ~/get_hostname.py** 

The output should be similar to the following where the hostname of C9300 is seen:

```
[guestshell@guestshell ~]$ python3 get_hostname.py
<?xml version="1.0" ?>
<rpc-reply message-id="urn:uuid:7e6042c3-4c32-4e27-bec3-7b8730607877" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
	<data>
		<native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
			<hostname>C9300</hostname>
		</native>
	</data>
</rpc-reply>
```

The aboce example shows that Guesthshell has made a connection to the localhost 127.0.0.1 on port 830 which is connected to IOS XE's netconf interface for management and operation data use cases.


## Conclusion

In this module the Guestshell was configured, enabled, and the NETCONF API was used to interact with IOS XE from the Guestshell using NETCONF payloads.
