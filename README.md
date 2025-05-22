# 🔐 Add SSH Public Keys to OpenShift Master & Worker Nodes

This documentation explains how to securely inject SSH public keys into **master** and **worker** nodes of an OpenShift cluster using `MachineConfig` resources. This works for all deployment types: **IPI, UPI, and Bare Metal**.

---

## 🧠 Why Use MachineConfig?

Using a MachineConfig ensures:

- Your SSH key is persistently added to RHCOS nodes
- MCO handles rolling out the change safely
- It applies to all current and future nodes in the pool

---

## 📁 Files Included
.
├── ssh-key-master.yaml   # For master nodes
└── ssh-key-worker.yaml   # For worker nodes

---

✍️ Step 1: Create master conifg:

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

