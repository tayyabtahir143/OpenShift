# 1. 🔐 Add SSH Public Keys to OpenShift Master & Worker Nodes

This documentation explains how to securely inject SSH public keys into **master** and **worker** nodes of an OpenShift cluster using `MachineConfig` resources. This works for all deployment types: **IPI, UPI, and Bare Metal**.

---

## 🧠 Why Use MachineConfig?

Using a MachineConfig ensures:

- Your SSH key is persistently added to RHCOS nodes
- MCO handles rolling out the change safely
- It applies to all current and future nodes in the pool

---

## 📁 Files Included

├── ssh-key-master.yaml   # For master nodes<br>
└── ssh-key-worker.yaml   # For worker nodes

---

✍️ Step 1: Create master conifg:<br>
vim ssh-key-master.yaml
```bash

apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-add-ssh-key-master
  labels:
    machineconfiguration.openshift.io/role: master
spec:
  config:
    ignition:
      version: 3.2.0
    passwd:
      users:
        - name: core
          sshAuthorizedKeys:
            - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD... your-public-key-here ...
```
Vim ssh-key-worker.yaml
```bash
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-add-ssh-key-worker
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.2.0
    passwd:
      users:
        - name: core
          sshAuthorizedKeys:
            - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD... your-public-key-here ...

```

---

📦 Step 2: Apply it
```bash
oc apply -f ssh-key-worker.yaml
oc apply -f ssh-key-master.yaml

```
---

⏱️ Step 3: Watch the rollout
```bash
oc get mcp -w
```
Once you see:
```bash
worker   UPDATED=False   UPDATING=True
```
You’re golden 🌟

---

🧪 Step 4: Confirm Key Access
Wait for nodes to update, then SSH in:

```bash
ssh core@worker1
```



# 2. 🌐 Fix Linux DNS “search suffix” side-effects (ndots + doubled names) — copy-paste ready 

Problem in one line: Linux name resolution can silently append a search suffix (e.g. BM.tayyabtahir.com) to what you typed, so oauth-openshift.apps.bm.tayyabtahir.com gets queried as oauth-openshift.apps.bm.tayyabtahir.com.BM.tayyabtahir.com. If your public DNS has a wildcard, that “wrong but valid” name resolves to a public IP → OpenShift routes fail with tls: unrecognized name.

Why it happens: glibc resolver + search domain + ndots rules. If a name has fewer dots than ndots, glibc tries name + search first. Even without an explicit search, glibc can derive one from your host’s FQDN. Public *.domain makes the mistake look like success.

What we’ll do: Tell NetworkManager to stop managing /etc/resolv.conf, then write a minimal resolver with no search list and ndots:1, pointing at your internal DNS. In this environment, doing this on the DNS server host was enough because it’s the choke point for lookups.

Copy-paste fix (replace 192.168.8.10 with your DNS IP):


```bash
# Stop NetworkManager from managing resolv.conf
mkdir -p /etc/NetworkManager/conf.d
cat >/etc/NetworkManager/conf.d/dns-none.conf <<'EOF'
[main]
dns=none
EOF
systemctl restart NetworkManager

# Write a minimal resolver (no search, try literal names first)
cat >/etc/resolv.conf <<'EOF'
nameserver 192.168.8.10
options ndots:1
EOF
```

How to verify quickly:

```bash
# Should show exactly the two lines you wrote
cat /etc/resolv.conf

# The absolute form (trailing dot) must resolve internally
getent hosts oauth-openshift.apps.bm.tayyabtahir.com.

# The non-absolute form should also resolve to your internal VIP now
getent hosts oauth-openshift.apps.bm.tayyabtahir.com
```

If suffixing still appears: your host’s FQDN is supplying a default domain. Either make the hostname short or force an “empty” search.

```bash
# Option A: short hostname (no dots)
hostnamectl set-hostname bastian

# Option B: explicitly set root search (prevents meaningful suffixing)
nmcli con mod ens33 ipv4.dns-search "." ipv6.dns-search "."
nmcli con down ens33 && nmcli con up ens33

```


OpenShift sanity checks (once DNS is sane):


```bash

oc get co authentication console -o wide   # auth/console should recover
POD=$(oc -n openshift-console get pod -l app=console -o jsonpath='{.items[0].metadata.name}')
BASE=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')
oc exec -n openshift-console "$POD" -c console -- \
  curl -sI --max-time 8 --cacert /var/oauth-serving-cert/ca-bundle.crt \
  https://oauth-openshift."$BASE"/healthz   # expect HTTP/1.1 200 OK
```

Notes you might need:

Pods have their own resolver. If a workload still plays games with suffixes, set pod DNS options:

```yaml
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "1"
```
or env var: RES_OPTIONS=ndots:1.





If you must keep a search domain on clients but want to “catch” doubled names at your dnsmasq, add VIP answers:


```ini
# /etc/dnsmasq.conf
address=/.apps.bm.tayyabtahir.com/192.168.8.28
address=/apps.bm.tayyabtahir.com.bm.tayyabtahir.com/192.168.8.28
systemctl restart dnsmasq
```
Reducing/avoiding public wildcards (*.bm.tayyabtahir.com) removes a whole class of “wrong but valid” answers.




Rollback (if you ever want NM to manage DNS again):
```bash
rm -f /etc/NetworkManager/conf.d/dns-none.conf
systemctl restart NetworkManager
# /etc/resolv.conf will be regenerated from DHCP/NM

```



FAQ

Q: Why not just set ndots:1 and keep search?
A: You can. It makes the resolver try the literal name first. But if the first try fails or your app uses short names, the search domain still applies. Removing NM control of resolv.conf and omitting the search list eliminates surprises.

Q: Why did doing this only on the DNS server fix my cluster?
A: In this environment, the DNS server host was the choke point that ultimately answered (or forwarded) app/route lookups. Ensuring it never appended/forwarded accidental, doubled names prevented the wrong answers from being cached/propagated. In other topologies you may also need to adjust clients/pods.

Q: What about public wildcard DNS?
A: If feasible, narrow or remove public wildcards (*.bm.tayyabtahir.com). That stops “wrong but valid” answers from the Internet when a doubled name escapes your network.



# 3 Multus Bridge Networking on OpenShift (VLAN-aware)

####Give Pods/VMs plain L2 access to your company VLANs using a Linux bridge on each worker plus Multus. This guide is copy-paste ready and also explains why each piece exists (DHCP daemon, privileged SCC, etc).

##What You Build

  * A VLAN-aware Linux bridge on target workers: `br-trunk` (same bridge name on all nodes; uplink name can differ per node).

  * NetworkAttachmentDefinitions:

      * `native` (untagged/native VLAN via br-trunk)
      * `vlan2` (tagged VLAN 2 using Whereabouts static pool)
      * `vlan3`, `vlan4` (tagged; DHCP)

  * A DHCP CNI daemon as a DaemonSet so `"ipam": "dhcp"` works reliably.

Works for both KubeVirt VMs and plain Pods.


##Why This Design (and Alternatives)

* Linux bridge + VLAN filtering: one trunk bridge per node, many VLANs per workload via Multus. Scales better than per-VLAN bridges and avoids per-node NIC-name headaches.

* NAD per VLAN: clear intent and easy selection in Pod/VM specs.
* DHCP or Whereabouts:

    * Use DHCP if there’s a real DHCP server on that VLAN.
    * Use Whereabouts (static pool IPAM) if you don’t want a daemon or your VLAN has no DHCP.

* Alternatives: OVS, SR-IOV, macvlan. Use those only if you need their specific properties (e.g., SR-IOV latency). For general L2 access, Linux bridge is simpler and portable.



##Why the DHCP Daemon Exists

The CNI dhcp plugin is split in two roles:

1. Short-lived CNI call (“ADD”/“DEL”) invoked by CRI-O/kubelet via Multus when a pod sandbox is created.
2. Long-running node-local daemon that:

    * Opens a Unix socket at `/run/cni/dhcp.sock`.
    * Enters the target network namespace to emit DHCPDISCOVER/DHCPREQUEST and capture the lease.
    * Renews the lease periodically (T1/T2) and releases it on DEL.

The CNI ADD must return fast. It cannot sit around to renew leases or keep sockets open. That’s why the dhcp plugin delegates to a daemon via `/run/cni/dhcp.sock`. No daemon = no one to talk to = failure.



##Why Privileged / hostNetwork / hostPID

The daemon needs to behave like a tiny system-level agent:

    * Enter pod/VM network namespaces to request DHCP on the secondary interface. That requires hostPID to find and nsenter into the sandbox netns, and CAP_SYS_ADMIN/CAP_NET_ADMIN/CAP_NET_RAW (simplest path: `privileged: true` under OpenShift’s SCC model).

    * Bind `/run/cni/dhcp.sock` on the host and interact with host paths (`-hostprefix /host`). That’s why we mount the host root (`hostPath: /`) and use hostNetwork so packets originate correctly and the socket is visible to the CNI plugin.

    * On OpenShift, the restricted PodSecurity profile doesn’t permit any of this; you must grant the privileged SCC to the service account that runs the DS.

Could you try to trim privileges? Yes, with a custom SCC granting the exact caps + host access you need. The operationally simple option is the privileged SCC for this specific SA.



##What Happens If You Use `"ipam": "dhcp"` Without the Daemon

    * Multus calls the CNI dhcp plugin during pod sandbox creation.
    * The plugin tries to dial `/run/cni/dhcp.sock`.
    * If the daemon isn’t running, you get events like:

```vbnet
Failed to create pod sandbox: ... 
error adding pod ... to CNI network "multus-cni-network":
... error dialing DHCP daemon: dial unix /run/cni/dhcp.sock: no such file or directory

```

    * Result: the pod sandbox isn’t created, your virt-launcher for a VM sits in ContainerCreating, and the VMI never reaches Running.

If you don’t want the daemon, don’t use DHCP IPAM. Use Whereabouts (static pool) or explicit static IP config.


##Manifests (Copy/Paste)

> Replace the uplink name `enp0s20f0u9u4` with your actual NIC on each node where you want `br-trunk`.

**1) Service Account and SCC (one-time)**

```bash
oc get ns vms >/dev/null 2>&1 || oc create ns vms
oc -n vms create sa cni-dhcp || true
oc adm policy add-scc-to-user privileged -z cni-dhcp -n vms
```

**2) VLAN-aware Linux bridge on worker3 (repeat per node with different uplink)**
```yaml
# br-trunk-worker3.yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br-trunk-worker3
spec:
  nodeSelector:
    kubernetes.io/hostname: worker3
  desiredState:
    interfaces:
    - name: br-trunk
      type: linux-bridge
      state: up
      bridge:
        options:
          stp: { enabled: false }
          vlan: { filtering: true }
        port:
        - name: enp0s20f0u9u4          # <-- CHANGE per node
    - name: ENS_UPLINK
      type: ethernet
      state: up
      ipv4: { enabled: false }
      ipv6: { enabled: false }
```

**3) NetworkAttachmentDefinitions**
```yaml
# nad-native.yaml (untagged/native VLAN via br-trunk, DHCP)
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: native
  namespace: vms
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br-trunk",
      "ipam": { "type": "dhcp" }
    }
---
# nad-vlan2.yaml (VLAN 2 via Whereabouts static pool)
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan2
  namespace: vms
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br-trunk",
      "vlan": 2,
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.2.0/24",
        "range_start": "192.168.2.100",
        "range_end": "192.168.2.150",
        "gateway": "192.168.2.1"
      }
    }
---
# nad-vlan3.yaml (VLAN 3 via DHCP)
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan3
  namespace: vms
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br-trunk",
      "vlan": 3,
      "ipam": { "type": "dhcp" }
    }
---
# nad-vlan4.yaml (VLAN 4 via DHCP)
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan4
  namespace: vms
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br-trunk",
      "vlan": 4,
      "ipam": { "type": "dhcp" }
    }
```

**4) DHCP CNI Daemon (uses host’s dhcp plugin)**
```yaml
# dhcp-daemon.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cni-dhcp-daemon
  namespace: vms
spec:
  selector:
    matchLabels:
      app: cni-dhcp-daemon
  template:
    metadata:
      labels:
        app: cni-dhcp-daemon
    spec:
      serviceAccountName: cni-dhcp
      hostNetwork: true
      hostPID: true
      nodeSelector:
        node-role.kubernetes.io/worker: ""   # only on workers
      tolerations:
      - operator: Exists
      priorityClassName: system-node-critical
      containers:
      - name: dhcp
        image: registry.access.redhat.com/ubi9/ubi-minimal:latest
        command: ["/usr/bin/bash","-lc"]
        args: ["/host/usr/libexec/cni/dhcp daemon -hostprefix /host"]
        securityContext:
          privileged: true
          seccompProfile:
            type: RuntimeDefault
        volumeMounts:
        - name: host-root
          mountPath: /host
      volumes:
      - name: host-root
        hostPath:
          path: /
```

> If ` /usr/libexec/cni/dhcp`  isn’t present on your nodes, use `registry.redhat.io/openshift4/ose-cni-plugins:<cluster-minor>` as the image and exec `/usr/libexec/cni/dhcp` from there (requires pull access). The host-binary approach avoids pulls.


**Apply in Order**
```bash
# Namespace + SA/SCC (once)
oc get ns vms >/dev/null 2>&1 || oc create ns vms
oc -n vms create sa cni-dhcp || true
oc adm policy add-scc-to-user privileged -z cni-dhcp -n vms
oc auth can-i use scc/privileged --as "system:serviceaccount:vms:cni-dhcp"

# Bridge on worker(s)
oc apply -f br-trunk-worker3.yaml
oc get nncp
oc get nnce -A | grep br-trunk
oc debug node/worker3 -- chroot /host ip link show br-trunk
oc debug node/worker3 -- chroot /host bridge -c vlan show dev br-trunk

# NADs
oc apply -f nad-native.yaml
oc apply -f nad-vlan2.yaml
oc apply -f nad-vlan3.yaml
oc apply -f nad-vlan4.yaml
oc -n vms get net-attach-def

# DHCP daemon (only if using ipam: dhcp)
oc apply -f dhcp-daemon.yaml
oc -n vms get pods -l app=cni-dhcp-daemon -o wide

# Socket should exist:
oc debug node/worker3 -- chroot /host ls -l /run/cni/dhcp.sock
```

**Verify & Smoke Tests**
**Pod on VLAN 3 (DHCP)**
```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: netsmoke-vlan3
  namespace: vms
  annotations:
    k8s.v1.cni.cncf.io/networks: vms/vlan3
spec:
  nodeSelector:
    kubernetes.io/hostname: worker3
  containers:
  - name: sh
    image: quay.io/cybozu/ubuntu:22.04
    command: ["bash","-lc","sleep infinity"]
  restartPolicy: Never
EOF

oc -n vms exec netsmoke-vlan3 -- ip -4 a
oc -n vms exec netsmoke-vlan3 -- ping -c2 <vlan3-gateway-ip>
```

**Pod on VLAN 2 (Whereabouts static)**
```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: netsmoke-vlan2
  namespace: vms
  annotations:
    k8s.v1.cni.cncf.io/networks: vms/vlan2
spec:
  nodeSelector:
    kubernetes.io/hostname: worker3
  containers:
  - name: sh
    image: quay.io/cybozu/ubuntu:22.04
    command: ["bash","-lc","sleep infinity"]
  restartPolicy: Never
EOF

oc -n vms exec netsmoke-vlan2 -- ip -4 a
oc -n vms exec netsmoke-vlan2 -- ping -c2 192.168.2.1
```

**Attach to a VM (KubeVirt)**
Add an interface with bridge binding and reference a NAD:
```yaml
spec:
  template:
    spec:
      interfaces:
        - name: vlan3
          bridge: {}
          model: virtio
      networks:
        - name: vlan3
          multus:
            networkName: vms/vlan3
```

Then confirm on the virt-launcher pod:
```bash
POD=$(oc get pod -n vms -l kubevirt.io=virt-launcher,vm=<VM_NAME> -o jsonpath='{.items[0].metadata.name}')
oc get pod -n vms "$POD" -o json | jq -r '.metadata.annotations["k8s.v1.cni.cncf.io/networks-status"]'
```

**Troubleshooting**
**Pod/VM stuck with DHCP error**
Symptoms:
    * Pod sandbox creation fails.
    * Events show:
```arduino
error dialing DHCP daemon: dial unix /run/cni/dhcp.sock: connect: no such file or directory
```
Fix:
    * Ensure DS is Running on the node:
      `oc -n vms get pods -l app=cni-dhcp-daemon -o wide`
    * Ensure socket exists:
      `oc debug node/worker3 -- chroot /host ls -l /run/cni/dhcp.sock`
    * If pulling an image fails, switch to host-binary DS (this README’s version) or fix registry pull.

**No IP on Whereabouts NAD**
    * Check the pool range/gateway.
    * Look for IPAM logs in the pod’s annotations and events:
      `oc describe pod <pod> | sed -n '/Events:/,$p'`
**No traffic on VLAN**
    * Upstream trunk missing VLAN.
    * Bridge VLAN filtering not enabled.
    * vSphere PG security not set to Accept (Promiscuous/MAC Changes/Forged Transmits).
Check on the node:
      `oc debug node/worker3 -- chroot /host bridge -c vlan show dev br-trunk`

**VM stuck Scheduling (local storage)**
    * Local PV node-affinity conflict (e.g., LVMS RWO must schedule to the node that holds the PV).
    * Node taints / insufficient resources.

**Security Notes**
    * Scope the DHCP DS to workers only (`nodeSelector`) and a dedicated SA with the privileged SCC.
    * The DS mounts the host root at `/host` to interact with `/run`, netns, and the host’s dhcp plugin binary. Keep this in a separate namespace (`vms`) and use only where needed.
    * If you want to avoid privileged pods entirely, use Whereabouts instead of DHCP.

**Cleanup**
```bash
oc -n vms delete net-attach-def native vlan2 vlan3 vlan4
oc delete nncp br-trunk-worker3
oc -n vms delete ds cni-dhcp-daemon
oc -n vms delete sa cni-dhcp
```

**Final Notes**
    * Standardize the bridge name (`br-trunk`) across nodes; uplink names can differ per node NNCP.
    * For VM live migration: every candidate node must have the same bridge/VLANs and the VM must use shared RWX storage (local LVMS RWO cannot live-migrate).









