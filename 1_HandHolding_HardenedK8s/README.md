## Pre-reqs
* Move the rke2 ranch repo specification to your yum's repo directory
* Move the rke2_config.yaml config to /etc/rancher/rke2/ directory

# Installation & the best cli text editor
```
yum -y install rke2-server vim
```

# Create an internal registry and unpack all images from the tarbal
```
mkdir -p /var/lib/rancher/rke2/agent/images/
scp rke2-images.linux-amd64.tar kmaster_001:/var/lib/rancher/rke2/agent/images/
cd /var/lib/rancher/rke2/agent/images/
```

# Setup the paths to the applicable binaries
```
echo 'PATH=$PATH:/usr/local/bin' >> /etc/profile
echo 'PATH=$PATH:/var/lib/rancher/rke2/bin' >> /etc/profile
source /etc/profile
```

# Ensure the local firewall and SELinux is running properly
```
setenforce 1
getenforce
sed -i 's/=\(disabled\|permissive\)/=enforcing/g' /etc/sysconfig/selinux
systemctl start firewalld
systemctl enable firewalld
```

# Defense in depth by ensuring airgap nodes remain airgapped locally from the Internet
```
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -p tcp -m tcp --dport=443 -j DROP
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -p tcp -m tcp --dport=80 -j DROP
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 1 -j ACCEPT
firewall-cmd --reload
```

# Setup the hardened rke2 config and ensure it uses the correct user
```
sudo cp -f /usr/share/rke2/rke2-cis-sysctl.conf /etc/sysctl.d/60-rke2-cis.conf
sysctl -p /etc/sysctl.d/60-rke2-cis.conf
useradd -r -c "etcd user" -s /sbin/nologin -M etcd
```

# Configure the engine and APIs
```
mkdir -p /var/lib/rancher/rke2/server/manifests/
```
* Move the manifest files into /var/lib/rancher/rke2/server/manifests directory.  Strive to pick the correct interface.

# Setup the kubernetes service
```
systemctl enable rke2-server.service
systemctl start rke2-server.service
journalctl -u rke2-server -f
mkdir ~/.kube
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config
ln -s /var/lib/rancher/rke2/agent/etc/crictl.yaml /etc/crictl.yaml
chmod 600 ~/.kube/config
kubectl get node
crictl ps
crictl images
```

# Remove the apparmor lines
```
kubectl edit psp global-restricted-psp
```
