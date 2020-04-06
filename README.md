# packet-test

## Setup Private key to be stored in repo.

If you want to use Ansible you will need to store your private key so that the Ansible host can use it to authenticate to the target machines. GitHub Secrets have a small character storage size, so we encrypt the Private key file with `gpg` and then commit it in the repo.

You will set the passphrase for decryption as a repository secret, just like `PACKET_API_KEY` and your public key

**To Use GPG:**

`gpg --symmetric --cipher-algo AES256 /path/to/my_private_key.pem`

This will output `my_private_key.gpg`

Place that in the `root` directory of the repo, alongside the `ansible.cfg` and `your-playbook.yml`

**Decrypting this file:**

Next create a secret called `PACKET_PRIVATE_KEY` and assign the **gpg passphrase** as the value to this secret.

Use that secret in an action that is a part of the `run-ansible.yml` file. It looks like this:

```yaml
- name: Decrypt Pem
     run: |
       mkdir $HOME/secrets
       gpg --quiet --batch --yes --decrypt --passphrase="$SECRET_PASSPHRASE" --output $HOME/secrets/my_private_key.pem my_private_key.gpg
       chmod 600 $HOME/secrets/packet_private_key.pem
     env:
       SECRET_PASSPHRASE: ${{ secrets.PACKET_PRIVATE_KEY }}
```

This will decrypt your key inside of the runner and it will be destroyed when the job is complete.

## Ansible Config

There is an `ansible.cfg` file in the root of this directory. It has a few options enabled, you need to have the same enabled if you want to avoid the authentication problems I was having. It also set's up whatever is needed to point to the proper inventory files for your playbooks as well. Feel free to modify whatever setting you'd like in this file, it will be the cfg file that is used by default inside of the runner environment.

## Directory structure

If you choose to put your playbooks in a `playbooks` folder or something similar, make sure you reference that folder in the action that calls `ansible-playbook`
