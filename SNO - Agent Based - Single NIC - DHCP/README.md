# SNO - Agent Based - Single NIC - Static IP

This is an example one of the most basic OpenShift deployments possible. This uses the agent based installer to create a single node OpenShift cluster operating on a single NIC using DHCP.

## Variables

| Variable Name      | Type    | Example            | Description                             |
|--------------------|---------|--------------------|-----------------------------------------|
| `cluster-name`     | string  | `sno-cluster`      | Desired name of OCP cluster             |
| `domain`           | string  | `example.com`      | Domain name of OCP cluster              |
| `machine-subnet`   | string  | `192.168.0.0/24`   | Physical subnet for OCP node            |
| `node-int`         | string  | `eno1`             | Node interface name                     |
| `node-int-mac`     | string  | `00:25:64:fd:1f:ac`| Node interface MAC address              |
| `public-ssh-key`   | string  |                    | Public SSH key for SSH access to node   |
| `pull-secret`      | string  |                    | OCP pull secret                         |
| `rendezvous-ip`    | string  | `192.168.0.10`     | IP DHCP will assign to node             |

## Networking Requirements

- DNS
  - api.{{ cluster-name }}.{{ domain }} resolves to {{ rendezvous-ip }}
  - api-int.{{ cluster-name }}.{{ domain }} resolves to {{ rendezvous-ip }}
  - *.apps.{{ cluster-name }}.{{ domain }} resolves to {{ rendezvous-ip }}
  - DNS provider is reachable from {{ physical-subnet }}

- DHCP
  - DHCP server configured for {{ machine subnet }}
  - DHCP server configured to assign {{ rendezvous-ip }} to {{ node-int-mac }}

- Routing
  - {{ physical-subnet }} can route to the internet (or local registry)

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
            dhcp: true
          ipv6:
            enabled: false
```