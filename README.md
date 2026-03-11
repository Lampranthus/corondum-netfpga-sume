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
sudo ip addr add 192.168.1.100/24 dev enp5s0
```

### namespace
```bash
# Crear namespace para la FPGA
sudo ip netns add fpga-ns

# Mover interfaz FPGA al namespace
sudo ip link set enp1s0np0 netns fpga-ns

# Configurar IP en el namespace
sudo ip netns exec fpga-ns ip addr add 192.168.1.128/24 dev enp1s0np0
sudo ip netns exec fpga-ns ip link set enp1s0np0 up

# Ahora ARP desde el namespace
sudo ip netns exec fpga-ns arping -I enp1s0np0 192.168.1.100

# Abrir una shell bash dentro del namespace
sudo ip netns exec fpga-ns bash

# Deshabilitar IPv6 para esta interfaz
sysctl -w net.ipv6.conf.enp1s0np0.disable_ipv6=1
```
### Arp test
```bash
sudo arping -I enp5s0 192.168.1.128
sudo arping -I enp1s0np0 192.168.1.100
```
