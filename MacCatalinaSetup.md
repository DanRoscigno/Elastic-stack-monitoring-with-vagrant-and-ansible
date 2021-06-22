# Prerequisites:
 - Python 3
 - Vagrant
 - Virtualbox
 - Ansible
 - Sufficient CPU & RAM to build 5 vms (adjust the resource allocation in 'inventory.yml')

# Commands for macOS Catalina setup
```
/bin/bash -c \
   "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install pyenv

pyenv install 3.9.1

echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.zshrc

source ~/.zshrc

python --version

python -m pip install --upgrade pip

pip install pyyaml

pip install wheel

pip install --upgrade pip setuptools wheel

pip install pip-tools

brew install rust

brew install vagrant

brew install virtualbox
#( allow the system extension and reboot)

vagrant plugin install vagrant-vbguest

brew install ansible

ansible-galaxy install elastic.elasticsearch
ansible-galaxy install elastic.beats
```

# Configure your workstation for `vagrant ssh`

There seems to be some kind of an issue with many keys being added and using ssh, so tell ssh that the vagrant user should not be allowed to use passwords
Add this to ~/.ssh/config
```
User vagrant
  IdentitiesOnly yes
```
