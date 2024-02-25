# Remote server setup:
Make sure that ipv6 is supported.

also, once up generate an SSH key Locally, and copy that public key int available_hosts in ~/.ssh  on the new machine

For Azure, add 80, and 443 inbound rules to a Network Security Group on the NIC.

update domain dns records with external IP address.



In sections under site and trellis there may be more info on how to install bedrock/trellis and vagrant

THe above can be useful in installng Trellis and Vagrant. 
Here are also some more rough notes to assist:
Install trellis-cli 
https://github.com/roots/trellis-cli
  
this should work on Mac.
  brew install roots/tap/trellis-cli

  Vagrant can be tricky on Mac M silicon, but it seems the latest versions work.
  The brew approach did not work, so I followed the "manual" approach:
  - Download the vagrant pacakge : get the latest from vagrant site.
  .
  - Fnally check , with $ sudo vagrant --version
    Also for VMWare, this was needed:

Now install vagrant Vmware Utility (check for latest bht this shold workl : https://releases.hashicorp.com/vagrant-vmware-utility/1.0.22/vagrant-vmware-utility_1.0.22_darwin_amd64.dmg

  - Now install vagrant manager on M1 machine : brew install vagrant-manager --cask 




This error : 
Trellis-cli was not on path. Fix:
echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc


## Creating new Wordpress Site Locally
Navigate to Repos/mynewsitedir

trellis new sitename

where sitename is the name of the public domain name you have in mind.


## Review config in Group_vars
Mostly wordpress_sites.yml in group_vars folder stays unchanged, for dev/staging /production.
but remember to eventually configure:
repo: git@github.com:user/repo.git in production.
also make sure branch: is master or main depending on what was done.


## Create Virtual Development server using Vagrant
Edit trellis/vagrant.default.yml
It is KEY to use a vagrant box that is arm_64 compatible for APple M Siicon. also for Vmware Fusion ,t seems 'dhcp' is the only vagrant_ip that works.

For example, in vagrant.default.yml, these setings work for VMware fusion

vagrant_ip: 'dhcp'
vagrant_cpus: 1
vagrant_memory: 1024 # in MB
vagrant_box: 'bento/ubuntu-22.04-arm64'

Now, run vagrant up --provider=vmware-desktop


## Encrypt vault.yml using ansible
IN Trellis directory, run:
ansible-playbook server.yml -e env=production --vault-password-file .vault_pass

the --vault-password-file should be optional.

If you need to view files decrypted:
ansible-vault view group_vars/<environment>/vault.yml
 Theres also a way to persist this to disk drive. you can google that.

## commit to Git.
make sure trellis/.vault_pass  is in .gitignore
also make sure site/.env is in .gitignore


## Prepare to deploy
## set up ssh keys.
Generate Key Pair: Open a terminal on your local machine and run:

cd ~/.ssh
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
Follow the prompts to choose a save location (e.g., ~/.ssh/azure_rsa) and optionally set a passphrase for the key.

Public Key for Azure Server: The public key (azure_rsa.pub) needs to be added to your Azure server to allow SSH access. Do not move the private key to the remote server. The private key should remain secure on your local machine.

you may then have to add the private key : ssh-add ~/.ssh/your_private_key  of the remote azure server,
and the run
ssh-copy-id -i ~/.ssh/azure_rsa.pub azureuser@xxx.xxx.xxx.xxx
to copy the public key of the local machine to the remote ubuntu.


Edit and add the following to sudo ~/.ssh/config
The Host is a name of your choosing, e.g. ubuntu_azure
HostName is the IP address
user is the user used to log into remote server, eg. azureuser
The ForwardAgent yes is used to allow git repo access.

For this ForwardAgent to work
Check Server Configuration: Your server's SSH daemon must allow agent forwarding. This is typically enabled by default, but you can verify in /etc/ssh/sshd_config on the server:
AllowAgentForwarding yes

If you make changes to sshd_config, remember to restart the SSH service (sudo systemctl restart ssh).

on mac: 
Open System Preferences.
Go to Sharing.
Uncheck Remote Login to turn off SSH access.
Wait a few seconds, then re-check Remote Login to turn SSH access back on.


Host ubuntu-azure
    HostName xxx.xxx.xxx.xxx
    User azureuser
    IdentityFile ~/.ssh/azure_rsa
    ForwardAgent yes



Now in trellis/hosts/
for both production and optionlaly staging, put this in both (lets say we went with Host ubuntu_azure above)
.

[production]
ubuntu_azurevm ansible_user=azureuser

[web]
ubuntu_azurevm

## provision server
Configure Git in wordpress_sites:


now we can prepare site locally, 
in trellis folder run

ansible-playbook server.yml -e env=production





## deploy , or update Production!
After each Provision, Prior to deployment ,there seems to be a bug:
ssh into remote server using azureuser.
then sudo chown -R azureuser:www-data /srv/www


Now, in trellis/ folder, run:
trellis deploy production



where yoursite.com is the site specified on group_vars/ for the environment
befreenow.space


DNS Configuration: Make sure your domain's DNS records are updated to point to your Azure VM's public IP address.
Firewall and Security Groups: Verify that your Azure VM's network security group allows HTTP (port 80) and HTTPS (port 443) traffic.
Database: Trellis will set up a MariaDB database as part of the provisioning process. Ensure your database details in vault.yml match what you intend to use for your site.


## access remote server via ssh
ssh ubuntu-azurevm

## remote server 80,443
Remember to add network security rules for 80,443 traffic.
Also you might have to add a firewall rule to ubuntu remote:

try netcast to confirm 80,443
nc -zv xxx.xxx.xxx.xxx 443

rememober also to set ssl=true in wordpress_sites.yml 

and to use letsencrypt, add this to main.yml
letsencrypt_contact_emails:
  - username@myemailbox.com
