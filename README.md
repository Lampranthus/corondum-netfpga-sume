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
```bash
sudo ip link set enp1s0np0 up
sudo ip addr add 192.168.1.128/24 dev enp1s0np0
```
 Desactivate IPv6
 ```bash
sysctl -w net.ipv6.conf.enp1s0np0.disable_ipv6=1
```

### namespace
Creathe a namespace for nic
```bash
sudo ip netns add ns-nic
sudo ip netns list
```
This creates and verify a new network namespace called ns-nic.
Make the namespace persistent with services.
```bash
sudo nano /etc/systemd/system/network-namespace@.service
```
paste
```bash
[Unit]
Description=Network namespace %I
Before=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/ip netns add %I
ExecStop=/bin/ip netns del %I

[Install]
WantedBy=multi-user.target
```
```bash
sudo nano /etc/systemd/system/move-enp5s0-to-ns.service
```
paste
```bash
[Unit]
Description=Move enp5s0 to ns-nic namespace
Requires=network-namespace@ns-nic.service
After=network-namespace@ns-nic.service
Before=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/ip link set enp5s0 netns ns-nic
ExecStart=/bin/ip -n ns-nic link set lo up
ExecStart=/bin/ip -n ns-nic link set enp5s0 up

# Disable IPv6 for the interface in the namespace
ExecStart=/bin/ip -n ns-nic netns exec sysctl -w net.ipv6.conf.enp5s0.disable_ipv6=1

# Add your IP configuration here (IPv4 only now)
ExecStart=/bin/ip -n ns-nic addr add 192.168.1.100/24 dev enp5s0
ExecStart=/bin/ip -n ns-nic route add default via 192.168.1.1

[Install]
WantedBy=multi-user.target
```
run services
```bash
sudo systemctl daemon-reload
sudo systemctl enable network-namespace@ns-nic.service
sudo systemctl enable move-enp5s0-to-ns.service
sudo systemctl start network-namespace@ns-nic.service
sudo systemctl start move-enp5s0-to-nicns.service
```
enter in the namespace
```bash
sudo ip netns exec nicns bash
```
 Desactivate IPv6
 ```bash
sysctl -w net.ipv6.conf.enp5s0.disable_ipv6=1
```
Nic configuration
```bash
sudo ethtool -C enp5s0 rx-usecs 0
```
See nic interruptions time
```bash
sudo ethtool -c enp5s0
```
### Arp request test
On the server
```bash
sudo arping -I enp1s0np0 192.168.1.100
```
On the namespaced 
```bash
sudo ip netns exec nicns bash
sudo arping -I enp5s0 192.168.1.128
```

## Remove Ipv6 address with file
Edit the sysctl configuration file
```bash
sudo nano /etc/sysctl.conf
```
Add lines to the end of the file
```bash
net.ipv6.conf.[interface].disable_ipv6 = 1
```
Save the file and apply the changes
```bash
sudo sysctl -p
```

## Set MTU 
```bash
sudo ip link set mtu 9000 dev enp1s0np0
```
On namespace
```bash
sudo ip netns exec nicns bash
sudo ip link set mtu 9000 dev enp5s0
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
On the namespace (client)
```bash
iperf3 -c 192.168.1.128
```
### PTP
On the server
```bash
sudo ptp4l -i enp1s0np0 --masterOnly=1 -m --logSyncInterval=-3
```
On the namespace (client)
```bash
sudo ptp4l -i enp5s0 --slaveOnly=1 -m
```
### sockperf
server
```bash
sockperf server -i 192.168.1.128 -p 1234 -m 8192
```
client
```bash
sockperf pp -i 192.168.1.128 -p 1234 -n 100000 -m 64 --histogram 1:0:100
```
