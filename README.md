# Web/ELK Server - Cloud Configuration

### Background

Understanding and building on the cloud can be a massive undertaking for organizations, and it is becoming more and more necessary as technological requirements expand. In enterprise environments in particular, this capability can significantly reduce the risks involved with hosting infrastructures at scale.

In this project, Microsoft's Azure was used in order to build out a virtualized web infrastructure, which was ultimately automated using the IaaC (Infrastructure as Code) provisioner, Ansible. 

As a reference, the network diagram below should be used in order to visualize the cloud infrastructure and interactions. 

![](resources/azurenet.png)

### Overview

In this project, two load balanced web servers will be spun up, both hosting DVWA (Damn Vulnerable Web Application). In future projects, this site will be used to exploit its many vulnerabilities. 

Additionally, a monitoring server will also be spun up. For this project, the ELK stack was chosen due to its search functionality and informative GUI. 

Lastly, a Jumpbox server will be established, which will act as the main touch point for any and all internal network connections. This Jumpbox will also be the hub for Ansible automation.


### Getting Going

Setting up cloud infrastructure is often unique to the platform used, but a high level overview is as follows:

1. Create a resource group. This resource group will contain all infrastructure. In this case, the resource group was named "Red-Team".

2. Create a virtual network. Virtual networks will provide connections between any machines on the network. This project will contain two virtual networks called “Red-Team Virtual Network” and “ELK Virtual Network”.

3. Create Virtual Machines. Initially, the Jumpbox VM is created which will be the access point for all other VMs on the network. In order to secure the Jumpbox, an SSH public key is chosen instead of a password. Running `ssh-keygen` on the host machine will generate a public and a private key. These keys are stored in the ` ~/.ssh/` directory. The public key is copied into the key area in the VM setup. Additionally, a public IP will need to be established for the Jumpbox in order to allow SSH. 


![](resources/addsshkey.png)

4. Next, two more VMs are created, which will be the web servers. During set up, be sure to run that ssh-keygen command on the Jumpbox and not the host, since only the Jumpbox should have access. For now, these VMs do not need a public IP.

5. In order to connect, security rules will need to be created. By default, Azure will block all incoming traffic, so an inbound rule that allows TCP traffic on port 22 is created. 

The VMs can now be accessed via SSH. 

### Propping Up DVWA 

In order to spin up the site, docker.io will be utilized to compartmentalize the resources within the web server. In this case, Ansible will be used to automate the setup.

From the Jumpbox, docker can be installed with `sudo apt install docker.io`. Once installed and running, the Ansible container will be fetched with `sudo docker pull [Ansible container]`.

From here, the Ansible container is started and attached to using `docker run -ti [ansible container]:latest bash`. This will spawn a new bash session as root of the Ansible container.

In order for Ansible to run commands on other machines, it will need to be able to connect to the web servers via SSH. A new `ssh-keygen` will be needed but this time from the Ansible container. From there, resetting the SSH public keys on Azure for each web server will be needed. Once complete, test the SSH connection.

Navigate to `/etc/ansible/` from the Ansible container in order to configure Ansible. Start by editing the `hosts` file, and add both web server IP address followed by "ansible_python_interpreter=/usr/bin/python3".


![](resources/webserver_group.png)

Once complete, edit the `ansible.cfg` file and add the username needed to SSH into the web servers. This is done by commenting out the remote_user line.

Example:


    ```bash
    # What flags to pass to sudo
    # WARNING: leaving out the defaults might create unexpected behaviours
    #sudo_flags = -H -S -n

    # SSH timeout
    #timeout = 10

    # default user to use for playbooks if user is not specified
    # (/usr/bin/ansible will use current user as default)
    remote_user = sysadmin

    # logging is off by default unless this path is defined
    # if so defined, consider logrotate
    #log_path = /var/log/ansible.log

    # default module name for /usr/bin/ansible
    #module_name = command

    ```

Test if Ansible will be able to work by running `ansible all -m ping`. If `ansible_python_interpreter=/usr/bin/python3` is used, the output should look as follows:

```bash
10.0.0.5 | SUCCESS => {
"changed": false, 
"ping": "pong"
}
10.0.0.6 | SUCCESS => {
"changed": false, 
"ping": "pong"
}
```

Finally, the Ansible playbook will need to be configured. This file contains all of the commands needed to setup the DVWA site on both web servers simultaneously.

[pentest.yml](resources/pentest.yml)

Run the playbook with `ansible-playbook pentest.yml`. If successful, SSH into a web server and run `curl localhost/setup.php`. If any html is returned, the site was successfully installed. While inside the web servers, run `sudo docker update --restart always [container ID]`. This will automatically restart the DVWA container anytime the server is rebooted.

The final step is to setup the load balancer. In Azure, a new load balancer will need to be created. From there a health probe is setup to regularly check all the VMs, making sure they are able to receive traffic. Lastly, create a backend pool and add both web servers.

Once complete, add a network security group rule that allows TCP traffic on port 80. A web connection can now be established to the public IP of our load balancer as seen below. 

![](resources/dvwa.png)


### ELK Stack

Now that the site is up and running, it is time to set up the ELK stack.

In this case, a new virtual network will need to be created along with a new virtual server. These will be configured in essentially the same way as the web servers, so those steps will be skipped. 

Once the network is configured correctly, Ansible should also be configured which will enable automation of the ELK stack deployment.

1. SSH onto the Jumpbox. From there, attach to the Ansible container.

![](resources/attach_ansible_container.png)

2. Once attached, navigate back to the `/etc/ansible/hosts` file in order to add the new ELK VMs IP address as a new host group.

![](resources/add_elk_group.png)

3. Once the new server is added, the YAML playbook can be configured.

Note: At a high level, the playbook will install docker, pull the preconfigured elk container, establish the ports that ELK will run on, and enable docker on reboot.

[install-elk.yml](resources/install-elk.yml)

4. Once the playbook is created, run it with `ansible-playbook install-elk.yml`. If successful, it should return no error messages, as seen below.

![](resources/run_playbook.png)

In order to confirm a successful setup, SSH into the ELK VM and run `sudo docker ps -a` in order to see the status of all containers. In this case, the container has been "up 8 minutes" which is a good sign.

![](resources/confirm_running.png)

 
If security rules have been properly configured (a tcp connection needs to to be allowed on port 5601), Kibana should be accessible from a browser.

The URL will look like: `http://[your.ELK-VM.External.IP]:5601/app/kibana` as seen below.

![](resources/kibana_page.png)


### File/Metric Beat - Log Setup

In order for ELK to be useful as a monitoring service, data will need to be sent from the web servers. In order to do so, both Filebeat and Metricbeat are added to the web servers.

These two services are used to collect logs from specific files. In this case, logs will be collected from the DVWA Apache server and MySQL database. 

The instructions for installing these services can be found on the Kibana GUI under `Add Log Data > System Logs` and selecting the Linux instructions. Additionally, Metricbeat instructions can be found under `Add Metric Data > Docker Metrics`.

Filebeat and Metricbeat setups are very similar. 

1. SSH into Jumpbox. From there, attach to the Ansible container and navigate to the `/etc/ansible/` directory. From there, run the following curl commands in order to retrieve the configuration files for Filebeat and Metricbeat.

Filebeat: `curl https://gist.githubusercontent.com/slape/5cc350109583af6cbe577bbcc0710c93/raw/eca603b72586fbe148c11f9c87bf96a63cb25760/Filebeat > /etc/ansible/files/filebeat-config.yml`

Metricbeat: `curl https://gist.githubusercontent.com/slape/58541585cc1886d2e26cd8be557ce04c/raw/0ce2c7e744c54513616966affb5e9d96f5e12f73/metricbeat > /etc/ansible/files/metricbeat-config.yml`

![](resources/curlConfig.png)

2. Once the two configuration files are obtained, they will need to be edited. The Filebeat file will be much larger, however the items to change can be found on line #1106 and #1806. The IP addresses here will need to be the internal IP of the ELK machine. The Metricbeat file is much smaller, but can be configured in the same way.

![](resources/config_edit.png)

3. Once the config files are updated, the Ansible playbooks can be created. Add these playbooks into a new directory `/etc/ansible/roles`. In this new directory, `touch` the playbook files called `filebeat-playbook.yml` and `metricbeat-playbook.yml`.

4. These files will be similar to one another, but at a high level they will download the required .deb file, install the package using `dpkg`, copy the config files into the web servers, then run the setup commands that are specified in the instructions.

The two YAML playbooks can be found here:

- [filebeat-playbook.yml](resources/filebeat-playbook.yml)
- [metricbeat-playbook.yml](resources/metricbeat-playbook.yml)

Once everything is ready to go, run the playbooks from the `/roles/` directory. If successful, the output should be similar to below:


![](resources/filebeatpb.png)

Once both playbooks have been successfully run, verify the installation of these services by taking a look back at the instructions on the Kibana GUI. Scroll down to step #5, click on "Check Data". If that is successful, click on "View Dashboard". This will display the dashboards, which might look similar to the ones below.

![](resources/filebeatdash.png)
![](resources/metricbeatdash.png)

### Conclusion

Just like that, a secure and functioning cloud network has been established. This project utilized the following tools:

Azure: This platform allows virtual infrastructure to be built on Microsoft's platform without the need for physical machines. In this case a network of machines with their own firewall rules and functionalities was established.

Docker: This lightweight tool enables compartmentalized and virtualized services ontop of an OS. In this project, Ansible, a DVWA site, an ELK stack, and two logging services were all utilized via docker.io

Ansible: This IaaC tool allows automation by SSHing into preconfigured machines and configuring infrastructure via a .yml playbook. Useful in this case for deploying multiple mirrored web servers simultaneously

ELK: This open source stack allows us to monitor the web server's activity logs to give better insights into the site.

File/Metricbeat: These services allow the logs from the web servers to be sent over to the ELK stack to be further analyzed.

