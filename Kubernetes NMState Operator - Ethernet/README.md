# Kubernetes NMState Operator - Ethernet

This is an example of adding an addition NIC to an existing OpenShift deployment using the Kubernetes NMState Operator. A common use case for this configuration would be enabling OpenShift to access an isolated network that cannot be routed to via the primary NIC.

## Variables

| Variable Name        | Type    | Example                  | Description                             |
|----------------------|---------|--------------------------|-----------------------------------------|
| `destination-subnet` | string  | `192.168.200.0/24`       | Additional subnet route                 |
| `dns-ip`             | string  | `192.168.100.2`          | Secondary subnet DNS server IP address  |
| `gwy-ip`             | string  | `192.168.100.1`          | Secondary subnet gateway IP address     |
| `node-int`           | string  | `eno2`                   | Node interface name                     |
| `node-ip`            | string  | `192.168.100.10`         | Desired IP of OCP node                  |
| `nodeselector-key`   | string  | `kubernetes.io/hostname` | Key of nodeSelector                     |
| `nodeselector-value` | string  | `master-0`               | Value of nodeSelector                   |

## Networking Requirements

- DHCP
  - DHCP server configured for secondary NIC subnet

- Switchport
  - The switch's port is configured as an access port

## Selecting Nodes

The NodeNetworkConfigurationPolicy is applied to all nodes that match the provided key/value pair for spec.nodeSelector. Common examples:
- kubernetes.io/hostname: master-0
  - Applies configuration to node master-0
- node-role.kubernetes.io/master: ""
  - Applies configuration to all controlplane nodes
- node-role.kubernetes.io/worker: ""
  - Applies configuration to all worker nodes

## Populated Examples

**'ethernet-dhcp.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding additional single NIC interface with DHCP
  name: ethernet-dhcp
spec:
  desiredState:
    interfaces:
      - ipv4:
          dhcp: true
          enabled: true
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
  nodeSelector:
    kubernetes.io/hostname: master-0
```

**'ethernet-static.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding additional single NIC interface with static IP
  name: ethernet-static
spec:
  desiredState:
    interfaces:
      - ipv4:
          enabled: true
          address:
            - ip: 192.168.100.10
              prefix-length: 24
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
  nodeSelector:
    kubernetes.io/hostname: master-0
```

**'ethernet-static-dns-route.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding additional single NIC interface with static IP
  name: ethernet-static-dns-route
spec:
  desiredState:
    interfaces:
      - ipv4:
          enabled: true
          address:
            - ip: 192.168.100.10
              prefix-length: 24
        ipv6:
          enabled: false
        name: eno2
        state: up
        type: ethernet
    dns-resolver:
      config:
        server:
          - 192.168.100.2
    routes:
      config:
        - destination: 192.168.200.0/24
          next-hop-address: 192.168.100.1
          next-hop-interface: eno2
          table-id: 254
  nodeSelector:
    kubernetes.io/hostname: master-0
```