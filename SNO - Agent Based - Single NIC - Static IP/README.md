# SNO - Agent Based - Single NIC - Static IP

This is an example of the most basic OpenShift deployment possible. This uses the agent based installer to create a single node OpenShift cluster operating on a single NIC.

## Variables

| Variable Name      | Type    | Example            | Description                             |
|--------------------|---------|--------------------|-----------------------------------------|
| `domain`           | string  | `example.com`      | Domain name of OCP cluster              |
| `cluster-name`     | string  | `sno-cluster`      | Desired name of OCP cluster             |
| `machine-subnet`   | string  | `192.168.0.0/24`   | Physical subnet for OCP node            |
| `node-ip`          | string  | `192.168.0.10`     | Desired IP of OCP node                  |
| `node-int`         | string  | `eno1`             | Node interface name                     |
| `node-int-mac`     | string  | `00:25:64:fd:1f:ac`| Node interface MAC address              |
| `dns-ip`           | string  | `192.168.0.2`      | DNS server IP address                   |
| `gwy-ip`           | string  | `192.168.0.1`      | Physical subnet gateway IP address      |

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
  - hostname: master-0.example.com
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