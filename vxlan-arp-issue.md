
# How to troubleshoot VXLAN ARP issue

## Problem

From BIG-IP, ARP records show that it's not able to resolve PODs mac address:

```
root@(bigip-a)(cfg-sync Standalone)(Active)(/Common)(tmos)# show net arp

-------------------------------------------------------------------------------------------------
Net::Arp
Name Address HWaddress Vlan Expire-in-sec Status
-------------------------------------------------------------------------------------------------
10.129.0.158 10.129.0.158 incomplete /Common/openshift-vxlan 0 unknown
10.129.0.159 10.129.0.159 incomplete /Common/openshift-vxlan 0 unknown
10.192.74.174 10.192.74.174 00:50:56:ac:89:8b /Common/internal 42 resolved
10.192.74.175 10.192.74.175 00:50:56:ac:0e:34 /Common/internal 42 resolved
10.192.74.176 10.192.74.176 00:50:56:ac:a6:93 /Common/internal 42 resolved
10.192.74.177 10.192.74.177 00:50:56:ac:42:b7 /Common/internal 42 resolved
10.192.74.178 10.192.74.178 00:50:56:ac:d4:3f /Common/internal 42 resolved
```
 
## Recommeded Actions

**Example Topology**

                                            BIGIP NW                                                                 CLUSTER NW
                                Openshift (Self)       Internal Self                                      Openshift Node.        Backend Pod
                                10.131.2.15            10.192.74.189  ------------------------------------10.192.74.177          10.129.0.158
 

**Verify**
...

 ping Self-ip on BIG-IP from Master:
```
[root@master0 ~]# ping 10.131.2.15
PING 10.131.2.15 (10.131.2.15) 56(84) bytes of data.
64 bytes from 10.131.2.15: icmp_seq=1 ttl=255 time=0.850 ms
64 bytes from 10.131.2.15: icmp_seq=2 ttl=255 time=0.383 ms
64 bytes from 10.131.2.15: icmp_seq=3 ttl=255 time=0.163 ms
```
 

**Verify**
...

```
root@(bigip-a)(cfg-sync Standalone)(Active)(/Common)(tmos)# show net route

-------------------------------------------------------------------------------------
Net::Routes
Name Destination Type NextHop Origin
-------------------------------------------------------------------------------------
127.20.0.0/16 127.20.0.0/16 interface tmm_bp connected
10.192.74.0/24 10.192.74.0/24 interface /Common/internal connected
10.128.0.0/14 10.128.0.0/14 interface /Common/openshift-vxlan connected
127.1.1.0/24 127.1.1.0/24 interface tmm connected
fe80::%vlan4094/64 fe80::%vlan4094/64 interface /Common/internal connected
ff02:fff::/64 ff02:fff::/64 interface tmm_bp connected
fe80::%vlan4095/64 fe80::%vlan4095/64 interface tmm_bp connected
fe80::/64 fe80::/64 interface /Common/openshift-vxlan connected
ff02::/64 ff02::/64 interface /Common/openshift-vxlan connected
fe80::/64 fe80::/64 interface /Common/socks-tunnel connected
fe80::/64 fe80::/64 interface /Common/http-tunnel connected
fe80::%vlan4095/64 fe80::%vlan4095/64 interface /Common/tmm_bp connected
ff02:fff::/64 ff02:fff::/64 interface /Common/tmm_bp connected
ff02:ffe::/64 ff02:ffe::/64 interface /Common/internal connected
ff02::/64 ff02::/64 interface tmm connected
fe80::/64 fe80::/64 interface tmm connected
```

**Verify**
...

### Solution
