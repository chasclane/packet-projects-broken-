# Charlie's Packet Adventure

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# Getting Started
Here's what you need in order to go on this adventure:

1. The ability to SSH, either natively or via an SSH Client such as `PuTTY` (This will be useful for troubleshooting and validation, though not mandatory for this exercise.)  If you want to use `PuTTy`, you can download it here: http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html 

2. A text editor - Preferably one that parses code easily for readability. Though also not required to execute anything, is helpful to peruse the code to gain a better understanding of how it works. There are many out there, choose the best one... :) 

3. A Web Browser - (Chrome, Firefox, Iceweasel, Safari, IE, Edge, etc.) 

4. An account on Packet.com - Sign up for an account, add a credit card, create an Organization.  

5. A GitHub account - Critical pieces of this project leverages the power of GitHub Actions. Can't run GitHub Actions without GitHub. (Right??)

***Note: Packet.com is not a free service. Yes... you must add a credit card. None of these workflows, Actions, or playbooks have access to your billing information. 

# Before We Begin 

There are some preliminary steps that we need to take before we can execute our Actions to begin our deployment workflows. 

## Pre Step 1. Create a GitHub account and a New Repository.

Create an account on GitHub.com if you don't have one already. Create a new repository named `packet-projects`. 

`Copy` or `Clone` Everything EXCEPT the "id_packet.gpg" from this repository contents into your new repository. You will be creating your own "id_packet.gpg" in later steps. Note: If you keep the same file, the steps that rely on that will ultimately fail. 

## Pre Step 2: Setup the Packet.com API secret:`PACKET_API_KEY` 

Create an account on Packet.com and follow the required steps to enable the free trial. 

Log in to the Packet.com portal, click on your user profile on the top right, select `API Keys`. 
Generate a new Read/Write API key, labeling it with an appropriate description set for explicit use. `Copy` the token shown there. 

Navigate to your repo's `Secrets` which can be found under the `settings` tab, create a new secret named `PACKET_API_KEY`, and `paste` the token you copied from the previous step, and save the secret.


## Pre Step 3: Store your new public key for use in the workflows as the new secret:`PACKET_PUBLIC_KEY`

If you have not generated a key pair yet and need steps to do so, follow the steps according to the OS you're running listed in the guide here: https://www.packet.com/developers/docs/servers/key-features/ssh-keys/. 

Once you have your key pair, `copy` the contents of your public key key file. 

Note: It should start with something like this:
  ```
  ssh-rsa AAAA............== rsa-key-xxxxxxx
  ```
Next, navigate to your repo's `Secrets` which can be found under the `settings` tab. Then, create a new secret named `PACKET_PUBLIC_KEY`, and `paste` the value you copied from the previous step, and save the secret.


## Pre Step 4: Setup Private key to be stored in the new secret:`PACKET_PRIVATE_KEY`

In order to use Ansible to run playbooks on the target systems, we will first need to store our private key so that the Ansible host can use it to authenticate to the target machines. GitHub Secrets have a small character storage size, so we have to encrypt the Private key file with `gpg` and then commit it in the repo.

Execute the following command to encrypt your private key (in `.pem` format) and specify an appropriately strong **gpg passphrase** when prompted to enter one. Copy this passphrase for the next step. 

```shell
  gpg --symmetric --cipher-algo AES256 /path/to/my_private_key.pem
```
This will output `my_private_key.gpg`. 

Rename this key to: `id_packet.gpg`.

Place the `id_packet.gpg` file in the `root` directory of the repository. This should be at the same level as the 'ansible' directory.

We will then set the **gpg passphrase** for decryption as a repository secret, exactly the same way we did for our API key ( `PACKET_API_KEY` ) and our public key (`PACKET_PUBLIC_KEY`)

  To do this, we again navigate to your repo's `Secrets` which can be found under the `settings` tab. Then, create a new secret named `PACKET_PRIVATE_KEY`, and `paste` the **gpg passphrase** you copied from the previous step, and save the secret.

  ### This object in use - Decrypting the secret:

  This secret is used in an Action that is a part of the `run-ansible.yml` file. It looks like this:

  ```yaml
  - name: Decrypt Pem
       run: |
         mkdir $HOME/secrets
         gpg --quiet --batch --yes --decrypt --passphrase="$SECRET_PASSPHRASE" --output $HOME/secrets/my_private_key.pem my_private_key.gpg
         chmod 600 $HOME/secrets/packet_private_key.pem
       env:
         SECRET_PASSPHRASE: ${{ secrets.PACKET_PRIVATE_KEY }}
  ```

   NOTE: This will decrypt your key inside of the runner and will be destroyed when the job is complete. (Additional steps should be taken for production environments to ensure the confidentiality of your keys is maintained.)


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

# So, what's our Plan? 

We're going to be running 3 Workflows:

1. Provision our Environment
2. Create a Kubernetes cluster with Ansible
3. Deploy Kubernetes Nodes w/ workloads

Looks something like this:

![Packet-Project](https://github.com/chasclane/packet-projects/blob/master/img/Charlie's%20Packet%20Adventure%20Diagram.png)


Okay, we're ready to run our first workflow!

## 1. Provision our Environment

- Actions In Use For This Workflow: 

https://github.com/mattdavis0351/packet-create-user-ssh-key 
https://github.com/mattdavis0351/packet-create-project 
https://github.com/mattdavis0351/packet-create-device-batch 

- How we're executing this one: 

We're executing this workflow when the event on `watch` occurs. This means our workflow will trigger and run when the workflow file `provision-env.yml` is "Stared". 

To run this workflow select the `Star` (or Unstar then Star) at the top-right corner of the project repo.

- What's happening and how:

The first thing we're going to in this workflow, is to ensure that we can remotely manage our targets. We do this by performing an "Action" which builds and uses a runner (the runners we'll be using today happen to be Ubuntu runners, hosted `free of charge` in Azure). This runner will be requesting a new SSH key and storing it for us for future use. We're using Ubuntu because we know that Ansible will successfully run on this, and this node will be able to use kubctl cli to manage the Kubernetes cluster later on. 

The runner does this by calling the Packet.com API to create a user SSH key and passing that to our workflow with the ID 'key'.

Next, the runner calls another Packet.com API to build our Project. 

Here, we give our project a Name, mine is called: **Charlie's Packet Adventure**

Once our Project is created, we send a runner to build a device batch group which will contain 4 VMs, each running CentOS operating system. 

Our new devices will be: One Kubernetes Master node named: `K8s-Master` and three Minion nodes named: `K8s-minion-1`, `K8s-minion-2`, and `K8s-minion-3`

**Let's Pause here to take care of a couple things** (THIS STEP WILL BE AUTOMATED IN THE NEAR FUTURE)

Before we move on, there are a couple things we have to do once the previous workflow is complete, and our 4 nodes are successfully deployed. NOTE: While the workflow will show as completed, the nodes may still be in the process of being provisioned. This can take some time. 

-Determine deployment status and grab the K8S master IP address: 

  View the status of the previous workflow by logging in to the Packet.com portal, opening your new project, and go to your newly deployed instances. From here, take note of the assigned IPv4 address for our `K8s-Master`machine. We'll need to then add this IP address in a couple of places for the next steps to run correctly.  (THIS STEP WILL BE AUTOMATED IN THE NEAR FUTURE)

  Copy the IPv4 address assigned to the `K8s-Master` node and paste it as the variable `ad_addr` in the environment variables file `env_variables` located at the root of the repository. The end result of that should look like: 

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
-Before we move on to the next workflow, let's paste the same IP address in the final Workflow .yaml file: `deploy-k8s.yml` in the line with: `run: scp -i` replacing the existing IP address with the correct/current IP address for the master node. This task is ensuring that for any minion nodes to be managed by the master, that the K8s-Master node will be able to successfully SSh into each, if needed.  

This will ensure that the fun doesn't have to stop at the end of the 3rd workflow in this exercise. Much more can be easily executed in our new k8s environment!

Now it's time to move on to Workflow 2!

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

## 2. Create a Kubernetes cluster with Ansible

- Actions In Use For This Workflow:  

https://github.com/mattdavis0351/packet-create-ansible-inventory
https://github.com/marketplace/actions/upload-artifact
https://github.com/actions/download-artifact
https://github.com/marketplace/actions/checkout 

- How we're executing this one: 

In order to execute this step, we have configured the workflow to trigger on the "labeled" event. This means that it will run when we add a Label to an issue logged on the `Issues` tab of the repository.

Either create a new `Label`, or add an existing: I've added the label named "Much Awesome", and applied that label to an Issue I named `Are we having fun yet?`, which runs the following workflow:

https://github.com/chasclane/packet-projects/blob/master/.github/workflows/run-ansible.yml

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

Once complete, we can move on to Workflow 3!

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

## 3. Deploy Kubernetes pods w/ workloads

The last workload we'll be executing is to deploy Kubernetes pods with our desired workloads

- Actions In Use For This Workflow: 

https://github.com/marketplace/actions/checkout

- How we're executing this one:

This workflow is executed on the event that a comment is added or edited on the workflow file: deploy-k8s.yml. 

Add or edit a comment to an Issue to run the following workflow:

https://github.com/chasclane/packet-projects/blob/master/.github/workflows/deploy-k8s.yml

The first job this workflow is executing is to deploy Kubernetes CLI kubectl to the runner so that it can execute commands for the kubernetes master remotely. 

To do this, we securely copy the ssh keys to the runner so it can execute commands using ssh. 

The runner then tests this by executing the command: 

```shell
kubectl get nodes
```

# So... what next? 

***Well, that's up to you! Now that we've put the CI/CD framework in place, you're ready to either scale your k8s cluster up or down, deploy container workloads and explore the power of deploying to a true bare-metal as a service platform using an awesome set of tools to build and use our awesome new CI/CD pipeline!***

