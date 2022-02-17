# We are going to use Cillium and Hubble in this tech stack to keep an eye on observability as well as encrypted traffic

## Prereqs
* Move over the rke2 repo configuration into the appropriate rpm directory

# Installing the packages
```
yum install -y epel-release
yum install -y nano curl wget git tmux jq vim-common
yum install -y iscsi-initiator-utils 
modprobe iscsi_tcp
echo "iscsi_tcp" >/etc/modules-load.d/iscsi-tcp.conf
systemctl enable iscsid
systemctl start iscsid 
```

# Setup the network
* Move over the rke2_canal.conf network interface configuration file into the network manager
```
systemctl reload NetworkManager
```
# Install rke2
```
yum -y install rke2-server
```

# Install utilities
```
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
echo 'PATH=$PATH:/usr/local/bin' >> /etc/profile
echo 'PATH=$PATH:/var/lib/rancher/rke2/bin' >> /etc/profile
source /etc/profile
sudo dnf copr -y enable cerenit/helm
sudo dnf install -y helm

#Setup the firewall rules for monitoring, hubble relay, cilium geneve, dns via masquerading, etc...
sudo firewall-cmd --add-port=9345/tcp --permanent
sudo firewall-cmd --add-port=6443/tcp --permanent
sudo firewall-cmd --add-port=10250Air-Gap/tcp --permanent
sudo firewall-cmd --add-port=2379/tcp --permanent
sudo firewall-cmd --add-port=2380/tcp --permanent
sudo firewall-cmd --add-port=30000-32767/tcp --permanent
sudo firewall-cmd --add-port=9796/tcp --permanent
sudo firewall-cmd --add-port=19090/tcp --permanent
sudo firewall-cmd --add-port=6942/tcp --permanent
sudo firewall-cmd --add-port=9091/tcp --permanent
sudo firewall-cmd --add-port=4244/tcp --permanent
sudo firewall-cmd --add-port=4240/tcp --permanent
sudo firewall-cmd --remove-icmp-block=echo-request --permanent
sudo firewall-cmd --remove-icmp-block=echo-reply --permanpublic
sudo firewall-cmd --add-port=6081/udp --permanent
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --zone=public  --add-masquerade --permanent
sudo firewall-cmd --reload
```

* Sanity check firewall work it looks akin to the following; sudo firewall-cmd --list-all
```
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eno3
  sources: 
  services: cockpit dhcpv6-client ssh wireguard
  ports: 9345/tcp 6443/tcp 10250/tcp 2379/tcp 2380/tcp 30000-32767/tcp 4240/tcp 6081/udp 80/tcp 443/tcp 4244/tcp 9796/tcp 19090/tcp 6942/tcp 9091/tcp
  protocols: 
  masquerade: yes
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

# Setup rke2 config directory
```
mkdir -p /etc/rancher/rke2
```
* Move over rke2_config.yaml into the new directory created

# Continue to bootstrap kubernetes
```
sudo cp -f /usr/share/rke2/rke2-cis-sysctl.conf /etc/sysctl.d/60-rke2-cis.conf
sysctl -p /etc/sysctl.d/60-rke2-cis.conf
useradd -r -c "etcd user" -s /sbin/nologin -M etcd
mkdir -p /var/lib/rancher/rke2/server/manifests/
```
* Move over the manifest files: manifest_*.yml

# Given the inherent challenges keeping the engine, addons, and components patched in sync - disable package patching of rke2
add /etc/dnf/dnf.conf or /etc/yum.conf -> exclude=rke2-*

# Sanity check eBPF is installed and functioning.  Should see non on /sys..... type bpf.... as a response to:
```
mount | grep /sys/fs/bpf
```
## If eBPF is not found, then create it by hand by doing the following:
```
sudo mount bpffs -t bpf /sys/fs/bpf
```
```
sudo bash -c 'cat <<EOF >> /etc/fstab
none /sys/fs/bpf bpf rw,relatime 0 0
EOF'
```
# Install Cilium
* Move over the rke2_cilium.yml file into the manifest directory mentioned above

# Try to start rke2 by hand
```
sudo systemctl enable rke2-server --now
sudo systemctl status rke2-server
sudo journalctl -u rke2-server -f
```

# Setup utilities
```
mkdir ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 /root/.kube/config
ln -s /var/lib/rancher/rke2/agent/etc/crictl.yaml /etc/crictl.yaml
kubectl get node
crictl ps
crictl images
kubectl get nodes
```
# Try out hello world like apps to ensure successful deployments
```
kubens default
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.9/examples/minikube/http-sw-app.yaml
kubectl apply -f cilium_demo.yaml
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```