In Kibana hamburger > Management > Fleet

Top right corner, Gear symbol > Fleet Settings

Add Fleet servers, use the IP addrs from the inventory.yml and port 8220.  Type the URL in and hit enter for each Fleet host:
https://192.168.33.30:8220 https://192.168.33.31:8220 

Set the Elasticsearch servers, again using the IP addrs from the inventory.yml
https://192.168.33.25:9200 https://192.168.33.26:9200 https://192.168.33.27:9200

Note: If the master node is a dedicated master then maybe only the data nodes?  Check into this.

now click the Agents tab (NOT add agent), and generate the token.  Store the token in a safe place, maybe in the ansible-vault.  If you are using a .deb or a .rpm to deploy Agent then you also need an Enrollment token.  These are available on the same page as the service token under the Enrollment Tokens tab.  Switch there and view / copy the secret for the Default Fleet Server policy.

Install an agent on each of the Fleet server hosts. In the instructions you are told to download the Agent binary, use vagrant ssh and curl the file:

vagrant ssh fleet-1

curl -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-7.13.1-amd64.deb

sudo dpkg -i ./elastic-agent-7.13.1-amd64.deb
sudo apt-get install -f


To enroll Agent with the Fleet server policy, and have proper certs this is the command (note the certs were created with the hostname and IP address of the fleet server in the instructions for using Ansible to deploy the stack)

Here is a list of the cert files being used:
Certificate Authority (used to sign both the Elasticsearch cert and the Fleet Server cert):
    /vagrant/files/certs/ca/ca.crt
Fleet server certificate:
    /vagrant/files/certs/fleet-1/fleet-1.crt
Fleet server private key:
    /vagrant/files/certs/fleet-1/fleet-1.key

sudo /usr/bin/elastic-agent enroll \
   --url=https://192.168.33.30:8220 \
   --fleet-server-es=https://192.168.33.25:9200 \
   --fleet-server-es-ca=/vagrant/files/certs/ca/ca.crt \
   --fleet-server-service-token=<< your service token here >> \
   --enrollment-token=<< your enrollment token here >> \
   --ca-sha256=/vagrant/files/certs/ca/ca.crt \
   --fleet-server-cert=/vagrant/files/certs/fleet-1/fleet-1.crt \
   --fleet-server-cert-key=/vagrant/files/certs/fleet-1/fleet-1.key

After enrolling the Elastic Agent process stops running.  To start the Agent, and the included Fleet server run:

sudo /usr/bin/elastic-agent run

or maybe:

sudo systemctl start elastic-agent
sudo systemctl status elastic-agent
