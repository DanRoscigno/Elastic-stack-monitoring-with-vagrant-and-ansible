# Goal
A repeatable, semi-automated Elastic Stack deployment that is configured to support
Stack Monitoring.

![Node advanced monitoring dashboard](https://raw.githubusercontent.com/DanRoscigno/Elastic-stack-monitoring-with-vagrant-and-ansible/main/images/StackMonitoring.png)

# Background
Most of my work with the Elastic Stack has been related to Kubernetes or Elastic Cloud
and it has been a long time since I deployed the stack myself.  This is a set of
configurations that allows me to build out several virtual machines (using Vagrant and 
Virtualbox), and then install the stack onto those VMs in a repeatable fashion (via 
Ansible) so that I can test ideas, write documentation, etc.

I had never used Vagrant nor Ansible before writing this, and I am sure that what I have
done is not the most efficient.  Here are my requirements:
 - No passwords in this repo
 - TLS used between stack components
 - TLS used between the browser and Kibana
 - Stack monitoring using Beats
 - Support for both Beats and Logstash to ingest non-stack-monitoring data
 - No simulated data 

Note: Not all of these requirements are met on day one, there are no logs being collected, 
and no non-stack-monitoring data is being collected.

Note: The Vagrantfile and inventory are from the blog 
[ELK Stack with Vagrant and Ansible](https://github.com/ashokc/ELK-Stack-with-Vagrant-and-Ansible), which cites [Ansible Skeleton](https://github.com/bertvv/ansible-skeleton).

This project uses the following Ansible roles:

- [elastic.elasticsearch](https://galaxy.ansible.com/elastic/elasticsearch/)
- [elastic.beats](https://galaxy.ansible.com/elastic/beats)


# Prerequisites:
 - Python 3
 - Vagrant
 - Virtualbox
 - Ansible
 - Sufficient CPU & RAM to build 5 vms (adjust the resource allocation in 'inventory.yml')

# Commands for macOS Catalina setup

See the file [MacCatalinaSetup.md](https://github.com/DanRoscigno/Elastic-stack-monitoring-with-vagrant-and-ansible/blob/main/MacCatalinaSetup.md) for details on macOS Catalina (probably also other versions) setup.

# Set the desired Elastic Stack version
Edit `group_vars/all.yml` and set `es_version`.

# Create CA and cert files for TLS
Note: This requires an Elasticsearch install
Note: Use this [support blog](https://www.elastic.co/blog/configuring-ssl-tls-and-https-to-secure-elasticsearch-kibana-beats-and-logstash) as a reference!!

An instance.yml file is provided.  If you add more nodes to your cluster add more entries to the instance.yml file
and rebuild the certificates.  The file looks like this:
```
instances:
  - name: "es-master-1" 
    ip: 
      - "192.168.33.25"
  - name: "es-data-1"
    ip:
      - "192.168.33.26"
  - name: "es-data-2"
    ip:
      - "192.168.33.27"
  - name: 'kibana-1'
    ip:
      - "192.168.33.28"
  - name: 'logstash-1'
    ip:
      - "192.168.33.29"
  - name: 'fleet-1'
    ip:
      - "192.168.33.30"
  - name: 'fleet-2'
    ip:
      - "192.168.33.31"
```

When this file gets used a separate certificate is created for each node, and all are signed with the CA that is also created.   By adding the IP address and hostname to each certificate Logstash will be able to connect.

First you need an Elasticsearch install.  You can download and extract Elasticsearch and run the certificate binary from that download.

Next generate a certificate authority (ca):
Note: You will be asked for an output filename.  The file will be produced in the `elasticsearch` directory.  In the example below, this would be `/usr/share/elasticsearch/`
```
/usr/share/elasticsearch/bin/elasticsearch-certutil ca --pem
```

Unzip the output zip file, as you will use the files in the zip file as an input to the next command.


This is the command to build the certificates:
```
/usr/share/elasticsearch/bin/elasticsearch-certutil cert \
  --ca-cert /usr/share/elasticsearch/ca/ca.crt \
  --ca-key /usr/share/elasticsearch/ca/ca.key \
  --pem \
  --in /tmp/instances.yml \
  --out /tmp/test4.zip
```

Make sure to unzip the two output files into a directory that is made available to the virtual machines built with Vagrant.  If you are working in the directory `Elastic-stack-monitoring-with-vagrant-and-ansible` then `Elastic-stack-monitoring-with-vagrant-and-ansible/files/certs` would be a good choice and work with the commands used in this tutorial.

# Build the VMs with Vagrant and Virtualbox
The `vagrant up` command shown below builds Virtualbox VMs that are specified in the file `Vagrantfile` in the current directory.  Change directory to the content of the GitHub repo for this tutorial and tun the following command:
```
vagrant up --no-provision
```

# Add secrets to Ansible Vault
By using the Ansible vault you can avoid storing your passwords in a configuration 
repository.  When you run the `ansible-vault create` command you will be in your 
default editor, add keys and values, and then substitute the keys in the Ansible
templates and playbooks.  For this tutorial store a password and a Kibana encryption
key.  Multiple passwords and API Keys should be stored in the vault.

The playbooks and templates used in this tutorial require two keys:
 - `elastic_pass`
 - `kibana_encryptionKey`

This is what your vault could contain:
```
elastic_pass: sup3rS3cr3t
kibana_encryptionKey: something_at_least_32_characters
```

You will use the password that you set in the vault when you run the command 
`elasticsearch-setup-passwords` later.  

Here is the command to create the file in the `vars/` directory:
```
ansible-vault create vars/credentials.yml
```

Note: You can edit the credentials.yml file with `ansible-vault edit vars/credentials.yml` or 
display it with `ansible-vault view vars/credentials.yml`.

# Monitoring with Beats

Deploy the Beats that will monitor  the Elastic Stack first as the Beats will be configured
during the deployment of Elasticsearch, Kibana, and Logstash.

Install Metricbeat

Note: Metricbeat will not connect until the passwords are set in Elasticsearch, which you will do in a few steps.  You will be prompted for the password that you used to encrypt the vault.

```
ansible-playbook -v -i inventory.yml \
   deploy-beats.yml \
   --ask-vault-pass
```

This command takes a while to run.  It is deploying Metricbeat on all fo the virtual machines.  At the end the summary will tell you how things went.  You should see many (25) `OK`, and many (12) `skipped` statuses.  You can scroll back through the output to see the details.

# Deploy Elasticsearch
Run the Ansible playbook `deploy-elasticsearch.yml`
```
ansible-playbook -v -i inventory.yml \
   deploy-elasticsearch.yml \
   --ask-vault-pass

```

This command takes a while to run.  It is deploying Metricbeat on all fo the virtual machines.  At the end the summary will tell you how things went.  You should see many (49 or so) `OK`, 19 or 20 `changed`, 100+ `skipped`, and 1 `ignored` statuses.  You can scroll back through the output to see the details.

# Verify TLS

There are several curl commands, and as we have enabled security you will be prompted for a password.  The default password for the `elastic` user is `changeme`.  You will change it in one of the steps coming up.

Use the `ca/ca.crt` file that is in `files/certs/ca/ca.crt`.  You may be tempted to add a `-k` to all of your curl commands to avoid verifying the SSL cert, but test it at least once using the ca.crt
```
curl --cacert files/certs/ca/ca.crt \
   -u elastic \
   'https://192.168.33.25:9200/_cat/nodes?v'
```

Sample output:
```
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role  master name
192.168.33.25       12          81     2    0.00    0.09     0.12   ilmr       *    es-master-1
192.168.33.27       10          80     4    0.02    0.14     0.11   cdfhilrstw -    es-data-2
192.168.33.26       27          80     3    0.31    0.17     0.13   cdfhilrstw -    es-data-1
```

# Check the license status
```
curl --cacert files/certs/ca/ca.crt \
  -u elastic 'https://192.168.33.25:9200/_license'
```

Sample output (look for a `type` of `basic`):
```
Enter host password for user 'elastic':
{
  "license" : {
    "status" : "active",
    "uid" : "9fbe79c9-1016-4a98-81a2-9750b345bbd9",
    "type" : "basic",
    "issue_date" : "2021-06-14T15:32:51.440Z",
    "issue_date_in_millis" : 1623684771440,
    "max_nodes" : 1000,
    "issued_to" : "test-cluster",
    "issuer" : "elasticsearch",
    "start_date_in_millis" : -1
  }
}
```

# Add a platinum license

Acquire a license from Elastic.  You can activate a trial license from within Kibana if you do not have a physical license file.

```
curl --cacert files/certs/ca/ca.crt \
  -XPUT -u elastic \
  'https://192.168.33.25:9200/_xpack/license' \
  -H "Content-Type: application/json" \
  -d @/Users/droscigno/Downloads/license-release-stack-platinum.json 
```

# Verify the license change (look for the license type of `platinum`)
```
curl --cacert files/certs/ca/ca.crt \
  -u elastic 'https://192.168.33.25:9200/_license'
```

# Check cluster health
```
curl --cacert files/certs/ca/ca.crt \
  -X GET -u elastic \
  'https://192.168.33.25:9200/_cluster/health?level=shards&pretty'
```

# Check that the certificates are in use
Verify that the IP Addresses that you added to the certificates are there
```
openssl s_client -connect 192.168.33.25:9200 \
  </dev/null 2>/dev/null | openssl x509 -noout \
  -text | egrep "Subject: CN=|IP Address"

openssl s_client -connect 192.168.33.26:9200 \
  </dev/null 2>/dev/null | openssl x509 -noout \
  -text | egrep "Subject: CN=|IP Address"

openssl s_client -connect 192.168.33.27:9200 \
  </dev/null 2>/dev/null | openssl x509 -noout \
  -text | egrep "Subject: CN=|IP Address"
```

Look for the names and IP addresses from the `inventory.yml` file.  For example:
```
        Subject: CN=es-master-1
                IP Address:192.168.33.25
```

# Set passwords
Use a Vagrant command to run the `elasticsearch-setup-passwords` command on `es-master-1`.  In the sample credentials.yml file created earlier with the `ansible-vault create vars/credentials.yml` command I showed a single password.  For test environments I use a single password for all of the default accounts.  Please do not do this in production, modify the scripts used to deploy your system to use the various account names and set proper individual passwords.

```
vagrant ssh -c \
  "sudo --user=elasticsearch /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive" \
  es-master-1
```

Sample output:
```
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y


Enter password for [elastic]: 
Reenter password for [elastic]: 
Enter password for [apm_system]: 
Reenter password for [apm_system]: 
Enter password for [kibana_system]:
.
.
.
```

# Deploy Logstash

Run the Ansible playbook `deploy-logstash.yml`
```
ansible-playbook -v -i inventory.yml \
   deploy-logstash.yml \
   --ask-vault-pass
```

# Deploy Kibana

Run the Ansible playbook `deploy-kibana.yml`
```
ansible-playbook -v -i inventory.yml \
   deploy-kibana.yml \
   --ask-vault-pass
```

# Open a browser to stack monitoring in Kibana

Depending on the CPU resources provided to your Kibana virtual machine your
Kibana server may take a minute or two to become ready to accept connections.

Because the TLS certificate used to encrypt the connection between your 
browser and the Kibana server is "self signed" you will be notified that the 
connection is not private and have to allow your browser to trust the connection.

When you open Stack Monitoring you will likely see that all of your Elastic 
Stack components (except the Beats) are represented on the monitoring page.
If they are not, you may have to enter setup mode.  Within setup mode you 
will be prompted to download and install Metricbeat.  Skip this and close the 
setup wizard and exit setup and you will likely have monitoring data 
available.

Note: The elastic password that you used in the `curl` commands earlier was the default one (`changeme`); when you log in this time through the browser use the account name `elastic`, and the password that you put in the `credentials.yml` file.

https://192.168.33.28:5601/app/monitoring

# Fleet server and Elastic Agent

Add [Fleet server](https://github.com/DanRoscigno/Elastic-stack-monitoring-with-vagrant-and-ansible/blob/main/Fleet.md)

# Cleanup 
To remove everything:

```
vagrant destroy
```

To remove a single VM:

```
vagrant destroy <<machine name from inventory.yml>>
```
