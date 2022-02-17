
<h1 align="center">
  <br>
  <a href="https://k8s.haxx.ninja"><img src="https://github.com/cloudsriseup/Kubernetes_Security_Basics_Walkthrough/raw/main/images/logo.jpg" alt="Kubernetes Security Walkthrough" width="200"></a>
  <br>
  Kubernetes Security Extravanganza
  <br>
</h1>

<h4 align="center">A minimal walkthrough on Kubernetes security features and controls.</h4>


<p align="center">
  <a href="#intro">Introduction</a> •
  <a href="#hand-holding-kubernetes-hardening">Hand Holding Kubernetes Hardening</a> •
  <a href="#simpler-hardening">Simpler Hardening</a> •
  <a href="#tls-certification-rotation">TLS Certification Rotation</a> •
  <a href="#linux-kernel-filtering">Linux Kernel Filtering</a> •
  <a href="#pod-security">Pod Security</a>
  <a href="#netsec">NetSec</a>
  <a href="#cluster">Cluster</a>
  <a href="#admission-controller">Admission Controller</a>
  <a href="#workload-integrity">Workload Integrity</a>
  <a href="#backups">Backups</a>
  <a href="#secrets">Secrets</a>
  <a href="#isolated-kernel">Isolated Kernel</a>
  <a href="#image-pulling">Image Pulling</a>

</p>


## Intro

Hardening an application server that was never intended to be hardened (wrapped by private R&D security tooling not made publicly available) is always a challenge.  Especially in light of everything else one must consider and engineer around;

* Cluster Architecture
* Containers
* Workloads
* Services, Load Balancing, and Networking
* Storage
* Configuration
* Policies
* Scheduling, Preemption and Eviction
* Cluster Administration
* Extending Kubernetes

Which is why it is worthwhile to view my lecture on the project that became Kubernetes.  Design decisions were made of that era's philosophy.  Hence this walkthrough to right the sins that were made.



## Hand Holding Kubernetes Hardening
Interesting manual techniques to harden kubernetes core

## Simpler Hardening
Semi-automated techniques to harden kubernetes core

## TLS Certification Rotation
X.509 builds the internal communication assurance and integrity checks between different Kubernete's services
## Linux Kernel Filtering
SECCOMP goes a long way but is difficult to master

## Pod Security
Pod Security is the new hotness

## NetSec
Service Mesh, DNS, autodiscovery - oh my!

## Cluster
The root of all governance

## Admission Controller
The Gatekeeper of Xul

## Workload Integrity
I think, therefore I am because my identity is mathematicaly proven. 

## Backups
You are only as available as your last successful restoration

## Secrets
Touching other people's underwear

## Isolated Kernel
I heard you like kernels so I put a kernel in your kernel

## Image Pulling
Or why DockerHub had to make money






----


## Credits

 [John Menerick](https://haxx.ninja)

 [Kubernetes](https://kubernetes.io)

 [Hashicorp](https://hashicorp.com)


---

## License

MIT

---

> [k8s.haxx.ninja](https://k8s.haxx.ninja) &nbsp;&middot;&nbsp;
> GitHub [@cloudsriseup](https://github.com/cloudsriseup) &nbsp;&middot;&nbsp;
> Keyoxide [John Menerick](https://keyoxide.org/34904984AC278D90AB17FCF599519FE194F7637A)
