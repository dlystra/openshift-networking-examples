# SNO - Agent Based - Dual NIC - Static IP

This is an example one of the most basic OpenShift deployments possible. This uses the agent based installer to create a single node OpenShift cluster operating on separate dual NICs using a static IP. A common use case for this configuration would be enabling OpenShift to access an isolated network that cannot be routed to via the primary NIC.

## Variables

| Variable Name      | Type    | Example            | Description                             |
|--------------------|---------|--------------------|-----------------------------------------|
| `cluster-name`     | string  | `sno-cluster`      | Desired name of OCP cluster             |
| `dns-ip`           | string  | `192.168.0.2`      | DNS server IP address                   |
| `domain`           | string  | `example.com`      | Domain name of OCP cluster              |
| `gwy-ip`           | string  | `192.168.0.1`      | Physical subnet gateway IP address      |
| `machine-subnet`   | string  | `192.168.0.0/24`   | Physical subnet for OCP node            |
| `node-int`         | string  | `eno1`             | Node interface name                     |
| `node-int-mac`     | string  | `00:25:64:fd:1f:ac`| Node interface MAC address              |
| `node-int2`        | string  | `eno2`             | Node secondary interface name           |
| `node-int2-mac`    | string  | `00:25:64:fd:1f:b0`| Node secondary interface MAC address    |
| `node-ip`          | string  | `192.168.0.10`     | Desired IP of OCP node                  |
| `node-ip2`         | string  | `192.168.1.10`     | Desired secondary IP of OCP node        |
| `public-ssh-key`   | string  |                    | Public SSH key for SSH access to node   |
| `pull-secret`      | string  |                    | OCP pull secret                         |

## Networking Requirements

- DNS
  - api.{{ cluster-name }}.{{ domain }} resolves to {{ node-ip }}
  - api-int.{{ cluster-name }}.{{ domain }} resolves to {{ node-ip }}
  - *.apps.{{ cluster-name }}.{{ domain }} resolves to {{ node-ip }}
  - DNS provider is reachable from {{ physical-subnet }}

- DHCP
  - None

- Routing
  - {{ physical-subnet }} can route to the internet (or local registry) via {{ gwy-ip }}

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
  - cidr: 192.168.0.0/24
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
rendezvousIP: 192.168.0.10
hosts:
  - hostname: master-0.sno-cluster.example.com
    interfaces:
      - name: eno1
        macAddress: 00:25:64:fd:1f:ac
    rootDeviceHints:
      deviceName: /dev/sda
    networkConfig:
      interfaces:
        - name: eno1
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:ac
          ipv4:
            enabled: true
            address:
              - ip: 192.168.0.10
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
        - name: eno2
          type: ethernet
          state: up
          mac-address: 00:25:64:fd:1f:b0
          ipv4:
            enabled: true
            address:
              - ip: 192.168.1.10
                prefix-length: 24
            dhcp: false
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - 192.168.0.2
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.0.1
            next-hop-interface: eno1
            table-id: 254
```