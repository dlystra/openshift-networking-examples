# Kubernetes NMState Operator - Linux Bridge

This is an example of adding a Linux bridge configuration for OpenShift worker nodes using the Kubernetes NMState Operator. A common use case for this configuration would be enabling OpenShift to provision VMs directly connected to a specific VLAN.

## Assumptions

- Kubernetes NMState Operator is installed and NMState object created
  - [Installing the Kubernetes NMState Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/networking/networking-operators#installing-the-kubernetes-nmstate-operator-cli)

## Scenario

The organization has a OpenShift cluster of six nodes with a single NIC on each node. Each node interface eno1 has been patched into the top of rack switch and the switchport has been configured as an access port on VLAN 10. The organization has supplied VLAN 10 (192.168.10.0/24) as the OpenShift machine network and designated 192.168.10.10-15 as the nodes IP addresses. The gateway for the 192.168.10.0/24 subnet is 192.168.10.1 and the DNS provider is 192.168.0.2. A new requirement for OpenShift virtualization has been created to provision VMs on VLAN 30 (192.168.30.0/24). This will require a Linux bridge on each worker. The eno2 interface on each worker node has been patched into an access port on the top of rack switch configured for VLAN 30.

![ocp-dual-nic-br](https://github.com/dlystra/openshift-networking-examples/blob/main/diagrams/ocp-dual-nic-br.png)

## Variables

| Variable Name        | Type    | Example                          | Description                           |
|----------------------|---------|----------------------------------|---------------------------------------|
| `node-int`           | string  | `eno2`                           | Node interface name                   |
| `nodeselector-key`   | string  | `node-role.kubernetes.io/worker` | Key of nodeSelector                   |
| `nodeselector-value` | string  | `""`                             | Value of nodeSelector                 |

## Networking Requirements

- Switchport
  - The switchport is configured as an access port on VLAN 30

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

**'bridge-eth.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding a Linux bridge to eno2
  name: vm-bridge
spec:
  desiredState:
    interfaces:
      - name: br0
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        ipv6:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eno2
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

## Additional Examples

### Bond interface
This configures eno2 and eno3 as a bond then creates the Linux bridge on that bond interface.

**'bridge-bond.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding a Linux bridge to bond0
  name: vm-bridge
spec:
  desiredState:
    interfaces:
      - name: bond0
        type: bond
        state: up
        ipv4:
          enabled: false
        ipv6:
          enabled: false
        link-aggregation:
          mode: 802.3ad
          options:
            miimon: '100'
            lacp_rate: fast
          port:
            - eno2
            - eno3
      - name: br0
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        ipv6:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: bond0
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

This configures the subinterfaces eno2.30 and eno2.40 then creates Linux bridges on those subinterfaces. This would be used to connect multiple VLANs on a single interface. The switchport would be configured as a trunk in this case.

Subinterface (for trunks)
**'bridge-sub.yaml'**
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    description: adding a Linux bridges and subinterfaces
  name: vm-bridge
spec:
  desiredState:
    interfaces:
      - name: eno2.30
        type: vlan
        state: up
        vlan:
          base-iface: eno2
          id: 30
        ipv4:
          enabled: false
        ipv6:
          enabled: false
      - name: eno2.40
        type: vlan
        state: up
        vlan:
          base-iface: eno2
          id: 40
        ipv4:
          enabled: false
        ipv6:
          enabled: false
      - name: br0.30
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        ipv6:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eno2.30
      - name: br0.40
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        ipv6:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eno2.40
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```