qm

Physical network	Proxmox virtual network
Cisco Layer 2 switch	Linux bridge (vmbr1)
Physical Ethernet port	Virtual bridge port
Ethernet cable	Virtual NIC connection
VLAN trunk	VLAN-aware bridge with tagged traffic
Router or Layer 3 switch	FW01 performing routing
Switch MAC address table	Linux bridge forwarding database

sudo systemctl is-active networking
nmcli device status
nmcli device disconnect ens18
sudo nmcli connection modify ens18 connection.autoconnect no
nmcli connection show
getenforce
swapon --show
findmnt /
