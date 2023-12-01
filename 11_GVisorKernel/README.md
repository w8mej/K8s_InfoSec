# GVisor or yet another isolated kernel
Now back to doing security for security sake.  In this case, the ring OS security model leaves much to be desired when one would prefer their workloads to have hardware and OS-level assurances via novel labeling and isolation which contain the effects of exploitable software flaws, including inter-process communication and system-wide information flow.  Either by a new event process abstraction via lightweight, isolated contexts within a single process or similar techniques.

 <a href="https://k8s.haxx.ninja"><img src="https://github.com/w8mej/K8s_InfoSec/raw/main/11_GVisorKernel/GVisor.png" alt="w8mej k8s GVisor" width="200"></a>

## Background
gVisor is an application kernel, written in Go, that implements a substantial portion of the Linux system surface. It includes an Open Container Initiative (OCI) runtime called runsc that provides an isolation boundary between the application and the host kernel. The runsc runtime integrates with Docker and Kubernetes, making it simple to run sandboxed containers.

## Why?! yet another middleware?
Containers are not a sandbox. While containers have revolutionized how we develop, package, and deploy applications, using them to run untrusted or potentially malicious code without additional isolation is not a good idea. While using a single, shared kernel allows for efficiency and performance gains, it also means that container escape is possible with a single vulnerability.  gVisor is an application kernel for containers. It limits the host kernel surface accessible to the application while still giving the application access to all the features it expects. Unlike most kernels, gVisor does not assume or require a fixed set of physical resources; instead, it leverages existing host kernel functionality and runs as a normal process. In other words, gVisor implements Linux by way of Linux.  gVisor should not be confused with technologies and tools to harden containers against external threats, provide additional integrity checks, or limit the scope of access for a service. One should always be careful about what data is made available to a container.  More information at gvisor.dev

## Installation
run this bash script's content to install the correct runtime shim
```
#!/bash
(
  set -e
  URL=https://storage.googleapis.com/gvisor/releases/release/latest
  wget ${URL}/runsc ${URL}/runsc.sha512 \
    ${URL}/gvisor-containerd-shim ${URL}/gvisor-containerd-shim.sha512 \
    ${URL}/containerd-shim-runsc-v1 ${URL}/containerd-shim-runsc-v1.sha512
  sha512sum -c runsc.sha512 \
    -c gvisor-containerd-shim.sha512 \
    -c containerd-shim-runsc-v1.sha512
  rm -f *.sha512
  chmod a+rx runsc gvisor-containerd-shim containerd-shim-runsc-v1
  sudo mv runsc gvisor-containerd-shim containerd-shim-runsc-v1 /usr/local/bin
)

bash install_gvisor.sh
```


Then make sure to copy the appropriate configs. 
```
cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml /var/lib/rancher/k3s/agent/etc/containerd/config.toml.back

cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml.back /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
```

Then edit the underlying OCI config with gvisor's shim
```
vim /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl

[plugins.cri.containerd]
  disable_snapshot_annotations = true
  snapshotter = "overlayfs"

disabled_plugins = ["restart"]

[plugins.linux]
  shim_debug = true

[plugins.cri.containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"

[plugins.cri.cni]
```

Then bounce kubernetes (k3s in this case.)  Make Kubernetes aware of the new runtime
```
cat<<EOF | kubectl apply -f -
apiVersion: node.k8s.io/v1beta1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
EOF
```

Then try to deploy hello world web server on top of the gvisor shim
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: www-runc
spec:
  containers:
  - image: nginx:1.18
    name: www
    ports:
    - containerPort: 80
EOF

cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: untrusted
  name: www-gvisor
spec:
  runtimeClassName: gvisor
  containers:
  - image: nginx:1.18
    name: www
    ports:
    - containerPort: 80
EOF
```

When one asks for the pods' descriptions and details, they will observe the gvisor shim running as well as the hello world web server protected by the gvisor shim

# Conclusion
Yes, it really is that easy to operate, install, maintain, and execute

