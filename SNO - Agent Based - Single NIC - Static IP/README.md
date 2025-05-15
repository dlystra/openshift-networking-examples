# SNO - Agent Based - Single NIC - Static IP

This is an example one of the most basic OpenShift deployments possible. This uses the agent based installer to create a single node OpenShift cluster operating on a single NIC using a static IP.

## Scenario
The organization is installing single node OpenShift cluster with one nic as a sandbox environment. The node interface eno1 has been patched into the top of rack switch and the switchport has been configured as an access port on VLAN 10. The organization has supplied VLAN 10 (192.168.10.0/24) as the OpenShift machine network and designated 192.168.10.10 as the node IP address. The gateway for the 192.168.10.0/24 subnet is 192.168.10.1 and the DNS provider is 192.168.0.2. DHCP is not available in this organization due to security requirements.

![ocp-sno-single-nic](https://github.com/dlystra/openshift-networking-examples/blob/main/SNO%20-%20Agent%20Based%20-%20Single%20NIC%20-%20Static%20IP/ocp-sno-single-nic.png)

## Variables

| Variable Name      | Type    | Example            | Description                             |
|--------------------|---------|--------------------|-----------------------------------------|
| `cluster-name`     | string  | `sno-cluster`      | Desired name of OCP cluster             |
| `dns-ip`           | string  | `192.168.0.2`      | DNS server IP address                   |
| `domain`           | string  | `example.com`      | Domain name of OCP cluster              |
| `gwy-ip`           | string  | `192.168.10.1`      | Physical subnet gateway IP address      |
| `machine-subnet`   | string  | `192.168.10.0/24`   | Physical subnet for OCP node            |
| `node-int`         | string  | `eno1`             | Node interface name                     |
| `node-int-mac`     | string  | `00:25:64:fd:1f:ac`| Node interface MAC address              |
| `node-ip`          | string  | `192.168.10.10`     | Desired IP of OCP node                  |
| `public-ssh-key`   | string  |                    | Public SSH key for SSH access to node   |
| `pull-secret`      | string  |                    | OCP pull secret                         |

## Networking Requirements

- DNS
  - api.sno-cluster.example.com resolves to 192.168.10.10
  - api-int.sno-cluster.example.com resolves to 192.168.10.10
  - *.apps.sno-cluster.example.comresolves to 192.168.10.10
  - DNS provider is reachable from 192.168.10.0/24

- DHCP
  - None

- Routing
  - 192.168.10.0/24 can route to the internet (or local registry) via 192.168.10.1

- Switchport
  - The switch's port is configured as an access port

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
              - ip: 192.168.10.10
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
            next-hop-address: 192.168.10.1
            next-hop-interface: eno1
            table-id: 254
```