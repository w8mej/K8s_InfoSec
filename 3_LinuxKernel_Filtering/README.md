# Kubernete's security model
Kubernete's model heavily relies upon the hosts' OS kernel to perform certain checks in userland and kernel land.  One of the security controls that was bolted onto Kubernetes and works well with BSD and the Linux Kernel is syscall filtering.  One may rely upon the asset's syscall filtering to allow or not allow a list of system calls to be executed.  More details at https://securesql.info/cloud-security/2020/11/19/generic-cloud-native-kubernete-things-need-securing .

# SECCOMP and Namespace control techniques
For simplicity sake, we will be relying upon Linux's Seccomp features and Kubernete's Namespaces.  Namespaces are one of the building blocks of isolation used by the docker-formatted containers. They provide such an environment for a process, that prevents the process from seeing or interacting with other processes. For example, a process inside a container can have PID 1, and the same process can have a normal PID outside of a container. The process ID (PID) namespace is the mechanism which remaps PIDs inside a container. However, containers can still access some resources from the host such as the kernel and kernel modules, the /proc file system and the system time. The Linux Capabilities and seccomp features can limit access by containerized processes to the system features.  The Linux capabilities feature breaks up the privileges available to processes run as the root user into smaller groups of privileges. This way a process running with root privilege can be limited to get only the minimal permissions it needs to perform its operation. A good strategy is to drop all capabilities and add the needed ones back.  If you are building a container which the Network Time Protocol (NTP) daemon, ntpd, you will need to add SYS_TIME so this container can modify the host’s system time. Otherwise the container will not run.  

## SECCOMP
Secure Computing Mode (seccomp) is an OS kernel feature that allows you to filter system calls to the kernel from a container. The combination of restricted and allowed calls are arranged in profiles, and you can pass different profiles to different containers. Seccomp provides more fine-grained control than capabilities, giving an attacker a limited number of syscalls from the container.  The default seccomp profile for docker is a JSON file and can be viewed here: https://github.com/docker/docker/blob/master/profiles/seccomp/default.json. It blocks 44 system calls out of more than 300 available.Making the list stricter would be a trade-off with application compatibility. A table with a significant part of the blocked calls and the reasoning for blocking can be found here: https://docs.docker.com/engine/security/seccomp/.  Seccomp uses the Berkeley Packet Filter (BPF) system, which is programmable on the fly so you can make a custom filter. You can also limit a certain syscall by also customizing the conditions on how or when it should be limited. A seccomp filter replaces the syscall with a pointer to a BPF program, which will execute that program instead of the syscall. All children to a process with this filter will inherit the filter as well. 


# Sanity check the existing seccomp filtering ruleset effectiveness by running the following sanity checker image
```
kubectl run -it bash --image=r.j3ss.co/amicontained --restart=Never bash
.....
Container Runtime: docker
Has Namespaces:
 pid: true
 user: false
AppArmor Profile: unconfined
Capabilities:
 BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: disabled
Blocked Syscalls (21):
 MSGRCV SYSLOG SETSID VHANGUP PIVOT_ROOT ACCT SETTIMEOFDAY SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME INIT_MODULE DELETE_MODULE LOOKUP_DCOOKIE KEXEC_LOAD FANOTIFY_INIT OPEN_BY_HANDLE_AT FINIT_MODULE KEXEC_FILE_LOAD BPF
Looking for Docker.sock
.....
```
As we see, 21 distinct system calls are blocked.  It isn't a bad start.  However, there is much risk to be managed beyond the default policy applied to your cluster.

## Let's take the default seccomp policy for a spin on the cluster
```
vim /var/lib/kubelet/config.yaml
#Ensure the seccomdefault parameter is set to true and enable seccomp in the default runtime.  If you are modifying an existing cluster and do not wish to affect all runtime environments, apply seccompdefault to your test runtime label.
--feature-gates="...,SeccompDefault=true"
--seccomp-default RuntimeDefault
```
### Restart kubelet to grab the new runtime configuration after modifying it
```
systemctl restart kubelet
kubectl edit cm kubelet-config-1.23 -n kube-system
## with the below settings....
- --feature-gates="...,SeccompDefault=true"
- --seccomp-default RuntimeDefault
```
## Sanity check the default seccomp policy applied to the test runtime
```
kubectl run -it bash --image=r.j3ss.co/amicontained --restart=Never bash
....
Container Runtime: containerd
Has Namespaces:
 pid: true
 user: false
AppArmor Profile: unconfined
Capabilities:
 BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: filtering
Blocked Syscalls (61):
 MSGRCV PTRACE SYSLOG SETSID USELIB USTAT SYSFS VHANGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPERM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFSSERVCTL GETPMSG PUTPMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANOTIFY_INIT NAME_TO_HANDLE_AT OPEN_BY_HANDLE_AT SETNS PROCESS_VM_READV PROCESS_VM_WRITEV KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD PKEY_MPROTECT PKEY_ALLOC PKEY_FREE
Looking for Docker.sock
.....
```
One will observe we are now blocking an additional 40 system calls that are risky in nature, enable simpler host privileges escalation / pivoting, or inquire additional information about the host's security posture.

# PodSec
For older clusters supporting pod security policies, one may write a podsec policy that enables the seccomp policy to the appropriate runtime environment.  For example:
```
apiVersion: v1
kind: Pod
metadata:
  name: some-pod
  labels:
    app: some-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: very_secure_seccomp_Apache2xReverseProxy.json
  containers:
```
## Custom Seccomp policies.  
Traditionally, Seccomp policies are written in eBPF.  However, there exists features and tooling which will allow one to write in JSON.  

### What do I need?
One is able to use dtrace or strace to identify and capture which system calls are required by your application.  These utilities work by monitoring the process(es)'s behavior over a time interval to capture the system calls to the OS kernel.  This requires a bit of a burn in session while exhaustively covering nearly every use case of the application to provide assurance the policy will not break the application.  Thankfully there are other utilities to make this process much less painful to adopt and operationalize.

## ZAZ
ZAZ is a wonderful, simple utility to identify and create the policy for a simple application on the fly.  https://github.com/pjbgf/zaz has the details.
### Identify the seccomp policy for telnet socket over port 53 to IP 1.1.1.1 on containerd within alpine2.
```
zaz seccomp containerd alpine2 "telnet 1.1.1.1 53"
```

## Manual policy construction

If you wish to manually craft a seccomp policy, take a look at simple_default_seccomp.json
```
{
    # By default - do not allow by responding with an error
    "defaultAction": "SCMP_ACT_ERRNO",
    # The supported architectures.  Not all system calls are supported on x86 vs. ARMF vs......
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    # The system calls we wish to allow.  Additional details and options on which calls may be available for a Linux system - https://www.kernel.org/doc/html/v4.16/userspace-api/seccomp_filter.html 
    "syscalls": [
        {
            "names": [
                "arch_prctl",
                "sched_yield",
                "futex",
                "write",
                "mmap",
                "exit_group",
                "madvise",
                "rt_sigprocmask",
                "getpid",
                "gettid",
                "tgkill",
                "rt_sigaction",
                "read",
                "getpgrp"
            ],
	    # To allow the above system calls, the below ...ALLOW action needs to be defined.  There are other actions available so please investigate to find the action well suited for your use case.
            "action": "SCMP_ACT_ALLOW",
   "args": [],
   "comment": "",
   "includes": {},
   "excludes": {}
        }
    ]
}

```
Instead of outright denying via SCMP_ACT_ERRNO, one may use SCMP_ACT_LOG to log the calls instead of blocking.  The example policy would look like the following (while still blocking ptrace and other calls that are not acceptable.)
```
{
    "defaultAction": "SCMP_ACT_LOG",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "arch_prctl",
                "sched_yield",
                "futex",
                "write",
                "mmap",
                "exit_group",
                "madvise",
                "rt_sigprocmask",
                "getpid",
                "gettid",
                "tgkill",
                "rt_sigaction",
                "read",
                "getpgrp"
            ],
            "action": "SCMP_ACT_ALLOW"
        },
        {
            "names": [
                "add_key",
                "keyctl",
                "ptrace"
            ],
            "action": "SCMP_ACT_ERRNO"
        }
    ]
}
```