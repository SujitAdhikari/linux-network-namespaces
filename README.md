Linux Network Namespaces:
--------------------------
**Namespaces:**
Namespaces are a feature of the Linux kernel that partitions kernel resources such that one set of processes sees one set of resources while another set of processes sees a different set of resources. The feature works by having the same namespace for a set of resources and processes, but those namespaces refer to distinct resources. Resources may exist in multiple spaces. Examples of such resources are process IDs, hostnames, user IDs, file names, and some names associated with network access, and interprocess communication.  

  **Linux Network Namespaces:**
In a network namespace, the scoped ‘identifiers’ are network devices; so a given network device, such as eth0, exists in a particular namespace. Linux starts up with a default network namespace, so if your operating system does not do anything special, that is where all the network devices will be located. But it is also possible to create further non-default namespaces, and create new devices in those namespaces, or to move an existing device from one namespace to another.

Each network namespace also has its own routing table, and in fact this is the main reason for namespaces to exist. A routing table is keyed by destination IP address, so network namespaces are what you need if you want the same destination IP address to mean different things at different times - which is something that OpenStack Networking requires for its feature of providing overlapping IP addresses in different virtual networks.

Each network namespace also has its own set of iptables (for both IPv4 and IPv6). So, you can apply different security to flows with the same IP addressing in different namespaces, as well as different routing.

**Creating and Listing Network Namespaces**  
```
  $ sudo ip netns add netns1  
```
**To verify that the network namespace has been created, use this command:**
```
$ sudo ip netns list  
 ```
We should see your network namespace netns1 listed there, ready for you to use.

**Assigning Interfaces to Network Namespaces:**
```
  $ sudo ip link add veth1 type veth peer name ceth1
```
We can verify that the veth pair was created using this command:
```
$ sudo ip link list
```
We should see a pair of veth interfaces (using the names you assigned in the command above) listed there. Right now, they both belong to the “default” or “global” namespace, along with the physical interfaces.

Let’s say that We want to connect the global namespace to the netns1 namespace. To do that, you’ll need to move one of the veth interfaces to the netns1 namespace using this command:
```
  $ sudo ip link set ceth1 netns netns1
```
If we then run the ip link list command again, we will see that the ceth1 interface has disappeared from the list. It’s now in the netns1 namespace, so to see it you’d need to run this command:
```
$ sudo ip netns exec netns1 ip link list
```
or
```
sudo ip netns exec netns1 sh
```
Now we are in the netns1 namespace with sh shell. Check interface in the netns1 
```
# ip a
```
Now we are able to see ceth1 in the list along with loopback interface(lo). Both link state is down.
**Configuring Interface IP in Network Namespaces**
```
$ sudo ip netns exec netns1 ip addr add 172.20.0.11/16 dev ceth1
$ sudo ip netns exec netns1 ip link set dev ceth1 up
$ sudo  ip netns exec netns1 ip link set lo up
```
or
```
$ sudo ip netns exec netns1 sh
$ sudo ip addr add 172.20.0.11/16 dev ceth1
$ sudo ip link set dev ceth1 up
$ sudo ip link set lo up
$ exit
```
**Configuring IP Address on veth1 in  “default” or “global” Namespaces**
```
  $ sudo ip addr add 172.20.0.1/16 dev veth1
  $ sudo ip link set dev veth1 up
```
Now we will go to netns1 namespace with sh shell.
```
$ sudo ip netns exec netns1 sh
$ ip route
$ route -n
$ ping 172.20.0.11
$ ping 172.20.0.1
$ arp -a
```
We will do the same for another network namespace.
```
$ sudo ip netns add netns2
$ sudo ip link add veth2 type veth peer name ceth2 netns netns2
$ sudo ip netns exec netns2 ip addr add 172.20.0.12/16 dev ceth2
$ sudo ip netns exec netns2 ip link set dev ceth2 up
$ sudo ip netns exec netns2 ip link set lo up
$ sudo ip addr add 172.20.0.2/16 dev veth2
$ sudo ip link set dev veth2 up
$ ping 172.20.0.12 -I veth2
```

  **Create a bridge device naming it `br0` and set it up.**  
  We do not need IP address for veth1 & veth2 because we are going to create bridge interface br0 and we will Add veth1 & veth2 interface to the bridge by setting the bridge device as their master.  
  * Remove IP address for veth1 & veth2
```
 $ sudo ip addr del 172.20.0.1/16  dev veth1
 $ sudo ip addr del 172.20.0.2/16  dev veth2

```
Create a bridge device:
```
$ sudo ip link add br0 type bridge
$ sudo ip link set br0 up
```
  **Add the veth1 & veth2 interface to the bridge by setting the bridge device as their master.**  
```
$ sudo ip link set veth1 master br0
$ sudo ip link set veth2 master br0
```
  **Set the address of the `br0` interface (bridge device)**  
```
$ bridge link show br0
$ sudo ip addr add 172.20.0.10/16 brd + dev br0
```
  **add the default gateway in all the network namespace.**  
```
$ sudo ip netns exec netns1 ip route add default via 172.20.0.10
$ sudo ip netns exec netns2 ip route add default via 172.20.0.10
```
* Set us up to have responses from the network.
* -t specifies the table to which the commands should be directed to. By default, it's `filter`.
* -A specifies that we're appending a rule to the chain that we tell the name after it.
* -s specifies a source address (with a mask in this case).
* -j specifies the target to jump to (what action to take).
```
$ sudo iptables -t nat -A POSTROUTING -s 172.20.0.0/16 -j MASQUERADE
```
**Enable packet forwarding**
```
$ sudo sysctl -w net.ipv4.ip_forward=1
```
sujit@srv:~$ sudo ip netns exec netns2 ping 8.8.8.8  
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.  
64 bytes from 8.8.8.8: icmp_seq=1 ttl=127 time=30.3 ms  
64 bytes from 8.8.8.8: icmp_seq=2 ttl=127 time=27.9 ms  
Now (finally), we’re good! We have connectivity all the way:  

* the host can direct traffic to an application inside a namespace;
* an application inside a namespace can direct traffic to an application in the host;
* an application inside a namespace can direct traffic to another application in another namespace; and
* an application inside a namespace can access the internet.


Reference:  
  https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/  
  https://unix.stackexchange.com/questions/524052/how-to-connect-a-namespace-to-a-physical-interface-through-a-bridge-and-veth-pai
