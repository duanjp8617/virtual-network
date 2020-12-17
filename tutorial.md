* This demo shows how to create three containers in one virtual machine. One VM 
  serves as the switch, while the other two VMs serve as host machine. The two
  host machines are directly connected by the switch and can ping each other.
  The following text contains procedures for setting up the demo.

* This demo contains a ping test on the following topology.
[host1] --> ping 10.10.0.3 --> [p1:switch:p2] --> [host2]

1. Create a KVM virtual machine running Ubuntu18.04. Install docker.

2. Launch a container, ubuntu:16.04 and ubuntu:18.04 should both work
```bash
sudo docker run --privileged --entrypoint /bin/bash --name host -it -d ubuntu:16.04
```

3. Attach to the container with
```bash
sudo docker exec -it host bash
```
Note: to detach from the container, hit Ctrl+p then Ctrl+q

4. Install the following packages for the container
```bash
sudo apt-get install iproute2
sudo apt-get install iputils-ping
sudo apt-get install tcpdump
sudo apt-get install net-tools
```

5. Detach from the container and shutdown the container
```bash
sudo docker stop host
```

6. Create a new image based on the modified contianer.
```bash
docker commit -m "1" -a "2" host demo_base
```

7. Create one container and one namespace
```bash
sudo ip netns add link
sudo docker run --net=none --privileged --entrypoint /bin/bash --name host1 -it -d demo_base
```

8. Create one pair of interfaces.
```bash
ip link add sp1 type veth peer name hp1
```

9. Create the following folder.
```bash
mkdir -p /var/run/netns
```

10. Create symbolic files for the network namespaces of the containers
```bash
sudo docker inspect -f "{{.State.Pid}}" {container_name}
sudo ln -s /proc/"$CONTAINER_PID"/ns/net /var/run/netns/"$CONTAINER_PID"
```

11. Attach the interfaces to the container.
```bash
sudo ip link set sp1 netns link
sudo ip link set hp1 netns {host1_container_pid}
```

12. Create a vxlan interface, remote should be remote vm's ip, local is this vm's ip.
The id is vni, which should be set to a unique value for different link connections.
The dstport should be fixed at 4789 to comply with standard.
```bash
ip link add vxlan21 type vxlan id 21 dstport 4789 remote 172.16.31.33 local 172.16.31.18
ip link set vxlan21 netns link
```

11. Attach to the switch namespace and set up the bridge
```bash
ip netns exec link ip link add name br0 type bridge
ip netns exec link ip link set dev br0 up
ip netns exec link ip link set dev sp1 master br0
ip netns exec link ip link set dev vxlan21 master br0
ip netns exec link ip link set dev sp1 up
ip netns exec link ip link set dev vxlan21 up
```

12. Attach to the hosts and set up the IP addresses.
```bash
sudo docker exec -it host1 bash
ifconfig hp1 10.10.0.2 netmask 255.255.255.0
```

13. Repeat the previous steps on the remote vm. In step 12, the vni should be 21. The remote and 
local ip addresses should be properly set.

15. move the veth device from the link netns to global netns:
```bash
ip netns exec link ip link set sp1 netns 1
ip netns exec link ip link set sp2 netns 1
```
