# Kubernetes NMState Operator - Ethernet - Static IP

This is an example of adding an addition NIC to an existing OpenShift deployment using the Kubernetes NMState Operator. A common use case for this configuration would be enabling OpenShift to access an isolated network that cannot be routed to via the primary NIC.

## Assumptions

- Kubernetes NMState Operator is installed and NMState object created
  - [Installing the Kubernetes NMState Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/networking/networking-operators#installing-the-kubernetes-nmstate-operator-cli)

## Scenario

The organization has a OpenShift cluster of six nodes with a single NIC on each node. Each node interface eno1 has been patched into the top of rack switch and the switchport has been configured as an access port on VLAN 10. The organization has supplied VLAN 10 (192.168.10.0/24) as the OpenShift machine network and designated 192.168.10.10-15 as the nodes IP addresses. The gateway for the 192.168.10.0/24 subnet is 192.168.10.1 and the DNS provider is 192.168.0.2. A new requirement for an isolated storage network has been defined post-deployment. The other node interface eno2 has been patched into a separate switch on an isolated storage network. This switchport is configured as an access port on VLAN 20. 192.168.20.10-15 has been designated as the node IP addresses for the storage network. There is no gateway or DNS requirement for VLAN 20. DHCP is not available in this organization due to security requirements.

![ocp-dual-nic](https://github.com/dlystra/openshift-networking-examples/blob/main/diagrams/ocp-dual-nic.png)

## Variables

| Variable Name        | Type    | Example                  | Description                           |
|----------------------|---------|--------------------------|---------------------------------------|
| `node-int`           | string  | `eno2`                   | Node interface name                   |
| `node-ip`            | string  | `192.168.20.10`          | Desired secondary IP of node          |
| `nodeselector-key`   | string  | `kubernetes.io/hostname` | Key of nodeSelector                   |
| `nodeselector-value` | string  | `master-0`               | Value of nodeSelector                 |

## Networking Requirements

- Switchport
  - The switchport is configured as an access port on VLAN 20

- DHCP
  - None

- Routing
  - None

- DNS
  - None

## Selecting Nodes

The NodeNetworkConfigurationPolicy is applied to all nodes that match the provided key/value pair for spec.nodeSelector. Common examples:
- kubernetes.io/hostname: master-0
  - Applies configuration to node master-0
- node-role.kubernetes.io/master: ""
  - Applies configuration to all controlplane nodes
- node-role.kubernetes.io/worker: ""
  - Applies configuration to all worker nodes

## Populated Example

Since each node has a unique static IP address a NodeNetworkConfigurationPolicy is required for each node.

**'master-0-eno2.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding an additional nic with static IP
  name: master-0-eno2
spec:
  desiredState:
    interfaces:
      - ipv4:
          enabled: true
          address:
            - ip: 192.168.20.10
              prefix-length: 24
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
  nodeSelector:
    kubernetes.io/hostname: master-0
```

**'master-1-eno2.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding an additional nic with static IP
  name: master-1-eno2
spec:
  desiredState:
    interfaces:
      - ipv4:
          enabled: true
          address:
            - ip: 192.168.20.11
              prefix-length: 24
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
  nodeSelector:
    kubernetes.io/hostname: master-1
```

**'master-2-eno2.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding an additional nic with static IP
  name: master-2-eno2
spec:
  desiredState:
    interfaces:
      - ipv4:
          enabled: true
          address:
            - ip: 192.168.20.12
              prefix-length: 24
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
  nodeSelector:
    kubernetes.io/hostname: master-2
```

**'worker-0-eno2.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding an additional nic with static IP
  name: worker-0-eno2
spec:
  desiredState:
    interfaces:
      - ipv4:
          enabled: true
          address:
            - ip: 192.168.20.13
              prefix-length: 24
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
  nodeSelector:
    kubernetes.io/hostname: worker-0
```

**'worker-1-eno2.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding an additional nic with static IP
  name: worker-1-eno2
spec:
  desiredState:
    interfaces:
      - ipv4:
          enabled: true
          address:
            - ip: 192.168.20.14
              prefix-length: 24
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
  nodeSelector:
    kubernetes.io/hostname: worker-1
```

**'worker-2-eno2.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding an additional nic with static IP
  name: add-nic-eno2
spec:
  desiredState:
    interfaces:
      - ipv4:
          enabled: true
          address:
            - ip: 192.168.20.15
              prefix-length: 24
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
  nodeSelector:
    kubernetes.io/hostname: worker-2
```