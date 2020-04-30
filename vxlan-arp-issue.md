
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
                                10.131.2.15            10.192.74.189  ------------------------------------10.192.74.177          10.129.0.185
 

**Verify self-ip is reachable from OpenShift master**

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

Install TCPDUMP on worker node:
```
[core@worker0 ~]$ toolbox
Trying to pull registry.redhat.io/rhel8/support-tools...
Getting image source signatures
Copying blob ee2244abc66f done
...
support-tools:latest
[root@worker0 /]# dnf -y install tcpdump
Updating Subscription Management repositories.
...
Nothing to do.
Complete!
[root@worker0 /]# tcpdump -ni tun0
```

tcpdump on tunnel interface:
```
[root@worker0 /]# tcpdump -ni tun0 host 10.131.2.15

[root@worker0 /]# tcpdump -ni tun0 host 10.131.2.15
[root@worker0 /]# tcpdump -ni ens192 host 10.192.74.189
[root@worker0 /]# tcpdump -ni ens192 | grep -B 1 -A 1 10.131.0.185
[root@worker0 /]# tcpdump -ni ens192 | grep -i -B 1 -A 1 reply
[root@worker0 /]# tcpdump -ni ens192 | grep -i -B 2 -A 2 reply

```

tcpdump on physical interface:
```
[root@worker0 /]# tcpdump -ni ens192 | grep -i -B 2 -A 2 10.131.0.185
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens192, link-type EN10MB (Ethernet), capture size 262144 bytes
ARP, Request who-has 10.131.0.187 tell 10.131.2.15, length 46
03:45:04.075643 IP 10.192.74.189.26340 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
ARP, Request who-has 10.131.0.185 tell 10.131.2.15, length 46
03:45:04.111634 IP 10.192.74.174.sun-sr-https > 10.192.74.177.48040: Flags [P.], seq 1109:1613, ack 1, win 1432, options [nop,nop,TS val 2683476539 ecr 1215308645], length 504
03:45:04.111738 IP 10.192.74.177.48040 > 10.192.74.174.sun-sr-https: Flags [.], ack 1613, win 7861, options [nop,nop,TS val 1215308717 ecr 2683476539], length 0
--
IP 10.131.0.180.39244 > 10.128.0.7.pcsync-https: Flags [.], ack 3864, win 1378, options [nop,nop,TS val 202700768 ecr 4023279153], length 0
03:45:05.075486 IP 10.192.74.189.26340 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
ARP, Request who-has 10.131.0.185 tell 10.131.2.15, length 46
03:45:05.075501 IP 10.192.74.189.26340 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
ARP, Request who-has 10.131.0.187 tell 10.131.2.15, length 46
--
ARP, Request who-has 10.131.0.187 tell 10.131.2.15, length 46
03:45:06.075578 IP 10.192.74.189.26340 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
ARP, Request who-has 10.131.0.185 tell 10.131.2.15, length 46
03:45:06.139166 IP 10.192.74.175.60466 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 11660792
ARP, Request who-has 10.131.0.180 tell 10.130.0.30, length 28
--
03:45:06.443015 IP 10.192.74.177.48040 > 10.192.74.174.sun-sr-https: Flags [.], ack 5987, win 7861, options [nop,nop,TS val 1215311048 ecr 2683478830], length 0
03:45:06.449549 IP 10.192.74.178.47596 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
IP 10.129.0.1.55274 > 10.131.0.185.webcache: Flags [S], seq 2743833764, win 28200, options [mss 1410,sackOK,TS val 4072969755 ecr 0,nop,wscale 7], length 0
03:45:06.449673 IP 10.192.74.177.34421 > 10.192.74.178.vxlan: VXLAN, flags [I] (0x08), vni 0
IP 10.131.0.185.webcache > 10.129.0.1.55274: Flags [S.], seq 2278705180, ack 2743833765, win 27960, options [mss 1410,sackOK,TS val 654834503 ecr 4072969755,nop,wscale 7], length 0
03:45:06.449866 IP 10.192.74.178.47596 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
IP 10.129.0.1.55274 > 10.131.0.185.webcache: Flags [R.], seq 1, ack 1, win 221, options [nop,nop,TS val 4072969755 ecr 654834503], length 0
03:45:06.494101 IP 10.192.74.179.52856 > 10.192.74.177.http: Flags [S], seq 2350636533, win 29200, options [mss 1460,sackOK,TS val 2892068440 ecr 0,nop,wscale 7], length 0
03:45:06.494198 IP 10.192.74.177.http > 10.192.74.179.52856: Flags [S.], seq 2459100841, ack 2350636534, win 28960, options [mss 1460,sackOK,TS val 2622523801 ecr 2892068440,nop,wscale 7], length 0
--
03:45:07.014961 IP 10.192.74.177.50034 > 10.192.74.179.sun-sr-https: Flags [.], ack 6878, win 14775, options [nop,nop,TS val 2622524321 ecr 2892068920], length 0
03:45:07.075416 IP 10.192.74.189.26340 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
ARP, Request who-has 10.131.0.185 tell 10.131.2.15, length 46
03:45:07.075442 IP 10.192.74.189.26340 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
ARP, Request who-has 10.131.0.187 tell 10.131.2.15, length 46
--
ARP, Request who-has 10.131.0.187 tell 10.131.2.15, length 46
03:45:08.075487 IP 10.192.74.189.26340 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 0
ARP, Request who-has 10.131.0.185 tell 10.131.2.15, length 46
03:45:08.116198 IP 10.192.74.178.54607 > 10.192.74.177.vxlan: VXLAN, flags [I] (0x08), vni 9267632
IP 10.129.0.254.42216 > 10.131.0.11.9095: Flags [P.], seq 353932834:353934086, ack 3087874079, win 1393, options [nop,nop,TS val 980078976 ecr 1418219764], length 1252
```
### Solution

```
root@master0 working]# oc get pods -n openshift-sdn
[root@master0 working]# oc get pods -n openshift-sdn -o wide
[root@master0 working]# oc delete pod ovs-2fm5n -n openshift-sdn
pod "ovs-2fm5n" deleted
[root@master0 working]# oc get pods -n openshift-sdn
```