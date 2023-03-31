# Red Hat CEPH 5 Cluster in AWS

## Prerequiste:

1. AWS Cloud Account.
2. Red Hat Developer Account with active Subscription.
3. OpenSSH or Putty
---

## Step 1: Find RHEL 8 AMI in Marketplace

### AWS Mumbai Region RHEL 8 AMI

```
ami-05c8ca4485f8b138a
```
---

## Step 2: Create a SG - CEPH-sg

Create the SG with three inbound rules
1. SSH         -   22 - My IP (For Secure Communication)
2. Custom TCP  - 8443 - Any IPv4 (Dashboard)
3. All traffic -  All - Self-Source (For all the servers to communicate with each other)
---

## Step 3: Provision the EC2-Instance

Provide the above AMI, SG and root_enable script in User Data field.

root_enable script does the following:
1. Enables `root` SSH with the same AWS Key which is used.
2. Changes the root password - redhat
3. Installs `vim` and `bash-completion`
4. Changes the Bash prompt color.
5. Changes the hostname accordingly. (`servera` - `servera.ceph.lab.com`)
---

## Step 4: Run the Register RHEL file in all the instances

Register RHEL file does the following:
1. Contains username and password of Red Hat Developer Account.
2. Registers the RHEL system.
3. Enables the RHCS repositories.
4. Enables `ceph-5` and `ansible-2.9` repos.
---

## Step 5: Append server details in the /etc/hosts file of servera 

```text
<private_ip_address>	<hostname_FQDN> 	<short_hostname>
172.31.10.150     servera.lab.com         servera
172.31.14.74      serverb.lab.com         serverb
172.31.11.25      serverc.lab.com         serverc
172.31.32.84      grafana.lab.com         grafana
172.31.33.234     clienta.lab.com         clienta
```
---

## Step 6: Provide the hosts file to serverb and client

Using `scp` command we will share the /etc/hosts file to serverb and client.

```code
scp 	/etc/hosts		serverb:/etc/hosts
scp 	/etc/hosts		serverc:/etc/hosts
scp 	/etc/hosts		grafana:/etc/hosts
scp 	/etc/hosts		clienta:/etc/hosts
```
---

## Step 7: Generate private file and copy it serverb and client.

```code
ssh-keygen
ssh-copy-id client
ssh-copy-id servera
ssh-copy-id serverb
```
---


## Step 8: Install `cephadm-ansible` in servera

```code
yum install -y cephadm-ansible
```
---

## Step 9: Create ceph-hosts file

1. Goto /usr/share/cephadm-ansible
2. Create a file named `ceph-hosts`

```code
client.ceph.lab.com
serverb.ceph.lab.com
servera.ceph.lab.com
```
---

## Step 9: Run the ansible-playbook

```code
ansible-playbook -i ceph-hosts cephadm-preflight.yml --extra-vars "ceph_origin=rhcs"
```
---

## Step 10: Run the ceph bootstrap command

```code
cephadm bootstrap                                  \
--apply-spec                   initial_config.yaml \
--registry-url                  registry.redhat.io \
--registry-username                     <USERNAME> \
--registry-password                     <PASSWORD> \
--initial-dashboard-password            <PASSWORD> \
--dashboard-password-noupdate                      \
--ssl-dashboard-port                          8443 \
--mon-ip                      <servera_private_ip> \
--cluster-network 	            <subnet_ipv4_cidr> \
--allow-fqdn-hostname
```

> NOTE: In AWS, `--apply-spec` will be done later because during execution of cephadm bootstrap, it will show an error while trying to add the /etc/ceph/ceph.pub key to serverb and client.
---

## Step 11: Copy public key to serverb and client

```code
ssh-copy-id -f -i /etc/ceph/ceph.pub serverb
ssh-copy-id -f -i /etc/ceph/ceph.pub client
```
---

## Step 12: Create the initial_config.yaml

```yaml
---
service_type: host
addr: <SERVERA_PRIVATE_IP>
hostname: servera.lab.com
---
service_type: host
addr: <SERVERB_PRIVATE_IP>
hostname: serverb.lab.com
---
service_type: host
addr: <SERVERC_PRIVATE_IP>
hostname: serverc.lab.com
---
service_type: host
addr: <CLIENTA_PRIVATE_IP>
hostname: clienta.lab.com
---
service_type: host
addr: <GRAFANA_PRIVATE_IP>
hostname: grafana.lab.com
---
service_type: mon
placement:
  hosts: 
    - servera.lab.com
    - serverb.lab.com
    - serverc.lab.com
    - clienta.lab.com
---
service_type: mgr
placement:
  hosts: 
    - servera.lab.com
    - serverb.lab.com
    - serverc.lab.com
    - clienta.lab.com
---
service_type: osd
service_id: default_drive_group
placement:
  hosts: 
    - servera.lab.com
    - serverb.lab.com
    - serverc.lab.com
data_devices:
  paths:
    - /dev/xvdb
    - /dev/xvdc
    - /dev/xvdd
---
service_type: grafana
service_name: grafana
placement:
  count: 1
  hosts:
    - grafana.lab.com
spec:
  initial_admin_password: redhat
  port: 3000
...
```

> NOTE: DO NOT PROVIDE `---` in the last, it will throw as TypeError.
> NOTE: `.yaml` starts with `---` and ends with `...`.
---

## Step 13: Apply the initial_config.yaml

First, dry run the config file.
```code
ceph orch apply -i initial_config.yaml --dry-run
```

Then apply it if there is no error.
```code
ceph orch apply -i initial_config.yaml --dry-run
```

> NOTE: This will take some time to configure. 
---

## Step 14: Copy the ceph.conf and keyring files to client

```code
cd /etc/ceph

scp {ceph.conf, ceph.client.admin.keyring} client:/etc/ceph
```

## Step 15: Add _admin label to client

```code
ceph orch host label add client.ceph.example.com _admin
```

## Step 16: Check the health of the Cluster

1. First ssh to clienta system.
   
```code
ssh clienta
```

2. Check the health of the CEPH Cluster.

```code
ceph -s
ceph status
```
svsvsv
