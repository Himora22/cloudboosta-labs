# AWS EC2 — Custom VPC Networking, Web Server Hosting & Jenkins Deployment

**Programme:** Cloudboosta CBA Training Programme — Feb Cohort 1 (Cloud Engineering track)
**Topic:** Amazon EC2 — custom VPC/subnet/route table design, launching instances, hosting a web server, and deploying Jenkins
**Environment:** AWS Management Console, Linux terminal (Amazon Linux 2023, Ubuntu 24.04)

## Objective

Design a custom VPC with public/private subnets, route tables, an internet gateway and a network ACL, then launch EC2 instances into it for two separate purposes: hosting a static website behind Apache, and deploying a self-managed Jenkins server reachable over the internet.

## Skills demonstrated

- Designing a custom VPC (CIDR block, subnets, route tables, internet gateway, network ACL)
- Configuring public subnet auto-assign IP settings
- Launching EC2 instances into a specific VPC/subnet with a chosen security group
- Connecting to an instance via EC2 Instance Connect (browser-based terminal)
- Installing and running Apache (httpd) on Amazon Linux, and Jenkins on Ubuntu
- Editing security group inbound rules to open the ports a service actually needs
- Diagnosing a package-manager installation failure and working around it with a vendor package
- Terminating instances (and cleaning up a VPC) to avoid ongoing charges

## Task 1: Custom VPC + EC2 instance hosting a web server

I logged into my IAM user (not the root account) and created a VPC through the VPC wizard, selecting "VPC and more" so the wizard would also provision subnets, route tables, an internet gateway and an S3 gateway endpoint alongside the VPC itself.

![VPC and more wizard preview](screenshots/01-vpc-wizard-preview.png)

![VPC wizard preview scrolled — route tables and network connections](screenshots/02-vpc-wizard-preview-scrolled.png)

The wizard completed successfully, creating the VPC, two subnets (one public, one private) in `eu-west-2a`, an internet gateway, two route tables, and associating the S3 gateway endpoint with the private subnet's route table.

![Create VPC workflow — success](screenshots/03-vpc-workflow-success.png)

Here's the VPC it created — a `/16` CIDR block (`10.0.0.0/16`) with DNS hostnames and DNS resolution enabled:

![VPC summary](screenshots/04-vpc-summary.png)

The two subnets it created, one public and one private, each a `/20`:

![Subnets created](screenshots/05-subnets-created.png)

The two route tables — one for the public subnet routing out through the internet gateway, one for the private subnet:

![Route tables created](screenshots/06-route-tables-created.png)

The internet gateway, attached to the VPC:

![Internet gateway created](screenshots/07-internet-gateway-created.png)

And the network ACL the wizard attached to both subnets:

![Network ACL created](screenshots/08-network-acl-created.png)

By default a new subnet doesn't auto-assign public IPv4/IPv6 addresses to instances launched into it, even if its route table sends traffic to an internet gateway. To fix that, I went to **VPC → Subnets**, selected the public subnet (`cba-subnet-public1-eu-west-2a`), opened **Actions → Edit subnet settings**, and under **Auto-assign IP settings** ticked both **Enable auto-assign IPv6 address** and **Enable auto-assign public IPv4 address**, then saved. This is a per-subnet setting — without it, instances launched into this subnet would only get a private IP, regardless of the route table.

![Auto-assign IP settings enabled on the public subnet](screenshots/09-subnet-auto-assign-ip.png)

With the network in place, I moved to EC2 to launch an instance. For the AMI I used the default **Amazon Linux 2023** image (Free tier eligible), and for instance type I kept `t3.micro` (also free-tier eligible). In **Network settings** I selected the VPC I'd just created and explicitly chose the **public** subnet (so the instance would be reachable from the internet) — with the subnet's auto-assign setting already enabled from the previous step, its **Auto-assign public IP** field defaulted to **Enable**, which I left as is. I created a security group allowing SSH, HTTP and HTTPS from `0.0.0.0/0`, kept the number of instances at 1, and launched.

![Instance launch — success](screenshots/10-instance-launch-success.png)

Once it was running, the status check showed 3/3 checks passed:

![Instance list — 3/3 checks passed](screenshots/11-instance-3-3-checks.png)

The instance summary shows its public/private IPv4 addresses, VPC ID and subnet ID:

![Instance summary](screenshots/12-instance-summary.png)

To get a shell on the instance, I selected it in the instance list and clicked **Connect**, which opened the **EC2 Instance Connect** tab. It defaulted to **Connect using a Public IP** with the instance's public IPv4 address already filled in, and a **Username** field pre-populated with `ec2-user` — the standard default for Amazon Linux AMIs. I left both at their defaults and clicked **Connect**, which opened a new browser tab running an SSH session straight to the instance (no local SSH client, key file, or terminal app needed, since EC2 Instance Connect pushes a short-lived public key to the instance for the session and connects over the browser).

![Connect dialog](screenshots/13-connect-dialog.png)

This dropped me into an Amazon Linux 2023 shell as `ec2-user`:

![Terminal opened](screenshots/14-terminal-opened.png)

I switched to the root user and updated the system before installing anything:

```bash
sudo su
yum update -y
```

![Root user, system update](screenshots/15-root-user-update.png)

Then installed the Apache web server:

```bash
yum install httpd -y
```

![Apache (httpd) installed](screenshots/16-apache-installed.png)

I moved into the web root, created an `index.html` file and opened it in `vi`:

```bash
cd /var/www/html/
touch index.html
vi index.html
```

![Creating index.html](screenshots/17-create-index-html.png)

I pasted in a small HTML/CSS portfolio page (saved with `Esc` then `:wq`):

![index.html contents in vi](screenshots/18-index-html-contents.png)

I confirmed the file was saved correctly with `cat`:

```bash
cat index.html
```

![Confirming index.html with cat](screenshots/19-cat-index-html.png)

Then started the Apache service, checked its status, and enabled it to start on boot:

```bash
service httpd start
service httpd status
chkconfig on
```

![httpd started, status active, enabled on boot](screenshots/20-httpd-started.png)

Pasting the instance's public IP into a browser served the page directly from the EC2 instance:

![Website loading in the browser](screenshots/21-website-loading.png)

Once I'd confirmed everything worked, I terminated the instance to avoid being billed for it.

![Terminating the instance](screenshots/22-terminate-instance.png)

## Task 2: Custom VPC + EC2 instance deploying a self-managed Jenkins server

For the second exercise I took on the role of a cloud solutions architect asked to stand up a Jenkins server on AWS. I created a second, separate custom VPC (named `jenkins`) the same way — via the VPC wizard, choosing an Amazon-provided IPv6 CIDR block and a single Availability Zone.

![Create VPC form](screenshots/23-jenkins-vpc-form.png)

The wizard again created the VPC along with its subnets, route tables, internet gateway and S3 endpoint.

![VPC workflow — success](screenshots/24-jenkins-vpc-workflow-success.png)

The resulting VPC:

![VPC summary](screenshots/25-jenkins-vpc-summary.png)

Its two subnets:

![Subnets created](screenshots/26-jenkins-subnets-created.png)

Its route tables:

![Route tables created](screenshots/27-jenkins-route-tables.png)

And its internet gateway:

![Internet gateway created](screenshots/28-jenkins-igw.png)

As with Task 1, the public subnet didn't auto-assign public IPs by default. I went to **VPC → Subnets**, selected `jenkins-subnet-public1-eu-west-2a`, opened **Actions → Edit subnet settings**, and ticked both **Enable auto-assign IPv6 address** and **Enable auto-assign public IPv4 address**, then saved:

![Enabling auto-assign IP on the public subnet](screenshots/29-jenkins-subnet-auto-assign.png)

![Auto-assign settings saved](screenshots/30-jenkins-subnet-auto-assign-saved.png)

Moving to EC2, I launched an instance named `jenkin_instance`. For the AMI I chose **Ubuntu Server 24.04 LTS** (rather than Amazon Linux, since the Jenkins install steps I'd been given targeted Debian/Ubuntu's `apt` package manager), and kept the instance type at `t3.micro`, free-tier eligible with 8 GiB of EBS storage.

![Launch instance — name and Ubuntu AMI](screenshots/31-jenkins-instance-ubuntu-ami.png)

I created a new key pair, `jenkin-keypair.pem`, downloading it to keep safe for SSH access:

![Key pair creation](screenshots/32-jenkins-keypair.png)

In network settings, I selected the `jenkins-vpc` and its public subnet, so the instance would be reachable from the internet. With the subnet's auto-assign setting already enabled from the previous step, **Auto-assign public IP** on the instance itself defaulted to **Enable**, which I left as is:

![Network settings — VPC and public subnet selected](screenshots/33-jenkins-network-settings.png)

For the security group, I opened inbound rules for SSH (22) and, since Jenkins serves its web UI on port 8080, a custom TCP rule for port 8080 — both from `0.0.0.0/0`:

![Security group — SSH and port 8080](screenshots/34-jenkins-security-group.png)

With a single instance selected, I launched it:

![Launch summary](screenshots/35-jenkins-launch-summary.png)

![Instance launch — success](screenshots/36-jenkins-instance-launch-success.png)

The instance summary once it was running:

![Instance summary](screenshots/37-jenkins-instance-summary.png)

Status checks passed 3/3:

![Instance list — 3/3 checks passed](screenshots/38-jenkins-3-3-checks.png)

To connect, I selected the instance and clicked **Connect**, opening the **EC2 Instance Connect** tab. This time the pre-filled **Username** defaulted to `ubuntu` — the standard default for Ubuntu AMIs, as opposed to `ec2-user` on Amazon Linux — with **Connect using a Public IP** and the instance's public IPv4 address already selected. I clicked **Connect** to open the browser-based SSH session:

![Connect dialog](screenshots/39-jenkins-connect-dialog.png)

This dropped me into an Ubuntu 24.04.3 LTS shell:

![Terminal opened — Ubuntu](screenshots/40-jenkins-terminal-opened.png)

I followed a set-up script (from the lab's GitHub reference) that updates the OS, adds the Jenkins APT repository, installs Java, then installs Jenkins itself:

```bash
# command to update the OS
sudo apt-get update -y

# command to add the Jenkins repo
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# command to import a key file from Jenkins-CI to enable installation from the package
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# command to update the OS
sudo apt-get update -y

# command to install java
sudo apt install openjdk-17-jre -y

# command to install Jenkins
sudo apt-get install jenkins -y
```

![GitHub reference script for installing Jenkins](screenshots/41-jenkins-github-script.png)

Running `sudo apt-get install jenkins -y` failed:

```
Package jenkins is not available but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source
E: Package 'jenkins' has no installation candidate
```

After some research I bypassed the repo and installed Jenkins directly from the official `.deb` package instead:

```bash
wget https://pkg.jenkins.io/debian-stable/binary/jenkins_2.452.3_all.deb
sudo dpkg -i jenkins_2.452.3_all.deb
sudo apt -f install -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Pointing my browser at `http://<instance-public-ip>:8080` didn't load at first. To rule out the security group as the cause, I added HTTP and HTTPS rules to it, alongside the existing SSH and custom-8080 rules.

![Security group updated with HTTP and HTTPS rules](screenshots/42-jenkins-sg-http-https.png)

With that in place, the Jenkins unlock screen loaded, and I pasted in the initial admin password I'd read from the terminal:

![Unlock Jenkins](screenshots/43-jenkins-unlock-screen.png)

From there I chose to install the suggested plugins to finish setting up the server:

![Customize Jenkins — install suggested plugins](screenshots/44-jenkins-customize-screen.png)

Once I'd verified Jenkins was reachable and working, I terminated the instance and deleted the VPC so nothing kept running (and billing) unattended.

## Key concepts

| Concept | Description |
|---|---|
| VPC wizard ("VPC and more") | Provisions a VPC together with subnets, route tables, an internet gateway, and an S3 gateway endpoint in one workflow, rather than creating each piece by hand. |
| Public vs. private subnet | A subnet is "public" only because its route table sends `0.0.0.0/0` traffic to an internet gateway; launching an instance there with auto-assign public IP enabled makes it reachable from the internet. |
| Auto-assign public IP | A per-subnet setting; even in a public subnet, an instance won't get a public IPv4 address unless this is enabled (at the subnet level, the instance level, or both). |
| Security group vs. NACL | A security group is stateful and attached to an instance's network interface; a network ACL is stateless and attached to a subnet, evaluating rules by number in order. |
| EC2 Instance Connect | A browser-based SSH client built into the EC2 console — no local SSH client or key file needed for basic access, provided the security group allows port 22. |
| Package availability differs by distro | `yum install httpd` worked cleanly on Amazon Linux, but `apt-get install jenkins` failed on Ubuntu because the package wasn't resolvable from the default repos — sometimes the fix is to add the vendor's own repository or install their `.deb`/`.rpm` directly. |
| Opening only the ports a service needs | Jenkins' web UI needed port 8080 (and HTTP/HTTPS to browse it), separate from SSH on port 22 — each service's port has to be explicitly allowed in the security group. |
| Terminating instances / deleting VPCs | EC2 instances (and some networking resources) bill while they exist; tearing down lab resources once verified avoids unexpected charges. |

## What I learned

Building the VPC by hand really only clicked once I could see the pieces the wizard created — the two subnets, the route tables pointing different places, the internet gateway, and the NACL — and how they all reference each other by ID rather than by name underneath the console's friendly labels. The most useful troubleshooting moment was the Jenkins install failure: `apt-get install jenkins` failing with "no installation candidate" was a good reminder that a package manager only knows about what's in its configured repositories, and that adding the Jenkins project's own APT repo (or falling back to their `.deb` package directly) is the standard fix rather than something being wrong with the instance itself. Similarly, getting a blank browser when hitting the Jenkins port taught me to check the security group before assuming the service itself was broken — the server was running the whole time, it just wasn't allowed to receive that traffic yet.
