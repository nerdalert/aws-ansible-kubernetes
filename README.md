# Kubernetes Multi-Node Ansible Deployment for AWS EC2

This repo fully automates the deployment of Kubernetes on EC2 and can scale out dynamically to however
many worker nodes desired. The playbooks use the latest changes to the `amazon.aws.ec2_instance` module.

Currently, the Kubernetes distribution is [K3s](https://github.com/k3s-io/k3s) as I am using this for datapath
performance testing, so the lighter the weight the better for my needs. I will be adding Microshift as an
alternative lightweight distribution as soon as this PR merges [Allow MicroShift to join new worker nodes](https://github.com/redhat-et/microshift/pull/471).

### Prerequisites

- This assumes little to no experience with Kubernetes or Ansible.
- An active AWS account. The default profiles and VPCs are sufficient. The default instance type used
  is `t2.micro` which is free tier eligible. The default AMI image used is Fedora but can be changed to
  any Linux flavor in the ENV file shown in the next section.
- Python - (newer the version the better) latest version can be found at [Download the latest version of Python](https://www.python.org/downloads/)
- Ansible - instructions in next section or at [Installing Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible)


### Install Ansible and Clone the Repo


- Install Ansible and Boto

```sh
# Details at https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html
$ /usr/local/bin/python3.10 -m pip install --user ansible
# boto is the AWS SDK for python
$ pip3 install boto

# example ansible output on OSX
$ ansible --version
ansible [core 2.12.3]
  configured module search path = ['/Users/brent/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /Users/brent/Library/Python/3.10/lib/python/site-packages/ansible
  ansible collection location = /Users/brent/.ansible/collections:/usr/share/ansible/collections
  executable location = /Users/brent/Library/Python/3.10/bin/ansible
  python version = 3.10.3 (v3.10.3:a342a49189, Mar 16 2022, 09:34:18) [Clang 13.0.0 (clang-1300.0.29.30)]
  jinja version = 3.0.3
  libyaml = True
```

- If you are on OSX you may need to run the following to resolve openSSL root certificate access (replace 3.10 with whatever your Python version is)

```sh
cd /Applications/Python\ 3.10/
./Install\ Certificates.command
```


- Clone the repo
```
git clone https://github.com/nerdalert/aws-ansible-kubernetes.git
cd aws-ansible-kubernetes 
```


- Setup Ansible vault - from within the cloned repo directory run the following and paste in your
  AWS credentials. Instructions on retrieving your AWS credentials at [AWS Getting Your Credentials](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/getting-your-credentials.html)

```sh
# The following will open an editor
ansible-vault create credentials.yml

# Paste the following in the opened editor and save the file
access_key: <add_access_key_here>
secret_key: <add_secret_key_here>
```

- Adjust the `ansible.cfg` file in the base directory to reflect your environment.
- The main one that needs to be edited for your environment will be the path to the
  `private_key_file.pem` entry that is associated with your AWS key specified in `vars.yml`.
  You can also simply copy the key to the base directory and leave off the path.
- In some environments you won't need to enable `become_ask_pass` but adding it to be as
  agnostic as possible to all installations.

```yaml
# ansible.cfg
[defaults]
# this is an default inventory location, user can change it accordingly
host_key_checking = false
deprecation_warnings = false
ask_pass = false
stdout_callback = yaml
remote_user = fedora
# defaults to the base directory in the project
inventory = ip.txt
# create .pem private_key_file and provide location
private_key_file = <aws_private_key_name>.pem

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = true
```

- Next set the environmentals for your AWS EC2 details in `env.yaml` located in the base
  of the project. Here are some example values.

```yaml
# env.yaml
aws_region: us-east-1                 # AWS region
vpc_id: vpc-xxxxxxxx                  # VPC id from your aws account
aws_subnet: subnet-xxxxxxxx           # VPC subnet id from your aws account
aws_image_id: ami-08b4ee602f76bff79   # Fedora 35 (this can be changed to most any Linux distro, be sure to change ansible_user name if you use a different distro)
aws_key_name: <key_pair_name>         # the key pair on your aws account to use
aws_instance_type: t2.micro           # t2.micro is free tier eligable, but you can use any type to scale up
ansible_user: fedora                  # this is the default user ID for your AMI image. Example, AWS AMI is ec2-user etc
worker_node_count: 6                  # the number of worker nodes you want to deploy
secgroup_name: <aws-security-group>   # the security group name can be an existing group or else it will be created by the playbook
inventory_location: ip.txt            # leaving this as is will use the ip.txt file in the base directory
security_group_description: "Security Group for Perf/Scale testing allowing ssh ingress"
```

### Run the installation

- Once your `env.yaml` is setup for your EC2 environment, you are ready to run the playbook to deploy the EC2 instances.
  This will create the VMs for the deployment (add -vv for verbose output).


```sh
$ ansible-playbook --ask-vault-pass setup-ec2.yml
# host being run from su password (may not be required in some setups, can disable in ansible.cfg)
BECOME password:
# password when you created the ansible vault
Vault password:
```

- After that run is complete, you can always ping the nodes to verify connectivity by running the following from the base directory:

```sh
ansible all -m ping
```

```sh
ansible-playbook --ask-vault-pass setup-ec2.yml
```

- Example inventory file after running `setup-ec2.yml` with 6 worker nodes specified stored in `ip.txt` in base directory:

```yaml
[masterNode]
3.84.200.218 ansible_user=fedora ansible_connection=ssh

[workerNode]
54.226.69.231 ansible_user=fedora ansible_connection=ssh
54.226.101.8 ansible_user=fedora ansible_connection=ssh
34.235.143.35 ansible_user=fedora ansible_connection=ssh
3.94.190.207 ansible_user=fedora ansible_connection=ssh
3.88.3.101 ansible_user=fedora ansible_connection=ssh
18.212.246.28 ansible_user=fedora ansible_connection=ssh
```

- You can double check connectivity to the new nodes with:

```sh
# This pings all of the hosts in your ip.txt file 
ansible all -m ping
```

- Once your nodes are running, deploy K3s Kubernetes to the nodes listed in your inventory file `ip.txt` by running the playbooks in `setup-kubernetes.yml`

```
ansible-playbook setup-k8s.yml
```

Assuming that runs with no issues, your k8s deployment is up and running.

### Verify the Kubernetes Deployment

- Connect and verify the installation by grabbing an address out of your Ansible inventory in `ip.txt`:

```sh
# ssh to the master node. you can look in ip.txt for the ip address
ssh -i ./<aws-private-key>.pem  fedora@<master_node_ip>

# export the kube config
export KUBECONFIG=~/.kube/config

# view the nodes in the cluster
[fedora@ip-172-31-23-39 ~]$ kubectl get nodes -o wide
NAME                            STATUS   ROLES                  AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                          KERNEL-VERSION            CONTAINER-RUNTIME
ip-172-31-18-204.ec2.internal   Ready    <none>                 5m9s    v1.22.7+k3s1   172.31.18.204   <none>        Fedora Linux 35 (Cloud Edition)   5.14.10-300.fc35.x86_64   containerd://1.5.9-k3s1
ip-172-31-23-39.ec2.internal    Ready    control-plane,master   8m25s   v1.22.7+k3s1   172.31.23.39    <none>        Fedora Linux 35 (Cloud Edition)   5.14.10-300.fc35.x86_64   containerd://1.5.9-k3s1
ip-172-31-17-164.ec2.internal   Ready    <none>                 6m42s   v1.22.7+k3s1   172.31.17.164   <none>        Fedora Linux 35 (Cloud Edition)   5.14.10-300.fc35.x86_64   containerd://1.5.9-k3s1
ip-172-31-16-79.ec2.internal    Ready    <none>                 6m2s    v1.22.7+k3s1   172.31.16.79    <none>        Fedora Linux 35 (Cloud Edition)   5.14.10-300.fc35.x86_64   containerd://1.5.9-k3s1
ip-172-31-25-147.ec2.internal   Ready    <none>                 6m1s    v1.22.7+k3s1   172.31.25.147   <none>        Fedora Linux 35 (Cloud Edition)   5.14.10-300.fc35.x86_64   containerd://1.5.9-k3s1
ip-172-31-24-26.ec2.internal    Ready    <none>                 5m58s   v1.22.7+k3s1   172.31.24.26    <none>        Fedora Linux 35 (Cloud Edition)   5.14.10-300.fc35.x86_64   containerd://1.5.9-k3s1
ip-172-31-19-38.ec2.internal    Ready    <none>                 5m56s   v1.22.7+k3s1   172.31.19.38    <none>        Fedora Linux 35 (Cloud Edition)   5.14.10-300.fc35.x86_64   containerd://1.5.9-k3s1

# view running pods
[fedora@ip-172-31-23-39 ~]$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE     IP          NODE                            NOMINATED NODE   READINESS GATES
kube-system   local-path-provisioner-84bb864455-tvw6n   1/1     Running     0          8m54s   10.42.0.5   ip-172-31-23-39.ec2.internal    <none>           <none>
kube-system   coredns-96cc4f57d-rxwcl                   1/1     Running     0          8m54s   10.42.0.4   ip-172-31-23-39.ec2.internal    <none>           <none>
kube-system   helm-install-traefik-crd--1-d26rg         0/1     Completed   0          8m55s   10.42.0.2   ip-172-31-23-39.ec2.internal    <none>           <none>
kube-system   metrics-server-ff9dbcb6c-hwg82            1/1     Running     0          8m54s   10.42.0.6   ip-172-31-23-39.ec2.internal    <none>           <none>
kube-system   helm-install-traefik--1-bwh4t             0/1     Completed   1          8m55s   10.42.0.3   ip-172-31-23-39.ec2.internal    <none>           <none>
kube-system   svclb-traefik-pzfpg                       2/2     Running     0          8m11s   10.42.0.7   ip-172-31-23-39.ec2.internal    <none>           <none>
kube-system   traefik-56c4b88c4b-zh557                  1/1     Running     0          8m13s   10.42.0.8   ip-172-31-23-39.ec2.internal    <none>           <none>
kube-system   svclb-traefik-5jqjh                       2/2     Running     0          7m24s   10.42.1.2   ip-172-31-17-164.ec2.internal   <none>           <none>
kube-system   svclb-traefik-mzb57                       2/2     Running     0          6m44s   10.42.2.2   ip-172-31-16-79.ec2.internal    <none>           <none>
kube-system   svclb-traefik-bjw4r                       2/2     Running     0          6m43s   10.42.3.2   ip-172-31-25-147.ec2.internal   <none>           <none>
kube-system   svclb-traefik-qclmh                       2/2     Running     0          6m40s   10.42.4.2   ip-172-31-24-26.ec2.internal    <none>           <none>
kube-system   svclb-traefik-64vqd                       2/2     Running     0          6m38s   10.42.5.2   ip-172-31-19-38.ec2.internal    <none>           <none>
kube-system   svclb-traefik-txtl8                       2/2     Running     0          5m51s   10.42.6.2   ip-172-31-18-204.ec2.internal   <none>           <none>
```

You are all set from there, feel free to leave feedback, open issues and/or PRs!
