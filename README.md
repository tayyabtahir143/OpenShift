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
        - name: core#Multus Bridge Networking on OpenShift (VLAN-aware)
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
---


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

---

# 3 Multus Bridge Networking on OpenShift (VLAN-aware)

#### Give Pods/VMs plain L2 access to your company VLANs using a Linux bridge on each worker plus Multus. This guide is copy-paste ready and also explains why each piece exists (DHCP daemon, privileged SCC, etc).

## What You Build

* A VLAN-aware Linux bridge on target workers: `br-trunk` (same bridge name on all nodes; uplink name can differ per node).
* NetworkAttachmentDefinitions:

* `native` (untagged/native VLAN via br-trunk)
* `vlan2` (tagged VLAN 2 using Whereabouts static pool)
* `vlan3`, `vlan4` (tagged; DHCP)

* A DHCP CNI daemon as a DaemonSet so `"ipam": "dhcp"` works reliably.

Works for both KubeVirt VMs and plain Pods.

---

## Why This Design (and Alternatives)

* Linux bridge + VLAN filtering: one trunk bridge per node, many VLANs per workload via Multus. Scales better than per-VLAN bridges and avoids per-node NIC-name headaches.

* NAD per VLAN: clear intent and easy selection in Pod/VM specs.
* DHCP or Whereabouts:

* Use DHCP if there’s a real DHCP server on that VLAN.
* Use Whereabouts (static pool IPAM) if you don’t want a daemon or your VLAN has no DHCP.

* Alternatives: OVS, SR-IOV, macvlan. Use those only if you need their specific properties (e.g., SR-IOV latency). For general L2 access, Linux bridge is simpler and portable.

---

## Why the DHCP Daemon Exists

The CNI dhcp plugin is split in two roles:

1. Short-lived CNI call (“ADD”/“DEL”) invoked by CRI-O/kubelet via Multus when a pod sandbox is created.
2. Long-running node-local daemon that:

* Opens a Unix socket at `/run/cni/dhcp.sock`.
* Enters the target network namespace to emit DHCPDISCOVER/DHCPREQUEST and capture the lease.
* Renews the lease periodically (T1/T2) and releases it on DEL.

The CNI ADD must return fast. It cannot sit around to renew leases or keep sockets open. That’s why the dhcp plugin delegates to a daemon via `/run/cni/dhcp.sock`. No daemon = no one to talk to = failure.

---

## Why Privileged / hostNetwork / hostPID

The daemon needs to behave like a tiny system-level agent:

* Enter pod/VM network namespaces to request DHCP on the secondary interface. That requires hostPID to find and nsenter into the sandbox netns, and CAP_SYS_ADMIN/CAP_NET_ADMIN/CAP_NET_RAW (simplest path: `privileged: true` under OpenShift’s SCC model).
* Bind `/run/cni/dhcp.sock` on the host and interact with host paths (`-hostprefix /host`). That’s why we mount the host root (`hostPath: /`) and use hostNetwork so packets originate correctly and the socket is visible to the CNI plugin.
* On OpenShift, the restricted PodSecurity profile doesn’t permit any of this; you must grant the privileged SCC to the service account that runs the DS.

Could you try to trim privileges? Yes, with a custom SCC granting the exact caps + host access you need. The operationally simple option is the privileged SCC for this specific SA.

---

## What Happens If You Use `"ipam": "dhcp"` Without the Daemon

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

---

##Manifests (Copy/Paste)

> Replace the uplink name `enp0s20f0u9u4` with your actual NIC on each node where you want `br-trunk`.

**1) Service Account and SCC (one-time)**

```bash
oc get ns vms >/dev/null 2>&1 || oc create ns vms
oc -n vms create sa cni-dhcp || true
oc adm policy add-scc-to-user privileged -z cni-dhcp -n vms
```
