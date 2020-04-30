
# How to troubleshoot VXLAN ARP issue

## Problem

The CIS is properly populating the pools, and OpenShift POD is up and running. From BIG-IP, however, ARP records show that it's not able to resolve PODs mac address:

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
 
## Recommended Actions

**Example Topology**

                                            BIGIP NW                                                                 CLUSTER NW
                                Openshift (Self)       Internal Self                                      Openshift Node.        Backend Pod
                                10.131.2.15            10.192.74.189  ------------------------------------10.192.74.177          10.131.0.185
 

PODs running on OpenShift 4.3:
```
[eric@ns1 ~]$ oc get pod -o wide
NAME                              READY   STATUS    RESTARTS   AGE    IP           NODE                                 NOMINATED NODE   READINESS GATES
f5-hello-world-6df6fcf4fd-lskdb   1/1     Running   0          4d     10.131.0.185   worker0.ocp4-cluster-001.bddemo.io   <none>           <none>
f5-hello-world-6df6fcf4fd-zpttc   1/1     Running   0          4d     10.131.0.187   worker0.ocp4-cluster-001.bddemo.io   <none>           <none>
```

**Verify self-ip is reachable from OpenShift master**

Verify that the internal network’s self IP is reachable from the cluster and pod with both ICMP and TCP traffic.

Issue ping to BIG-IP Self-IP from Master:
```
[root@master0 ~]# ping 10.131.2.15
PING 10.131.2.15 (10.131.2.15) 56(84) bytes of data.
64 bytes from 10.131.2.15: icmp_seq=1 ttl=255 time=0.850 ms
64 bytes from 10.131.2.15: icmp_seq=2 ttl=255 time=0.383 ms
64 bytes from 10.131.2.15: icmp_seq=3 ttl=255 time=0.163 ms
```
 

**Verify routing on BIG-IP** 

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

**TCPDUMP to capture traffic on BIG-IP**


On BIG-IP, issue ping to POD:
```
root@(bigip-a)(cfg-sync Standalone)(LICENSE EXPIRES IN 4 DAYS:Active)(/Common)(tmos)# ping 10.131.0.185
PING 10.131.0.185 (10.131.0.185) 56(84) bytes of data.
From 10.131.0.185 icmp_seq=3 Destination Host Unreachable
From 10.131.0.185 icmp_seq=4 Destination Host Unreachable
From 10.131.0.185 icmp_seq=5 Destination Host Unreachable
From 10.131.0.185 icmp_seq=6 Destination Host Unreachable
```

TCPDUMP on vxlan interface:
```
[root@bigip-a:Active:Standalone] config # tcpdump -ni openshift-vxlan
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on openshift-vxlan, link-type EN10MB (Ethernet), capture size 65535 bytes

11:54:56.193272 IP 10.131.2.15.57488 > 10.131.0.224.webcache: Flags [S], seq 4068912808, win 29200, options [mss 1460,sackOK,TS val 2825163968 ecr 0,nop,wscale 7], length 0 out slot1/tmm0 lis=
11:54:56.193871 IP 10.131.0.224.webcache > 10.131.2.15.57488: Flags [S.], seq 1492491491, ack 4068912809, win 27960, options [mss 1410,sackOK,TS val 2871767082 ecr 2825163968,nop,wscale 7], length 0 in slot1/tmm0 lis=
11:54:56.194143 IP 10.131.2.15.57488 > 10.131.0.224.webcache: Flags [.], ack 1, win 229, options [nop,nop,TS val 2825163968 ecr 2871767082], length 0 out slot1/tmm0 lis=
11:54:56.194211 IP 10.131.2.15.57488 > 10.131.0.224.webcache: Flags [P.], seq 1:15, ack 1, win 229, options [nop,nop,TS val 2825163969 ecr 2871767082], length 14: HTTP out slot1/tmm0 lis=
11:54:56.194417 IP 10.131.0.224.webcache > 10.131.2.15.57488: Flags [.], ack 15, win 219, options [nop,nop,TS val 2871767083 ecr 2825163969], length 0 in slot1/tmm0 lis=
11:54:56.194727 IP 10.131.0.224.webcache > 10.131.2.15.57488: Flags [P.], seq 1:481, ack 15, win 219, options [nop,nop,TS val 2871767083 ecr 2825163969], length 480: HTTP: HTTP/1.1 400 Bad Request in slot1/tmm0 lis=
11:54:56.194960 IP 10.131.2.15.57488 > 10.131.0.224.webcache: Flags [.], ack 481, win 237, options [nop,nop,TS val 2825163969 ecr 2871767083], length 0 out slot1/tmm0 lis=
11:54:56.195038 IP 10.131.2.15.57488 > 10.131.0.224.webcache: Flags [F.], seq 15, ack 481, win 237, options [nop,nop,TS val 2825163969 ecr 2871767083], length 0 out slot1/tmm0 lis=
11:54:56.195109 IP 10.131.0.224.webcache > 10.131.2.15.57488: Flags [F.], seq 481, ack 15, win 219, options [nop,nop,TS val 2871767083 ecr 2825163969], length 0 in slot1/tmm0 lis=
11:54:56.195304 IP 10.131.2.15.57488 > 10.131.0.224.webcache: Flags [.], ack 482, win 237, options [nop,nop,TS val 2825163970 ecr 2871767083], length 0 out slot1/tmm0 lis=
11:54:56.195380 IP 10.131.0.224.webcache > 10.131.2.15.57488: Flags [.], ack 16, win 219, options [nop,nop,TS val 2871767084 ecr 2825163969], length 0 in slot1/tmm0 lis=
11:54:58.672503 IP 10.131.2.15.45168 > 10.131.0.223.webcache: Flags [S], seq 3268459781, win 29200, options [mss 1460,sackOK,TS val 2825166447 ecr 0,nop,wscale 7], length 0 out slot1/tmm0 lis=
11:54:58.672901 IP 10.131.0.223.webcache > 10.131.2.15.45168: Flags [S.], seq 1013539685, ack 3268459782, win 27960, options [mss 1410,sackOK,TS val 671469688 ecr 2825166447,nop,wscale 7], length 0 in slot1/tmm2 lis=
11:54:58.673334 IP 10.131.2.15.45168 > 10.131.0.223.webcache: Flags [.], ack 1, win 229, options [nop,nop,TS val 2825166448 ecr 671469688], length 0 out slot1/tmm0 lis=
11:54:58.673546 IP 10.131.2.15.45168 > 10.131.0.223.webcache: Flags [P.], seq 1:15, ack 1, win 229, options [nop,nop,TS val 2825166448 ecr 671469688], length 14: HTTP out slot1/tmm0 lis=
11:54:58.673838 IP 10.131.0.223.webcache > 10.131.2.15.45168: Flags [.], ack 15, win 219, options [nop,nop,TS val 671469689 ecr 2825166448], length 0 in slot1/tmm2 lis=
11:54:58.673979 IP 10.131.0.223.webcache > 10.131.2.15.45168: Flags [P.], seq 1:481, ack 15, win 219, options [nop,nop,TS val 671469689 ecr 2825166448], length 480: HTTP: HTTP/1.1 400 Bad Request in slot1/tmm2 lis=
11:54:58.673994 IP 10.131.0.223.webcache > 10.131.2.15.45168: Flags [F.], seq 481, ack 15, win 219, options [nop,nop,TS val 671469690 ecr 2825166448], length 0 in slot1/tmm2 lis=
11:54:58.674120 IP 10.131.2.15.45168 > 10.131.0.223.webcache: Flags [.], ack 481, win 237, options [nop,nop,TS val 2825166449 ecr 671469689], length 0 out slot1/tmm0 lis=
```

TCPDUMP on physical interface  (internal):
```
[root@bigip-a:Active:Standalone] config # tcpdump -ni internal host 10.192.74.189 | grep vxlan
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on internal, link-type EN10MB (Ethernet), capture size 65535 bytes
IP 10.131.2.15.32151 > 10.131.0.223.webcache: Flags [S], seq 2336954648, win 29200, options [mss 1460,sackOK,TS val 2825021461 ecr 0,nop,wscale 7], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.32151 > 10.131.0.223.webcache: Flags [.], ack 1, win 229, options [nop,nop,TS val 2825021462 ecr 671324702], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.32151 > 10.131.0.223.webcache: Flags [P.], seq 1:15, ack 1, win 229, options [nop,nop,TS val 2825021462 ecr 671324702], length 14: HTTP out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.0.223.webcache > 10.131.2.15.32151: Flags [.], ack 15, win 219, options [nop,nop,TS val 671324703 ecr 2825021462], length 0 in slot1/tmm3 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.0.223.webcache > 10.131.2.15.32151: Flags [P.], seq 1:481, ack 15, win 219, options [nop,nop,TS val 671324703 ecr 2825021462], length 480: HTTP: HTTP/1.1 400 Bad Request in slot1/tmm3 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.0.223.webcache > 10.131.2.15.32151: Flags [F.], seq 481, ack 15, win 219, options [nop,nop,TS val 671324703 ecr 2825021462], length 0 in slot1/tmm3 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.32151 > 10.131.0.223.webcache: Flags [.], ack 481, win 237, options [nop,nop,TS val 2825021463 ecr 671324703], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.32151 > 10.131.0.223.webcache: Flags [F.], seq 15, ack 481, win 237, options [nop,nop,TS val 2825021463 ecr 671324703], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.32151 > 10.131.0.223.webcache: Flags [.], ack 482, win 237, options [nop,nop,TS val 2825021463 ecr 671324703], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.0.223.webcache > 10.131.2.15.32151: Flags [.], ack 16, win 219, options [nop,nop,TS val 671324703 ecr 2825021463], length 0 in slot1/tmm3 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.26103 > 10.131.0.224.webcache: Flags [S], seq 3246220929, win 29200, options [mss 1460,sackOK,TS val 2825023941 ecr 0,nop,wscale 7], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.26103 > 10.131.0.224.webcache: Flags [.], ack 1, win 229, options [nop,nop,TS val 2825023942 ecr 2871627055], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.26103 > 10.131.0.224.webcache: Flags [P.], seq 1:15, ack 1, win 229, options [nop,nop,TS val 2825023942 ecr 2871627055], length 14: HTTP out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.0.224.webcache > 10.131.2.15.26103: Flags [.], ack 15, win 219, options [nop,nop,TS val 2871627055 ecr 2825023942], length 0 in slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.0.224.webcache > 10.131.2.15.26103: Flags [P.], seq 1:481, ack 15, win 219, options [nop,nop,TS val 2871627055 ecr 2825023942], length 480: HTTP: HTTP/1.1 400 Bad Request in slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.0.224.webcache > 10.131.2.15.26103: Flags [F.], seq 481, ack 15, win 219, options [nop,nop,TS val 2871627055 ecr 2825023942], length 0 in slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.26103 > 10.131.0.224.webcache: Flags [.], ack 481, win 237, options [nop,nop,TS val 2825023942 ecr 2871627055], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
IP 10.131.2.15.26103 > 10.131.0.224.webcache: Flags [F.], seq 15, ack 481, win 237, options [nop,nop,TS val 2825023942 ecr 2871627055], length 0 out slot1/tmm0 lis=_wcard_tunnel_/Common/openshift-vxlan
```

From above, it shows the self IP requesting the mac address of the pod but not getting a response.

**Install TCPDUMP tool on work node**

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

**Tcpdump to capture traffic on OpenShift worker node**


Verify ARP REQUEST is received by work node, and also work node is sending the RESPONSE back.


On worker node, tcpdump first on tunnel interface:
```
[root@worker0 /]# tcpdump -ni tun0 host 10.131.2.15
TO BE ADDED
...

```
It appears the worker node is properly passing the encapsulated ARP requests and replied.


On worker node, tcpdump on physical interface:
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

[root@worker0 /]# tcpdump -ni ens192 | grep -i -B 1 -A 1 reply
[root@worker0 /]# tcpdump -ni ens192 | grep -i -B 2 -A 2 reply

```

In this example, we can see the following on worker node:
- ARP reply is seen on tunnel interface
- ARP reply NOT seen on physical interface

It means the packet stops at worker node. "The issue lies with ovs flow rules which is not forwarding the traffic from node's tun0 interface to physical interface"

## Solution



The solution is to restart ovs POD on workder node, which is managed by SDN-controller:
```
[root@master0 working]# oc get pods -n openshift-sdn -o wide
[eric@ns1 ~]$ oc get pods -n openshift-sdn -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP              NODE                                 NOMINATED NODE   READINESS GATES
ovs-6cp2g              1/1     Running   2          44d   10.192.74.174   master0.ocp4-cluster-001.bddemo.io   <none>           <none>
ovs-kdz2g              1/1     Running   0          14d   10.192.74.178   worker1.ocp4-cluster-001.bddemo.io   <none>           <none>
ovs-r4v78              1/1     Running   0          14d   10.192.74.177   worker0.ocp4-cluster-001.bddemo.io   <none>           <none>
ovs-rp4nn              1/1     Running   0          44d   10.192.74.175   master1.ocp4-cluster-001.bddemo.io   <none>           <none>
ovs-x72jn              1/1     Running   0          44d   10.192.74.176   master2.ocp4-cluster-001.bddemo.io   <none>           <none>
sdn-5t96p              1/1     Running   7          44d   10.192.74.174   master0.ocp4-cluster-001.bddemo.io   <none>           <none>
sdn-bbpgg              1/1     Running   1          44d   10.192.74.176   master2.ocp4-cluster-001.bddemo.io   <none>           <none>
sdn-bd9cz              1/1     Running   2          44d   10.192.74.177   worker0.ocp4-cluster-001.bddemo.io   <none>           <none>
sdn-bgsj7              1/1     Running   1          44d   10.192.74.175   master1.ocp4-cluster-001.bddemo.io   <none>           <none>
sdn-controller-6t9mg   1/1     Running   22         44d   10.192.74.176   master2.ocp4-cluster-001.bddemo.io   <none>           <none>
sdn-controller-7wbwg   1/1     Running   24         44d   10.192.74.175   master1.ocp4-cluster-001.bddemo.io   <none>           <none>
sdn-controller-p546q   1/1     Running   18         44d   10.192.74.174   master0.ocp4-cluster-001.bddemo.io   <none>           <none>
sdn-vcvjb              1/1     Running   4          44d   10.192.74.178   worker1.ocp4-cluster-001.bddemo.io   <none>           <none>
[eric@ns1 ~]$ oc delete pod ovs-r4v78 -n openshift-sdn
pod "ovs-r4v78" deleted

