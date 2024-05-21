---
title: Implementing Wazuh SIEM To My Home Lab
date: 2024-05-21 00:00:00 -600
categories: [Cybersecurity, Home Lab]
tags: [homelab, active directory, cybersecurity]     # TAG names should always be lowercase
---

In this post, I will demonstrate how I set up Wazuh SIEM and EDR platform, and put it to use with my Active Directory home lab environment.

The Wazuh SIEM server will be hosted on a cloud VM instead of a VM on my computer. Running Wazuh server requires 4 vCPU cores and 8 GB of memory. I can create a VM with lower resources to host Wazuh server, but running the server with 3 Windows VM simultaneously is not stable. Therefore, I will have to rely on a cloud provider to spin up a VM for Wazuh server.

After the Wazuh server is deployed, I am going to install Wazuh agent on all Windows VM, so Windows VM telemetry will be sent to Wazuh to collect events and generate alerts when security incidents are detected.

## Creating a cloud-based VM

For cloud provider, I choose DigitalOcean, because it provides me $200 of credit for use within 60 days from the day I sign up, which is plenty compared to other cloud providers.

After signing up, I create a Droplet (VM in DigitalOcean's term) to run Ubuntu Server 24.04. Hardware for the Droplet is configured to 4 vCPU and 8GB of RAM. Name the VM, then create it.

## Putting the cloud VM behind a firewall

By default, a cloud-based VM is assigned with a public IP address, and it is exposed to the Internet. All traffic is allowed to the VM, and it is note safe, susceptible to attackers aroound the world. The VM is needed to put behind a firewall to keep it secure. DigitalOcean provides network firewall for us to configure and put droplet(s) behind it.

I create firewall rules for the VM to only be accessible from my home network (allow traffic from my home's public IP address). Also, I create rules to only allow traffic to ports used by Wazuh.

Once the rules are created, select droplet to apply firewall rules. Only traffic specified in the rules is allowed to communicate with the VM, and other traffic will be denied.

## Installing Docker and Docker Compose

1. Open terminal, then access the server via SSH. For DigitalOcean VM, the default user is root.
2. Run ```sudo apt-get update && sudo apt-get upgrade```.
3. Increase ```max_map_count``` value of the server VM so Wazuh can run properly. Run this command to increase ```max_map_count``` value.
    ```bash
    sysctl -w vm.max_map_count=262144
    ```
    Also, set the value permanently by opening ```/etc/sysctl.conf``` file, then add ```vm.max_map_count = 262144``` at the end, save the file, and reboot the server.
4. Install Docker using installation script.
    
    ```bash
    curl -sSL https://get.docker.com/ | sh
    ```
5. Start Docker service.
    ```bash
    systemctl start docker
    ```
6. Download Docker Compose.
    ```bash
    curl -L "https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    ```
7. Grant execution permissions.
    ```bash
    chmod +x /usr/local/bin/docker-compose
    ```
8. Verify that Docker Compose is good after installation
    ```bash
    docker-compose --version
    ```

## Deploying Wazuh Docker

1. Clone Wazuh Docker from Github repository.
    ```bash
    git clone https://github.com/wazuh/wazuh-docker.git -b v4.7.4
    ```
    Then, ```cd``` to ```wazuh-docker/single-node``` directory.

2. Generate self-signed certificates.
    ```bash
    docker-compose -f generate-indexer-certs.yml run --rm generator
    ```
3. Start Wazuh Docker deployment.
    ```bash
    docker-compose up -d
    ```
    Verify that Wazuh is up and running using this command. There should be 3 Wazuh components listed in the output.
    ```bash
    docker-compose ps
    ```

4. Open up web browser, put in the Wazuh server's IP address, then Enter. There will be "invalid certificates", or "Your connection is not private" warning showing up, because the certificates are self-signed, which is all right. Go ahead and proceed to the site.

5. Log into Wazuh using default credentials, ```admin``` and ```SecretPassword```. After logged in, Wazuh Dashboard will show up.

    ![wazuh](/assets/img/screenshots/wazuh01.PNG)

## Changing Wazuh default password

1. Stop Wazuh process.
    ```bash
    docker-compose down
    ```
2. Run this command to open hash generator.
    ```bash
    docker run --rm -ti wazuh/wazuh-indexer:4.7.4 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
    ```
    The hash generator will prompt the password input. Put in the new password, and the hash of the passowrd will be generated. Copy the generated has, and use this for the next step.

3. Open ```config/wazuh_indexer/internal_users.yml``` file. Replace hash for ```admin``` and ```kibanaserver``` user, and save the file.

    ![replace-hash](/assets/img/screenshots/wazuh-hash.PNG)

4. Open ```docker-compose.yml```. Under ```wazuh.manager```, change ```INDEXER_PASSWORD``` value.

    ![change-password](/assets/img/screenshots/wazuh-manager-password.PNG)

    Under ```wazuh.dashboard```, change value of ```INDEXER_PASSWORD``` and ```DASHBOARD_PASSWORD```.

    ![change-password](/assets/img/screenshots/wazuh-dashboard-passowrd.PNG)

    Then, save the file.

5. Start Wazuh back up again.
    ```bash
    docker-compose up -d
    ```

6. Run ```docker exec -it single-node-wazuh.indexer-1 bash``` to enter the Wazuh Indexer container, then set the following variables.
    ```bash
    export INSTALLATION_DIR=/usr/share/wazuh-indexer
    CACERT=$INSTALLATION_DIR/certs/root-ca.pem
    KEY=$INSTALLATION_DIR/certs/admin-key.pem
    CERT=$INSTALLATION_DIR/certs/admin.pem
    export JAVA_HOME=/usr/share/wazuh-indexer/jdk
    ```
7. Wait for a few minutes for the Wazuh Indexer to initialize properly, then run this command to apply all changes.
    ```bash
    bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd /usr/share/wazuh-indexer/opensearch-security/ -nhnv -cacert  $CACERT -cert $CERT -key $KEY -p 9200 -icl
    ```

8. Exit Wazuh Indexer container. Log in to Wazuh Dashboard using the new password.

## Deploying Wazuh Agent on Windows VM

1. Log into Wazuh in web browser.

2. Click Wazuh logo at the top left, then click "Agent".

3. Click "Deploy new agent".

4. In "Deploy new agent" page:
    * Select Windows MSI package.
    * Set server address to the IP address of Wazuh server.
    * Assign agent name to hostname of the Windows VM. For example, ```LAB-Client01``` for LAB-Client01 VM.
    * Copy the generated command. The command will look like this.

        ```powershell
        Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.4-1.msi -OutFile ${env.tmp}\wazuh-agent; msiexec.exe /i ${env.tmp}\wazuh-agent /q WAZUH_MANAGER='[Wazuh server IP address]' WAZUH_AGENT_NAME='[hostname]' WAZUH_REGISTRATION_SERVER='[Wazuh server IP address]' 
        ```
5. In Windows VM, open PowerShell as admin, paste the command copied from step 4, then run.
6. Start Wazuh service on the Windows VM.
    ```powershell
    NET START WazuhSvc
    ```
7. Open Services, scroll down the list and see if Wazuh service is running.

8. Repeat step 1-7 on other two Windows VM.

9. Back to Wazuh Dashboard. The number of total agents should be 3. If those Windows VM are still up and running, the number of avtive agents will also be 3. That means the Wazuh agents are checking in with the Wazuh server, forwarding telemetry.

## Wrapping up

In this post, Wazuh SIEM server has been deployed on a cloud VM, and Wazuh agents are also deployed on Windows VMs.

Next up, I am going to generate telemetry on Windows VMs to trigger some alerts on Wazuh. Also, I am going to create custom rules on Wazuh to detect and alert when specific events occur.