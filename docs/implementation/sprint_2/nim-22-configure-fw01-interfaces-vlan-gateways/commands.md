sysctl net.ipv4.ip_forward
ip neigh show
sudo firewall-cmd --permanent --list-all
sudo firewall-cmd --get-active-zones
sudo nmcli connection modify
sudo nmcli connection modify ens18 connection.zone drop
sudo nmcli connection up ens18
getent ahostsv4 redhat.com
