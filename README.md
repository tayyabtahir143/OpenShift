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

```bash
.
├── 99-add-ssh-key-master.yaml   # For master nodes
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

└── 99-add-ssh-key-worker.yaml   # For worker nodes
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
