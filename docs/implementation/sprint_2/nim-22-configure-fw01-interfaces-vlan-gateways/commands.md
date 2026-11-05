sysctl net.ipv4.ip_forward
ip neigh show
sudo firewall-cmd --permanent --list-all
sudo firewall-cmd --get-active-zones
sudo nmcli connection modify ens18 connection.zone drop
sudo nmcli connection up ens18
getent ahostsv4 redhat.com
sudo qm help set | grep -A 12 -- '--net[n]'
sudo qm help set | grep -A 15 'trunks='
sudo qm set 101 --net1 'virtio=BC:24:11:D9:A0:51,bridge=vmbr1,firewall=1,trunks=100;200;300;400'
sudo qm status 101
sudo qm config 101 --current | grep '^net'
"sudo nmcli connection add type vlan \
  con-name vlan100-management \
  ifname ens19.100 \
  dev ens19 \
  id 100 \
  ipv4.method manual \
  ipv4.addresses 10.20.10.1/24 \
  ipv4.never-default yes \
  ipv6.method disabled \
  connection.zone drop \
  connection.autoconnect no"
ip -d link show ens19.100
sudo nmcli device reapply ens18
nmcli -f .... connection show <profile>
nmcli -f NAME,TYPE,DEVICE,AUTOCONNECT connection show
nmcli device status
sudo bridge vlan show dev fwpr101p1
ip -6 addr show dev ens18
sudo ss -lntp
sudo nmcli connection modify ens18 ipv6.method disabled
sudo nmcli connection modify vlan400-peripherals connection.autoconnect yes
sudo qm listsnaphot 101
sudo qm rollback 101 <snapshot-name>
sudo journalctl -u NetworkManager -b --no-pager
sudo sysctl -w net.ipv4.ip_forward=0

