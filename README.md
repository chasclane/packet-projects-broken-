# Charlie's Packet Adventure

##(An exciting adventure taking some of our favorite tools with us on a journey to deploying on Packet.com. Some of these involved in this project include: Ansible, Kubernetes, GitHub, and GitHub Actions, helping us arrive at our destination: Packet Bare Metal!! )##


## Before We Begin 

There are some preliminary steps that we need to take before we can execute our Actions to begin our deployment workflows. 

## Setup Private key to be stored in repo.

In order to use Ansible, first we will need to store our private key so that the Ansible host can use it to authenticate to the target machines. GitHub Secrets have a small character storage size, so we have to encrypt the Private key file with `gpg` and then commit it in the repo.

We will set the passphrase for decryption as a repository secret, exactly the same way we did for our API key ( `PACKET_API_KEY` ) and our  public key (PACKET_PUBLIC_KEY)

**To Use GPG and store for later use:**
Run:
```shell
  `gpg --symmetric --cipher-algo AES256 /path/to/my_private_key.pem`
```
This will output `my_private_key.gpg`

Place that in the `root` directory of the repository, alongside the 'ansible' directory.

**Decrypting this file:**

Next we create a secret called `PACKET_PRIVATE_KEY` and assign the **gpg passphrase** as the value to this secret.

We use that secret in an action that is a part of the `run-ansible.yml` file. It looks like this:

```yaml
- name: Decrypt Pem
     run: |
       mkdir $HOME/secrets
       gpg --quiet --batch --yes --decrypt --passphrase="$SECRET_PASSPHRASE" --output $HOME/secrets/my_private_key.pem my_private_key.gpg
       chmod 600 $HOME/secrets/packet_private_key.pem
     env:
       SECRET_PASSPHRASE: ${{ secrets.PACKET_PRIVATE_KEY }}
```

This will decrypt your key inside of the runner and it will be destroyed when the job is complete

## So, what's the Plan? 

We're going to be running 3 Workflows:

1. Provision our Environment
2. Create a Kubernetes cluster with Ansible
3. Deploy Kubernetes Nodes w/ workloads

**1. Provision our Environment**

- Actions In Use For This Workflow: 

https://github.com/mattdavis0351/packet-create-user-ssh-key
https://github.com/mattdavis0351/packet-create-project 
https://github.com/mattdavis0351/packet-create-device-batch 

- How we're executing this one: 

We're executing this workflow when the event on `watch` occurs. This means our workflow will trigger and run when the workflow file `provision-env.yml` is "watched".

- What's happening and how:

The first thing we're going to in this workflow, is to ensure that we can remotely manage our targets. We do this by performing an "Action" which builds and uses a runner (the runners we'll be using today happen to be Ubuntu runners, hosted `free of charge` in Azure). This runner will be requesting a new SSH key and storing it for us for future use. We're using Ubuntu because we know that Ansible will successfully run on this, and this node will be able to use kubctl cli to manage the Kubernetes cluster later on. 

The runner does this by calling the Packet.com API to create a user SSH key and passing that to our workflow with the ID 'key'.

Next, the runner calls another Packet.com API to build our Project. 

Here, we give our project a Name, mine is called: **Charlie's Packet Adventure**

Once our Project is created, we send a runner to build a device batch group which will contain 4 VMs, each running CentOS operating system. 

Our new devices will be: One Kubernetes Master node named: `K8s-Master` and three Minion nodes named: `K8s-minion-1`, `K8s-minion-2`, and `K8s-minion-3`

**Let's Pause here for a minute**

Before we move on, there are a couple things we have to do once the previous workflow is complete, and our 4 nodes are successfully deployed. NOTE: While the workflow will show as completed, the nodes may still be in the process of being provisioned. This can take some time. 

Once these have been successfully deployed, we need to access the Packet Portal, navigate to our project, find our newely deployed instances, and document the assigned IPv4 address for our `K8s-Master`. We'll need to then add this IP address in a couple of places for the next steps to run correctly. 

Copy the address assigned to the `K8s-Master` node and paste it as the variable `ad_addr` in the environment variables file `env_variables` located at the root of the repository. The end result of that should look like:

```yaml
#Enter the IP Address of the Kubernetes Master node in for the ad_addr variable.
ad_addr: *Kube Master IP Here* 
cidr_v: 10.244.0.0/16 #NOTE: Do Not Change This - flannel is looking for this block by default and we haven't configured it otherwise.

packages:
- kubeadm
- kubectl

services:
- docker
- kubelet
- firewalld

ports:
- "6443/tcp"
- "10250/tcp"

token_file: join_token
```
Next thing before we move on to the next workflow, is we'll paste the same IP address in the final Workflow .yaml file: `deploy-k8s.yml` in the line with: `run: scp -i` replacing the existing IP address with the correct/current IP address for the master node. This task is ensuring that for any minion nodes to be managed by the master, that the K8s-Master node will be able to successfully SSh into each, if needed.  

Now we can move on to Workflow 2!

**2. Create a Kubernetes cluster with Ansible**

- Actions In Use For This Workflow:  

https://github.com/mattdavis0351/packet-create-ansible-inventory
https://github.com/marketplace/actions/upload-artifact
https://github.com/actions/download-artifact
https://github.com/marketplace/actions/checkout 

- How we're executing this one: 

In order to execute this step, we have configured it to trigger on the "labeled" event. This means it will run when we add a Label to an issue for the workflow file: `run-ansible.yml`

- What's happening and how:

The first Action performed here is to send a runner to gather an Inventory of our nodes, which also helps us ensure that they are all running and accessible. 

This is accomplished by our runner using another Packet.com API querying for device IDs and hostnames, then adding them to a comma separated inventory file. Items in this file are then grouped based on the `group_names`: `master` and `minion`. 

The runner then does the job of using an Ansible playbook to create a `hosts` inventory. This inventory file is then uploaded to our repository as an artifact that can be downloaded and used for future tasks. We'll be using this pretty much immediately and will download it in the very next job. 

This Ansible "inventory" `hosts` file is then placed in the directory alongside our `ansible.cfg` and `env_variables` files. 

The final 2 jobs in this workflow are using Ansible Playbooks to set up the K8s-Master node and minion nodes. 

The Ansible playbook: `configure_master_nodes.yml` performs the following tasks on the k8s-master node:

- Pull images required for setting up a kubernetes cluster
- Reset kubeadm
- Initializing kubernetes cluster
- Store logs and the generated token (to be used in future workflow)
- Copy required files to appropraiate locations for future use.
- Install Network Add-on to ensure compatibility with container drivers

The Ansible playbook: `configure_worker_nodes.yml` performs the following tasks on the minion nodes found in the `hosts` file:

- Copies the token to the minion nodes to the destination: /tmp/join_token
- Join the minion nodes to the kubernetes cluster 

Note: This may take some time to complete

**3. Deploy Kubernetes pods w/ workloads**

The last workload we'll be executing is to deploy Kubernetes pods with our desired workloads

- Actions In Use For This Workflow: 

https://github.com/marketplace/actions/checkout

- How we're executing this one:

This workflow is executed on the event that a comment is added or edited on the workflow file: deploy-k8s.yml. 

The first job this workflow is executing is to deploy Kubernetes CLI kubectl to the runner so that it can execute commands for the kubernetes master remotely. 

To do this, we securely copy the ssh keys to the runner so it can execute commands using ssh. 

The runner then tests this by executing the command: ```kubectl get nodes```
