Linux Network Namespaces:
--------------------------
**Namespaces**
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
You should see your network namespace netns1 listed there, ready for you to use.

**Assigning Interfaces to Network Namespaces:**
```
  $ sudo ip link add veth1 type veth peer name ceth1
```
You can verify that the veth pair was created using this command:
```
$ sudo ip link list
```
You should see a pair of veth interfaces (using the names you assigned in the command above) listed there. Right now, they both belong to the “default” or “global” namespace, along with the physical interfaces.

Let’s say that you want to connect the global namespace to the netns1 namespace. To do that, you’ll need to move one of the veth interfaces to the netns1 namespace using this command:
```
  $ sudo ip link set ceth1 netns netns1
```
If you then run the ip link list command again, you’ll see that the ceth1 interface has disappeared from the list. It’s now in the netns1 namespace, so to see it you’d need to run this command:
```
ip netns exec netns1 ip link list
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
$ sudo ip netns exec netns1 ip addr add 172.20.0.1/16 dev ceth1
$ sudo ip netns exec netns1 ip link set dev ceth1 up
$ sudo  ip netns exec netns1 ip link set lo up
```
or
```
$ sudo ip netns exec netns1 sh
$ ip addr add 172.20.0.2/16 dev ceth1
$ ip link set dev ceth1 up
$ ip link set lo up
$ exit
```
**Configuring IP Address on veth1 in  “default” or “global” Namespaces**
```
  $ ip addr add 172.20.0.1/16 deb veth1
  $ ip link set dev veth1 up
```
Now we will go to netns1 namespace with sh shell.
```
$ sudo ip netns exec netns1 sh
$ ip route
$ route -n
$ ping 172.20.0.1
$ ping 172.20.0.2
$ arp -a
```
