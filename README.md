# Corondum for NetFPGA-SUME

Generate the .bit file with [Source Code](https://github.com/corundum/corundum).

## NetFPGA-SUME Flash Configuration

[Install Digilent Adept Tools](https://github.com/NetFPGA/NetFPGA-SUME-public/wiki/Reference-Operating-System-Setup-Guide)

Write .bit file in flash
```bash
dsumecfg write -d NetSUME -f file.bit -s <flash_section> -verify
```
there are 4 flash sections (0,1,2 and 3).

Configure the FPGA
```bash
dsumecfg reconfig -d NetSUME -s <flash_section>
```

Set default flash memory boot section
```bash
dsumecfg setbootsec -d NetSUME -s <flash_section>
```
Then reboot the PC.
## Install PCIe Driver
Once the kernel module is built, load it with insmod inside /modules/mqnic/
```bash
sudo insmod mqnic.ko
```
## IP configuration
## Remove Ipv6 address with file
Edit the sysctl configuration file
```bash
sudo nano /etc/sysctl.conf
```
 Paste
 ```bash
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

net.core.rmem_max = 2147483647
net.core.wmem_max = 2147483647
net.core.netdev_max_backlog = 1000000
net.core.optmem_max = 4194304

net.ipv4.tcp_rmem = 4096 87380 2147483647
net.ipv4.tcp_wmem = 4096 65536 2147483647
net.ipv4.udp_mem = 4194304 4194304 4194304
```
Save the file and apply the changes
```bash
sudo sysctl -p
```
NIC configuration
```bash
sudo ip link set enp5s0 up
sudo ip addr add 192.168.1.100/24 dev enp5s0
sudo ip link set mtu 9000 dev enp5s0
sudo ethtool -C enp5s0 rx-usecs 0
sudo sysctl -p
```
See nic interruptions time
```bash
sudo ethtool -c enp5s0
```
### namespace
Create a namespace for nic
```bash
sudo ip netns add corundum
sudo ip netns list
```
This creates and verify a new network namespace called corundum.
Move NIC to the namespace,
```bash
sudo ip link set enp1s0np0 netns corundum
```
enter in the namespace
```bash
sudo ip netns exec corundum bash
```
Corundum NIC configuration
```bash
sudo ip link set enp1s0np0 up
sudo ip addr add 192.168.1.128/24 dev enp1s0np0
sudo ip link set mtu 9000 dev enp1s0np0
sudo sysctl -p
```
### Arp request test
On the server
```bash
sudo arping -I enp1s0np0 192.168.1.128
```
On the namespaced 
```bash
sudo ip netns exec corundum bash
sudo arping -I enp5s0 192.168.1.100
```

## Tests
### Ping
```bash
ping 192.168.1.128
ping 192.168.1.100
```
### iperf
On the server
```bash
iperf3 -s
```
On the client
```bash
iperf3 -c 192.168.1.128
iperf3 -c 192.168.1.100
```
### PTP test
Check de firewall if dont run.
On the server
```bash
sudo ptp4l -i enp5s0 --masterOnly=1 -m --logSyncInterval=-3
sudo ptp4l -i enp1s0np0 --masterOnly=1 -m --logSyncInterval=-3
```
On the client
```bash
sudo ptp4l -i enp1s0np0 --slaveOnly=1 -m
sudo ptp4l -i enp5s0 --slaveOnly=1 -m
```
### sockperf
server
```bash
sockperf server -i 192.168.1.128 -p 1234 -m 8192
sockperf server -i 192.168.1.100 -p 1234 -m 8192
```
client
```bash
sockperf pp -i 192.168.1.128 -p 1234 -n 100000 -m 64 --histogram 1:0:100
sockperf pp -i 192.168.1.100 -p 1234 -n 100000 -m 64 --histogram 1:0:100
```
