# SNO - Agent Based - Bond - Static IP

This is an example a OpenShift deployment with two NICs forming a bond. This uses the agent based installer to create a single node OpenShift cluster operating on dual NICs to form a single bond. A common use case for this configuration would be to create network redundancy and increase total bandwidth to the OpenShift node.

## Scenario
The organization requires that all physical servers have physical network redundancy to their stacked top of rack switches in the datacenter. This will be accomplished forming a LACP bond from the OpenShift node to the the top of rack switches. The node interfaces eno1 and eno2 have been patched into each switch and configured as a LACP lag (802.3ad) on the switches to form the bond. The organization has supplied VLAN 10 (192.168.10.0/24) as the OpenShift machine network and designated 192.168.10.10 as the node IP address. The gateway for the 192.168.10.0/24 subnet is 192.168.10.1 and the DNS provider is 192.168.0.2. DHCP is not available in this org due to security requirements.

![ocp-sno-bond](https://github.com/dlystra/openshift-networking-examples/blob/main/SNO%20-%20Agent%20Based%20-%20Bond%20-%20Static%20IP/ocp-sno-bond.png)

## Variables

| Variable Name      | Type    | Example            | Description                             |
|--------------------|---------|--------------------|-----------------------------------------|
| `cluster-name`     | string  | `sno-cluster`      | Desired name of OCP cluster             |
| `dns-ip`           | string  | `192.168.0.2`      | DNS server IP address                   |
| `domain`           | string  | `example.com`      | Domain name of OCP cluster              |
| `gwy-ip`           | string  | `192.168.10.1`     | Physical subnet gateway IP address      |
| `lacp-mode`        | string  | `802.3ad`          | Mode of LACP to use                     |
| `machine-subnet`   | string  | `192.168.10.0/24`  | Physical subnet for OCP node            |
| `node-int`         | string  | `eno1`             | Node interface name                     |
| `node-int-mac`     | string  | `00:25:64:fd:1f:ac`| Node interface MAC address              |
| `node-int2`        | string  | `eno2`             | Node secondary interface name           |
| `node-int2-mac`    | string  | `00:25:64:fd:1f:b0`| Node secondary interface MAC address    |
| `node-ip`          | string  | `192.168.10.10`    | Desired IP of OCP node                  |
| `public-ssh-key`   | string  |                    | Public SSH key for SSH access to node   |
| `pull-secret`      | string  |                    | OCP pull secret                         |

## Networking Requirements

- Switchports
  - The switch's ports are configured as access ports
  - The switch's ports are configured as an LACP LAG

- DHCP
  - None

- Routing
  - 192.168.10.0/24 can route to the internet (or local registry) via 192.168.10.1
  - 192.168.10.0/24 can route to the DNS provider (192.168.0.2)

- DNS
  - api.sno-cluster.example.com resolves to 192.168.10.10
  - api-int.sno-cluster.example.com resolves to 192.168.10.10
  - *.apps.sno-cluster.example.com resolves to 192.168.10.10

## Populated Examples

**'install-config.yaml'**
```yaml
apiVersion: v1
baseDomain: example.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 1
metadata:
  name: sno-cluster
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.10.0/24
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{{ pull-secret }}'
sshKey: '{{ public-ssh-key }}'
```

**'agent-config.yaml'**
```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: sno-cluster
rendezvousIP: 192.168.10.10
hosts:
  - hostname: master-0.sno-cluster.example.com
    interfaces:
      - name: eno1
        macAddress: 00:25:64:fd:1f:ac
      - name: eno2
        macAddress: 00:25:64:fd:1f:b0
    rootDeviceHints:
      deviceName: /dev/sda
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:ac
          ipv4:
            enabled: false
          ipv6:
            enabled: false
        - name: eno2
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:b0
          ipv4:
            enabled: false
          ipv6:
            enabled: false
        - name: bond0
          type: bond
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 192.168.10.10
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
          link-aggregation:
            mode: 802.3ad
            options:
              miimon: '100'
              lacp_rate: fast
            port:
              - eno1
              - eno2
      dns-resolver:
        config:
          server:
            - 192.168.0.2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.10.1
            next-hop-interface: bond0
            table-id: 254
```