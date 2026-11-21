# Amazon Web Services (AWS) - Intro to Cloud Computing

Cloud computing is a technology that allows businesses and individuals to access and use computing resources over the internet, without the need for owning or maintaining physical hardware. 
Amazon Web Services (AWS) is a leading provider of cloud computing services, offering a wide range of tools and platforms that enable businesses to deploy, scale, and manage their applications and data in the cloud.
With AWS, organizations can benefit from the flexibility, scalability, and cost-effectiveness of cloud computing, while focusing on their core business objectives.

## Region and Zones

AWS operates state-of-the-art, highly available data centers. Although rare, failures can occur that affect the availability of instances that are in the same location. If you host all of your instances in a single location that is affected by a failure, none of your instances would be available.

Each **Region** is designed to be isolated from the other Regions. This achieves the greatest possible **fault tolerance** and **stability**.


In this course we will use the `us-east-1` (N. Virginia) region *only*.

The `us-east-1` region (and all other regions) has multiple, isolated locations known as **Availability Zones**. The code for Availability Zone is its Region code followed by a letter identifier. For example, `us-east-1a`, `us-east-1b`, `us-east-1c`, etc.

*In the AWS web console, please make sure you are operating in the `us-east-1` region. You can check this in the top right corner of the console, next to your account name.*


## Launch a virtual machine (EC2 instance) 

Amazon EC2 (Elastic Compute Cloud) is a web service that provides resizable compute capacity in the cloud. 
It allows users to create and manage virtual machines, commonly referred to as "instances", which can be launched in a matter of minutes and configured with custom hardware, network settings, and operating systems.

![][networking_project_stop]

1. Open the Amazon EC2 console at [https://console\.aws\.amazon\.com/ec2/](https://console.aws.amazon.com/ec2/).

2. From the EC2 console dashboard, in the **Launch instance** box, choose **Launch instance**, and then choose **Launch instance** from the options that appear\.

3. Under **Name and tags**, for **Name**, enter a descriptive name for your instance\.

4. Under **Application and OS Images \(Amazon Machine Image\)**, do the following:

   1. Choose **Quick Start**, and then choose **Ubuntu**\. This is the operating system \(OS\) for your instance\.
   
5. Under **Instance type**, from the **Instance type** list, you can select the hardware configuration for your instance\. Choose the `t3.medium` instance type (the cheapest one). In Regions where `t3.medium` is unavailable, you can use a `t2.medium` instance.

6. Under **Key pair \(login\)**, choose **create new key pair** the key pair that you created when getting set up\.

   1. For **Name**, enter a descriptive name for the key pair\. Amazon EC2 associates the public key with the name that you specify as the key name\.

   2. For **Key pair type**, choose either **RSA**.

   3. For **Private key file format**, choose the format in which to save the private key\. Since we will use `ssh` to connect to the machine, choose **pem**.

   4. Choose **Create key pair**\.
   
     **Important**  
     This step should be done once! once you've created a key-pair, use it for every EC2 instance you are launching. 

   5. The private key file is automatically downloaded by your browser\. The base file name is the name you specified as the name of your key pair, and the file name extension is determined by the file format you chose\. Save the private key file in a **safe place**\.
      
      **Important**  
      This is the only chance for you to save the private key file\.

7. Next to **Network settings**, choose **Edit**\. 

   1. For **VPC** choose the default VPC fo your region.
   2. For **Subnet** choose any subnet you want. 
   3. Choose **Create security group** while providing a name other than `launch-wizard-x`. 


8. Under **Configure storage**, set the root volume size to **15 GiB**\.


9. Keep the default selections for the other configuration settings for your instance\.

10. Review a summary of your instance configuration in the **Summary** panel, and when you're ready, choose **Launch instance**\.

11. A confirmation page lets you know that your instance is launching\. Choose **View all instances** to close the confirmation page and return to the console\.

12. On the **Instances** screen, you can view the status of the launch\. It takes a short time for an instance to launch\. When you launch an instance, its initial state is `pending`\. After the instance starts, its state changes to `running` and it receives a public DNS name\.

13. It can take a few minutes for the instance to be ready for you to connect to it\. Check that your instance has passed its status checks; you can view this information in the **Status check** column\.

> [!NOTE]
> When stopping the instance, please note that the public IP address may change, while the private IP address remains unchanged


To connect to your instance, open the terminal in your local machine, and connect to your instance by: 

```shell
ssh -i "</path/key-pair-name.pem>" ubuntu@<instance-public-dns-name-or-ip>
```



[networking_project_stop]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_project_stop.gif


# Exercises

## :pencil2: EC2 instance security group

AWS uses **Security Groups** as virtual firewalls that control inbound and outbound traffic to your EC2 instances.
By default, a new security group allows **only port 22 (SSH)** for inbound traffic - everything else is blocked.

> Why do you think only port 22 is open by default?

To allow other types of traffic, you need to add inbound rules to the security group.
Here's how:

1. In the EC2 console, select your instance and open the **Security** tab.
2. Under **Security groups**, click on your security group's link.
3. Click **Edit inbound rules**.
4. Click **Add rule** and configure:
   - **Type**: select from the dropdown (e.g. `All ICMP - IPv4` for ping, or `Custom TCP` for a specific port)
   - **Source**: `0.0.0.0/0` (all IP addresses)
5. Click **Save rules**.

Now try it: try to `ping` your instance from your local machine:

```bash
ping <instance-public-ip>
```


## :pencil2: Deploy AgroPilot on an EC2 instance

Deploy your AgroPilot `registry-service` on the instance, and run it as a Linux service. You already registered that service with systemd on your own machine in the [Processes](linux_processes.md#exercises) tutorial.

**On your EC2 instance**, install the tools and get the code:

1. Install `git` and `curl`, then install `uv`:

   ```bash
   sudo apt update
   sudo apt install -y git curl
   curl -LsSf https://astral.sh/uv/install.sh | sh
   source $HOME/.local/bin/env
   uv --version
   ```

2. Clone your own copy of ArgoPilot, the same repository you created in the [Python environment](python_env.md) tutorial:

   ```bash
   git clone https://github.com/YOUR-USERNAME/ArgoPilot.git
   cd ArgoPilot/services/registry-service
   uv sync
   ```

   `uv sync` creates `.venv` and installs the locked packages, including uvicorn. It also installs the Python version the service requires.

3. Create the database. `registry.db` is not in git, so the clone does not contain it. If `seed_registry.sh` is not already in this folder, copy it here the same way you did in [Task P1](../tasks/p1_registry_api.md), then run:

   ```bash
   bash seed_registry.sh
   ```

Register the service with systemd. Create `/etc/systemd/system/registry.service` with the unit from the [Processes](linux_processes.md#exercises) exercise, using the paths on this instance and `--host 0.0.0.0`:

```text
[Unit]
Description=AgroPilot registry-service
After=network.target

[Service]
WorkingDirectory=/home/ubuntu/ArgoPilot/services/registry-service
ExecStart=/home/ubuntu/ArgoPilot/services/registry-service/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

`WorkingDirectory` is the folder that contains `main.py`. `ExecStart` is the `uvicorn` installed in that service's virtual environment. `--host 0.0.0.0` accepts connections on the instance's public address. The unit on your own machine uses `127.0.0.1`, which accepts connections only from that machine.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now registry.service
sudo systemctl status registry.service
```

`enable --now` starts the service, and starts it again on every boot. On the instance itself:

```bash
curl http://127.0.0.1:8000/health
```

A running service returns `{"status":"ok"}`. Follow the log with `journalctl -fu registry.service`, the same command as in the Processes exercise.

**Open port 8000 in your security group.** You did this for ping. Add an inbound rule:

- **Type**: `Custom TCP`
- **Port range**: `8000`
- **Source**: `0.0.0.0/0`

**On your local machine**, call the service through the public address:

```bash
curl http://<your-instance-public-ip>:8000/health
curl http://<your-instance-public-ip>:8000/v1/crops
```

Stop the instance in the EC2 console when you are done. The service is enabled, so the next boot starts `registry-service` again. The public IP address may change after a stop, as noted above.
