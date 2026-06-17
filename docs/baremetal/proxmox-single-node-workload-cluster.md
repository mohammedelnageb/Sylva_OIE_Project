# Proxmox Single-Node Workload Cluster With CAPM3

This document explains how to continue after the Sylva management cluster is deployed and create a single-node workload cluster for the O-CU/O-DU showcase.

The reference OCUDU articles use physical servers or libvirt with a fake Redfish BMC. In this lab, the same Sylva/CAPM3/Metal3 model is adapted to Proxmox by using ProxmoxBMC and another Proxmox VM as a fake bare-metal node.

## Target Topology

```text
Proxmox host
|
|-- Bootstrap VM
|   |-- sylva-core
|   |-- kubectl
|   |-- ProxmoxBMC
|
|-- Management cluster VM
|   |-- Sylva management cluster
|   |-- Rancher / Flux / CAPI / CAPM3 / Metal3
|
|-- Workload VM
    |-- provisioned by Sylva management cluster
    |-- single-node RKE2 workload cluster
    |-- namespace: oran
        |-- O-CU
        |-- O-DU
```

The management cluster node and workload cluster node must be different Proxmox VMs.

## Important Lab Values

Use different values from the management cluster.

```text
Management cluster API VIP: 10.237.71.56
Management node IP:        10.237.71.25
Management BMC port:       6625

Workload cluster API VIP:  10.237.71.104
Workload node IP:          10.237.71.105
Workload BMC port:         6626
Workload boot MAC:         BC:24:11:4F:26:BB
```

Do not reuse the management cluster VM, MAC address, node IP, API VIP, or BMC port for the workload cluster.

## Step 1: Create A Workload Values Folder

Run this on the bootstrap VM:

```bash
cd ~/sylva-core

mkdir -p environment-values/ocudu-workload-rke2-capm3
cp environment-values/my-rke2-capm3/values.yaml environment-values/ocudu-workload-rke2-capm3/values.yaml
cp environment-values/my-rke2-capm3/secrets.yaml environment-values/ocudu-workload-rke2-capm3/secrets.yaml
```

Use the management cluster values only as a starting point. The workload values must use the workload VM IPs, MAC address, and ProxmoxBMC port.

## Step 2: Configure The Workload Values

Edit:

```bash
nano environment-values/ocudu-workload-rke2-capm3/values.yaml
```

Use this structure:

```yaml
# Single-node workload cluster for O-CU/O-DU using CAPM3 + ProxmoxBMC.

cluster_virtual_ip: 10.237.71.104

cluster:
  capi_providers:
    infra_provider: capm3
    bootstrap_provider: cabpr

  control_plane_replicas: 1

  rke2:
    additionalUserData:
      config:
        users:
          - name: sylva-user
            groups: users,sylva-ops
            sudo: ALL=(ALL) NOPASSWD:ALL
            shell: /bin/bash
            lock_passwd: false
            passwd: "<generated-password-hash>"
            ssh_authorized_keys:
              - "<your-ssh-public-key>"

  capm3:
    os_image_selector:
      os: ubuntu
      hardened: true
    networks:
      primary:
        subnet: 10.237.71.0/24
        gateway: 10.237.71.254
        start: 10.237.71.105
        end: 10.237.71.105
    dns_servers:
      - 10.237.25.2
      - 8.8.8.8

  control_plane:
    capm3:
      hostSelector:
        matchLabels:
          cluster-role: control-plane
      networks:
        primary:
          interface: ens18

    network_interfaces:
      ens18:
        type: phy

  machine_deployment_default:
    capm3:
      hostSelector:
        matchLabels:
          cluster-role: worker
      networks:
        primary:
          interface: ens18

  machine_deployments: {}

  baremetal_host_default:
    bmh_spec:
      externallyProvisioned: false
      bmc:
        disableCertificateVerification: true
      bootMode: legacy

  baremetal_hosts:
    ocudu-workload-node:
      bmh_metadata:
        labels:
          cluster-role: control-plane
          host-type: generic
      bmh_spec:
        description: single-node O-CU/O-DU workload cluster
        bmc:
          address: ipmi://172.18.0.1:6626
          credentialsName: ocudu-workload-bmc-secret
          disableCertificateVerification: true
        bootMACAddress: "BC:24:11:4F:26:BB"
        rootDeviceHints:
          deviceName: /dev/sda

units:
  longhorn:
    enabled: false
  local-path-provisioner:
    enabled: true

metal3:
  bootstrap_ip: 10.237.71.153

proxies:
  http_proxy: ""
  https_proxy: ""
  no_proxy: ""

ntp:
  servers:
    - "0.pool.ntp.org"
    - "1.pool.ntp.org"
```

## Step 3: Check The BMC Secret

The values above use:

```yaml
credentialsName: ocudu-workload-bmc-secret
```

The copied `secrets.yaml` must include a matching secret. If the same ProxmoxBMC username and password are used, either create a new secret with this name or reuse the existing secret name from the management cluster values.

Example simple option:

```yaml
credentialsName: fake-bm-01-bmc-secret
```

Use this only if the same BMC credentials are valid for the workload ProxmoxBMC process.

## Step 4: Start A Second ProxmoxBMC Process

Keep the existing management ProxmoxBMC process on port `6625`.

Start another ProxmoxBMC process for the workload VM:

```text
management VM BMC -> 6625
workload VM BMC   -> 6626
```

The exact command depends on how ProxmoxBMC was installed, but the important change is:

```text
port = 6626
vmid = workload VM ID
```

Check the existing command:

```bash
ps aux | grep -i proxmox
```

Then start the second process using the workload VM ID and port `6626`.

## Step 5: Test The Workload BMC

From the bootstrap VM or from a pod that can reach the Docker bridge address:

```bash
ipmitool -I lanplus -H 172.18.0.1 -p 6626 -U admin -P password power status
```

Expected result:

```text
Chassis Power is on
```

or:

```text
Chassis Power is off
```

If this fails, fix ProxmoxBMC before running the workload cluster deployment.

## Step 6: Deploy The Workload Cluster

Workload cluster deployment must run against the management cluster kubeconfig:

```bash
export KUBECONFIG=/tmp/management-cluster.kubeconfig

cd ~/sylva-core
./apply-workload-cluster.sh environment-values/ocudu-workload-rke2-capm3
```

This is different from the management cluster bootstrap flow:

```text
Management cluster deployment -> bootstrap Kind context
Workload cluster deployment   -> management cluster kubeconfig
```

## Step 7: Validate The Workload Cluster

```bash
export KUBECONFIG=/tmp/management-cluster.kubeconfig

kubectl get clusters -A
kubectl get machines -A
kubectl get baremetalhosts -A
kubectl get kustomizations -A | grep -E "ocudu|False|Unknown|InProgress"
kubectl get helmreleases -A | grep -E "ocudu|False|Unknown|Progressing"
```

Expected signs:

```text
BareMetalHost is provisioned
Machine is Ready
Cluster is Ready
Workload node receives 10.237.71.105
```

## Step 8: Prepare For O-CU/O-DU

After the workload cluster is ready, get its kubeconfig from Sylva/Rancher or the generated cluster secret, then create the O-RAN namespace:

```bash
kubectl create namespace oran
```

For a single-node workload cluster, allow workloads on the control-plane node if needed:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

If the taint does not exist, the command can be ignored.

The final showcase layout is:

```text
namespace: oran
|
|-- O-CU
|-- O-DU
```

## Key Rules

- Do not use the article values directly.
- Use the article only as a structure reference.
- Keep your Proxmox lab IPs and ProxmoxBMC settings.
- Use a new Proxmox VM for the workload cluster.
- Use a new BMC port, such as `6626`.
- Use the workload VM MAC address: `BC:24:11:4F:26:BB`.
- Keep the management cluster and workload cluster on separate VMs.
- Use the management cluster kubeconfig when deploying workload clusters.
