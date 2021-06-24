# Specify Fleet server settings
In Kibana hamburger > Management > Fleet

Top right corner, Gear symbol > Fleet Settings

Add Fleet servers, use the IP addrs from the inventory.yml and port 8220.  Type the URL in and hit enter for each Fleet host:
```
https://192.168.33.30:8220
https://192.168.33.31:8220
```

Set the Elasticsearch servers, again using the IP addrs from the inventory.yml
```
https://192.168.33.25:9200
https://192.168.33.26:9200
https://192.168.33.27:9200
```

In the panel `Elasticsearch output configuration (YAML)` specify the TLS certificate authority with the full path to the ca.crt:
```
ssl.certificate_authorities: ["/vagrant/files/certs/ca/ca.crt"]
```
![Fleet Server settings flyout](https://raw.githubusercontent.com/DanRoscigno/Elastic-stack-monitoring-with-vagrant-and-ansible/main/images/FleetSettings.png)

# Generate the service token 
The service token is used to specify the Fleet server process running in the Elastic Agent.

Click the Agents tab (NOT add agent), and generate the token (I don't remember exactly what the UI shows, and I no longer see the content as I have generated the service token).  Store the token in a safe place, maybe in the ansible-vault.  

# Install Elastic Agent on the box(es) that will be Fleet servers

Fleet server is a process running in the Agent.

Install an agent on each of the Fleet server hosts. In the instructions you are told to download the Agent binary, use vagrant ssh and curl the file:

```
vagrant ssh fleet-1
```

```
curl -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-7.13.1-amd64.deb

sudo dpkg -i ./elastic-agent-7.13.1-amd64.deb
sudo apt-get install -f
```

# Enroll the Agent
To enroll Agent with the Fleet server policy, and have proper certs this is the command (note the certs were created with the hostname and IP address of the fleet server in the instructions for using Ansible to deploy the stack)

Here is a list of the cert files being used:
Certificate Authority (used to sign both the Elasticsearch cert and the Fleet Server cert):
    /vagrant/files/certs/ca/ca.crt
Fleet server certificate:
    /vagrant/files/certs/fleet-1/fleet-1.crt
Fleet server private key:
    /vagrant/files/certs/fleet-1/fleet-1.key

```
sudo /usr/bin/elastic-agent enroll \
   --url=https://192.168.33.30:8220 \
   --fleet-server-es=https://192.168.33.25:9200 \
   --fleet-server-es-ca=/vagrant/files/certs/ca/ca.crt \
   --fleet-server-service-token=<< your service token here >> \
   --ca-sha256=/vagrant/files/certs/ca/ca.crt \
   --fleet-server-cert=/vagrant/files/certs/fleet-1/fleet-1.crt \
   --fleet-server-cert-key=/vagrant/files/certs/fleet-1/fleet-1.key
```

# Start the Agent
After enrolling the Elastic Agent process stops running.  To start the Agent, and the included Fleet server run:

```
sudo /usr/bin/elastic-agent run
```

or maybe:

```
sudo systemctl start elastic-agent
sudo systemctl status elastic-agent
```

# Enroll a second Agent with a new Policy
To add an Agent on a second server that is managed by the Fleet server running in Elastic Agent follow these steps:

- Create a policy in Kibana > hamburger > Fleet > Policies > Create Agent policy
- Select the new policy
- Add integrations (for example, System and MySQL)
- Customize the integrations (for example, set the password for MySQL metric collection)
![MySQL user and password settings](https://raw.githubusercontent.com/DanRoscigno/Elastic-stack-monitoring-with-vagrant-and-ansible/main/images/MySQL-details.png)
- Retrieve the enrollment token from Kibana > hamburger > Fleet > Agents > Enrollment Tokens
    - Click the view icon (eye icon) for the new policy
    - Copy the enrollment token from the Secret column
    - Run the enroll command:
```
sudo /usr/bin/elastic-agent enroll \
    --url=https://192.168.33.30:8220 \
    --enrollment-token=<< token copied from above >> \
    --certificate-authorities=/vagrant/files/certs/ca/ca.crt
```

After a successful enrollment, run the agent:

```
sudo /usr/bin/elastic-agent run
```

or maybe:

```
sudo systemctl start elastic-agent
sudo systemctl status elastic-agent
```

# MySQL
If you added the MySQL integration you might as well test it.  Install MySQL on the machine where you added the second Elastic Agent:
```
sudo apt-get install mysql-server
```

When prompted for the root password for MySQL use the password that you put in the policy, or even better use somethign different.  If you used a different password then you will see errors in the MySQL logs (in Kibana).  Fix the error by updating the policy, and notice that when you save the update to the policy the agent process is soon restarted and the metrics will start flowing.
