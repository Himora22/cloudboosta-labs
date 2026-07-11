# High Availability & Auto Scaling — VPC, Application Load Balancer, and EC2 Auto Scaling Group

**Programme:** Cloudboosta CBA Training Programme — Feb Cohort 1 (Cloud Engineering track)
**Topic:** Provisioning a highly available infrastructure on AWS
**Environment:** AWS Management Console (eu-west-2 / Europe London)

## Objective

Design and provision a highly available infrastructure on AWS: a custom VPC spanning two Availability Zones, an internet-facing Application Load Balancer to distribute traffic, and an Auto Scaling Group that keeps the application available by automatically replacing unhealthy or terminated EC2 instances.

## Skills demonstrated

- Creating a custom VPC with public and private subnets across two Availability Zones using the "VPC and more" wizard
- Enabling auto-assign public IPv4 addressing on public subnets
- Provisioning an internet-facing Application Load Balancer (ALB) with its own security group
- Creating a target group with a custom health check path and advanced health check thresholds
- Building a Launch Template with a specific AMI, instance type, key pair, security group, and EC2 user data script
- Creating an Auto Scaling Group attached to an existing load balancer target group, with Elastic Load Balancing health checks
- Verifying Auto Scaling Group activity history and instance health via the console
- Testing high availability by manually terminating instances and observing automatic replacement
- Verifying end-to-end traffic distribution via the load balancer's public DNS name
- Cleaning up a full stack of dependent AWS resources in the correct order

## Step 1: Create the VPC

The first resource needed was the custom network. In the VPC console, I used **Create VPC** with the **VPC and more** option, which provisions the VPC along with subnets, route tables, and other networking resources in one step. I set the Name tag auto-generation to `New-CBA`, kept the default `10.0.0.0/16` CIDR block, and selected **2** Availability Zones. This automatically previewed 4 subnets (2 public, 2 private) across `eu-west-2a` and `eu-west-2b`, along with 3 route tables. For NAT gateways, I selected **Zonal** rather than the default, before clicking **Create VPC**.

![Create VPC settings — New-CBA, 2 AZs, VPC and more](screenshots/01-create-vpc-settings.png)

The Create VPC workflow completed successfully, provisioning the VPC, subnets, internet gateway, route tables, a NAT gateway, and an S3 endpoint all in one pass.

![Create VPC workflow — all steps succeeded, resource IDs listed](screenshots/02-create-vpc-workflow-success.png)

The VPC details page confirmed `New-CBA-vpc` was available with its 4 subnets, 4 route tables, and 3 network connections (internet gateway, NAT gateway, and an S3 VPC endpoint).

![New-CBA-vpc details and resource map](screenshots/03-vpc-details-resource-map.png)

## Step 2: Enable auto-assign public IP on the public subnets

By default, subnets don't auto-assign public IPv4 addresses to launched instances. Since the ASG instances need to be reachable (indirectly, via the load balancer) I enabled this on both public subnets. From **Subnets**, I selected the first public subnet, then **Actions → Edit subnet settings**.

![Subnets list — Actions menu, Edit subnet settings](screenshots/04-subnets-edit-settings-menu.png)

I checked **Enable auto-assign public IPv4 address** and saved.

![Edit subnet settings — auto-assign public IPv4 enabled](screenshots/05-edit-subnet-settings-enable-public-ip.png)

The subnets list confirmed the change. I repeated the same process for the second public subnet.

![Subnets list — settings successfully changed](screenshots/06-subnets-settings-changed-confirmed.png)

## Step 3: Provision an Application Load Balancer

With the network in place, I moved to EC2 to create the load balancer that would distribute traffic across instances in both Availability Zones.

![EC2 dashboard](screenshots/07-ec2-dashboard.png)

From **Load Balancers**, I clicked **Create load balancer**, starting from an empty list.

![Load balancers — empty list before creation](screenshots/08-load-balancers-empty-list.png)

I compared the load balancer types and selected **Application Load Balancer**, since the workload is standard HTTP/HTTPS web traffic.

![Compare and select load balancer type — Application Load Balancer chosen](screenshots/09-compare-select-lb-type.png)

For the basic configuration, I named it `new-CBA-LB`, with scheme **Internet-facing** and IP address type **IPv4**.

![Basic configuration — new-CBA-LB, Internet-facing, IPv4](screenshots/10-lb-basic-configuration.png)

For network mapping, I selected the `New-CBA-vpc` VPC and mapped both Availability Zones to their respective **public** subnets, since the load balancer itself needs to be internet-reachable in both zones.

![Network mapping — VPC and public subnets for both AZs](screenshots/11-lb-network-mapping.png)

For the security group, rather than use an existing one, I created a new one specifically for the load balancer.

![Security groups section — before creating a new group](screenshots/12-lb-security-groups-section.png)

In the security group creation form (which opens in a new browser tab), I named it `new-CBA-LB`, described it as enabling web access to the load balancer, and selected the `New-CBA-vpc` VPC.

![Create security group — new-CBA-LB, description, VPC](screenshots/13-create-security-group-form.png)

I added inbound rules for **HTTP** and **HTTPS**, both open to `0.0.0.0/0`, then created the security group.

![Inbound rules — HTTP and HTTPS open to 0.0.0.0/0](screenshots/14-security-group-inbound-rules.png)

Back in the load balancer configuration tab, I refreshed the security groups dropdown and selected the newly created `new-CBA-LB` group.

![Load balancer security group — new-CBA-LB selected after refresh](screenshots/15-lb-security-group-selected.png)

For listeners and routing, under the `HTTP:80` listener's default action, I chose **Forward to target groups** and clicked **create target group**, which opens in another new tab.

![Listeners and routing — HTTP:80, create target group](screenshots/16-lb-listener-create-target-group.png)

In the target group setup, I set the target type to **Instances**, named it `new-CBA-LB`, and chose protocol version **HTTP1**.

![Target group — protocol version HTTP1](screenshots/17-target-group-protocol-version.png)

For health checks, I set the health check path to `/index.html`.

![Target group — health check path /index.html](screenshots/18-target-group-health-check-path.png)

Under advanced health check settings, I configured a healthy threshold of 2, unhealthy threshold of 2, timeout of 5 seconds, and an interval of 10 seconds.

![Advanced health check settings — thresholds 2/2, timeout 5s, interval 10s](screenshots/19-target-group-advanced-health-check.png)

At the register targets step, no instances existed yet, so I proceeded with 0 targets registered — targets would be added automatically once the Auto Scaling Group was created.

![Register targets — no instances available yet](screenshots/20-target-group-register-targets.png)

After reviewing the target group configuration, I created it.

![Target group review and create](screenshots/21-target-group-review-create.png)

Back in the load balancer tab, I refreshed and selected the newly created `new-CBA-LB` target group for the listener.

![Load balancer listener — new-CBA-LB target group selected](screenshots/22-lb-target-group-selected.png)

I reviewed the full load balancer configuration summary before creating it. One thing worth flagging honestly: at this final review stage, the network mapping listed `eu-west-2b` against `New-CBA-subnet-private2-eu-west-2b` rather than the public subnet I'd selected during the network mapping step — likely because the AZ/subnet selection reset at some point while I was switching between browser tabs to create the security group and target group. The load balancer still created successfully and traffic worked end-to-end (see Step 8), but it's a good reminder to re-check the review screen line by line before finalizing, rather than assuming earlier selections held.

![Load balancer review summary — eu-west-2b shows the private subnet, not the public one selected earlier](screenshots/23-lb-review-summary.png)

The load balancer was created successfully.

![new-CBA-LB successfully created](screenshots/24-lb-created-successfully.png)

## Step 4: Create a Launch Template

Next, I defined the instance configuration that the Auto Scaling Group would use to launch new EC2 instances. From **Launch Templates**, I clicked **Create launch template**.

![Navigating to Launch Templates](screenshots/25-navigate-launch-templates.png)

I named it `new-CBA-template`, described it as version 1, and checked **Provided guidance to help me set up a template that I can use with EC2 Auto Scaling**.

![Launch template name and description](screenshots/26-launch-template-name-description.png)

For the AMI, I used Quick Start and selected **Amazon Linux 2023 kernel-6.12** (free tier eligible). For instance type, I chose `t3.micro`.

![Instance type — t3.micro, free tier eligible](screenshots/27-launch-template-instance-type.png)

For the key pair, I selected an existing key pair, `cba-keypair`.

![Key pair — cba-keypair selected](screenshots/28-launch-template-key-pair.png)

For network settings, I left the subnet as **Do not include in launch template** (since the Auto Scaling Group configures subnets separately), and created a new security group, `new-CBA-LT`, in the `New-CBA-vpc` VPC.

![Network settings — subnet excluded, new-CBA-LT security group](screenshots/29-launch-template-network-settings.png)

For the inbound rule on this security group, I set the type to **HTTP** with the source set to the load balancer's security group (`new-CBA-LB`) — meaning only traffic coming through the load balancer is allowed to reach the instances directly.

![Inbound security group rule — HTTP from new-CBA-LB security group](screenshots/30-launch-template-security-group-rule.png)

Under **Advanced details**, I set the IAM instance profile to **Don't include in Launch Template**, and pasted a user data script that installs and starts a web server, writing a simple HTML page identifying the instance, then setting correct file permissions and restarting Apache.

![User data script — installs web server, writes index.html, restarts Apache](screenshots/31-launch-template-user-data.png)

## Step 5: Create an Auto Scaling Group

With the launch template ready, I navigated to **Auto Scaling Groups** and clicked **Create Auto Scaling group**.

![Navigating to Auto Scaling Groups](screenshots/32-navigate-auto-scaling-groups.png)

I named it `new-CBA-ASG` and selected `new-CBA-template` as the launch template.

![ASG name and launch template — new-CBA-ASG, new-CBA-template](screenshots/33-asg-name-and-launch-template.png)

For network settings, I selected the `New-CBA-vpc` VPC and both **public** subnets (matching the load balancer's subnets), keeping the default **Balanced best effort** Availability Zone distribution strategy.

![ASG network — VPC and both public subnets](screenshots/34-asg-network-vpc-subnets.png)

For load balancing, I selected **Attach to an existing load balancer**, then chose the `new-CBA-LB | HTTP` target group created earlier.

![ASG load balancing — attach to existing target group new-CBA-LB](screenshots/35-asg-load-balancing-attach.png)

I turned on **Elastic Load Balancing health checks** so the ASG would react to the load balancer's own health check results (not just basic EC2 status checks), and set the health check grace period to 200 seconds to give new instances time to finish initializing before being health-checked.

![Health checks — Elastic Load Balancing health checks on, 200s grace period](screenshots/36-asg-health-checks.png)

For group size and scaling, I set desired capacity to 2, with min and max desired capacity both set to 2 — keeping the group at a fixed size of 2 instances rather than scaling based on demand.

![Group size and scaling — desired/min/max capacity all set to 2](screenshots/37-asg-group-size-scaling.png)

Under additional settings, I enabled **group metrics collection within CloudWatch** for monitoring.

![Additional settings — CloudWatch group metrics enabled](screenshots/38-asg-additional-settings.png)

I added a tag, `Name: new-CBA-app`, so instances launched by the group would be identifiable.

![Add tags — Name: new-CBA-app](screenshots/39-asg-add-tags.png)

After reviewing, I created the Auto Scaling group.

![Auto Scaling group creation confirmed](screenshots/40-asg-create-confirmation.png)

## Step 6: Verify the ASG launched instances

The Auto Scaling Groups list showed `new-CBA-ASG` updating its capacity toward the desired count of 2.

![Auto Scaling groups list — new-CBA-ASG updating capacity](screenshots/41-asg-list-updating-capacity.png)

The Activity tab confirmed two EC2 instances were successfully launched to bring capacity from 0 to 2.

![ASG Activity history — two instances launched successfully](screenshots/42-asg-activity-history.png)

The Instance management tab showed both instances as **Healthy**, each running the launch template in a different Availability Zone.

![ASG Instance management — both instances Healthy, different AZs](screenshots/43-asg-instance-management-healthy.png)

In the EC2 console, both instances showed as **Running** with all status checks passed.

![EC2 Instances — both new-CBA-app instances Running](screenshots/44-ec2-instances-running.png)

## Step 7: Test high availability

To confirm the Auto Scaling Group would actually self-heal, I manually terminated one of the running instances.

![Terminate instance action](screenshots/45-ec2-terminate-instance-action.png)

The termination was confirmed, and the instance moved to a **Terminated** state.

![Instance terminated confirmed](screenshots/46-ec2-instance-terminated-confirmed.png)

Within a couple of minutes, the ASG detected the drop below its desired capacity and launched a replacement instance automatically. Over the course of testing, I terminated and stopped several more instances to confirm the pattern held consistently — each time, the group briefly showed a mix of terminating/unhealthy and newly launching instances before settling back at 2 healthy, in-service instances.

![Instances mid-transition — 2 terminating/unhealthy alongside 2 healthy in-service instances](screenshots/47-ec2-instances-4-terminating.png)

The ASG's capacity overview confirmed it remained configured for a desired capacity of 2 (min 2, max 2) throughout, and had settled back to exactly 2 healthy instances — the extra instances seen mid-test were always transient, on their way to being terminated once a healthy replacement was in service.

![ASG capacity overview — desired capacity 2, scaling limits 2-2, settled at 2 healthy instances](screenshots/48-asg-capacity-overview.png)

I also tested stopping multiple instances at once to see how the group responded under a bigger disruption.

![Multiple instances stopped simultaneously for a harder test](screenshots/49-ec2-instances-stopping-multiple.png)

## Step 8: Test via the load balancer's DNS name

The real test of high availability is whether the application stays reachable through the load balancer regardless of what's happening to any individual instance. I navigated to **Load Balancers** by way of the **Target Groups** section, to first confirm the target group's registered targets were healthy.

![Navigating via the Load Balancing sidebar](screenshots/50-navigate-load-balancing-sidebar.png)

![Target groups list — new-CBA-LB target group](screenshots/51-target-group-list.png)

The target group showed both registered instances as **Healthy**.

![Target group registered targets — both Healthy](screenshots/52-target-group-registered-targets-healthy.png)

The Load Balancers list confirmed `new-CBA-LB` as **Active**.

![Load balancers list — new-CBA-LB Active](screenshots/53-load-balancers-list-active.png)

I opened the load balancer's details, copied its DNS name, and noted its ARN.

![Load balancer details — ARN and DNS name](screenshots/54-lb-details-arn-dns.png)

Pasting the DNS name into a browser opened the demo web page served by whichever instance the load balancer happened to route the request to — confirming the full path from load balancer → target group → healthy EC2 instance was working end-to-end. The page displayed the responding instance's private IP, and refreshing repeatedly showed the IP changing as the load balancer distributed requests across the two instances.

![HA demo web page served via the ALB's DNS name](screenshots/55-ha-demo-webpage-via-alb-dns.png)

## Step 9: Clean up

With the lab complete, I cleaned up the deployed resources, starting with the Auto Scaling Group.

![Deleting the Auto Scaling group](screenshots/56-asg-delete-action.png)

Deleting the ASG terminated all of its managed instances.

![All instances terminated after the ASG was deleted](screenshots/57-instances-terminated-after-asg-delete.png)

I then continued cleaning up the remaining resources (load balancer, target group, launch template, and VPC) in the same session.

## Key concepts

| Concept | Description |
|---|---|
| VPC and more | The "VPC and more" wizard provisions a VPC along with subnets, route tables, an internet gateway, and NAT gateway(s) in one step, instead of creating each piece manually. |
| Public vs. private subnets | A subnet is only "public" in practice if its route table sends `0.0.0.0/0` traffic to an internet gateway, and instances in it have public IPs — auto-assign public IPv4 must be explicitly enabled per subnet. |
| Application Load Balancer (ALB) | Distributes incoming HTTP/HTTPS traffic across healthy targets in a target group, performing its own health checks independent of EC2 status checks. |
| Target groups and health checks | A target group defines how the load balancer checks target health (protocol, path, thresholds, interval) — targets that fail enough consecutive checks are marked unhealthy and stop receiving traffic. |
| Launch templates | Define everything about how a new EC2 instance should be configured (AMI, instance type, key pair, security groups, user data) so an Auto Scaling Group can launch consistent instances on demand. |
| Auto Scaling Group (ASG) | Maintains a desired number of healthy instances across one or more Availability Zones, automatically replacing any that are terminated or fail health checks. |
| ELB health checks vs. EC2 health checks | By default, an ASG only reacts to EC2-level health checks (is the instance running); turning on Elastic Load Balancing health checks lets the ASG also react to the load balancer's own application-level health checks. |
| Health check grace period | Delays the first health check on a newly launched instance so it isn't marked unhealthy while it's still booting and initializing (e.g. installing and starting a web server via user data). |
| Self-healing via desired capacity | Setting min = max = desired capacity keeps a fixed-size fleet; if an instance is terminated, the ASG launches a replacement to bring the count back to the desired capacity, without any scaling policy needed. |
| Testing HA via the load balancer DNS, not the instance IP | Connecting directly to an instance's IP bypasses the load balancer entirely; the real test of high availability is that the load balancer's DNS name keeps working even as individual instances are terminated behind it. |

## What I learned

This lab made the difference between "an EC2 instance is running" and "the application is highly available" very concrete. Provisioning the VPC through the guided wizard was straightforward, but understanding *why* the load balancer's subnets had to be public (while the Auto Scaling Group's instances could sit behind it without needing public IPs of their own to be reachable) required actually reasoning through the traffic path: user → ALB (public subnet) → target group → EC2 instance (registered by the ASG).

The most instructive part was deliberately breaking things to prove the setup worked. Terminating an instance and watching the ASG detect the capacity shortfall and launch a replacement within a couple of minutes was a good demonstration of self-healing infrastructure that requires no manual intervention. Testing this repeatedly — including stopping multiple instances at once — showed the desired/min/max capacity of 2 acting as a firm floor: no matter how many instances were removed, the group always worked to bring healthy capacity back to exactly 2, with the extra terminating instances just being transient noise during the transition.

Turning on Elastic Load Balancing health checks (rather than relying on default EC2 health checks alone) was an important detail — without it, the ASG would only notice an instance was dead, not that it was running but failing to actually serve traffic correctly. Finally, testing via the load balancer's DNS name rather than any individual instance's IP address was the right way to validate the whole point of the exercise: the application stayed reachable and the responding instance's IP changed between refreshes, confirming traffic really was being distributed and failed-over automatically rather than pinned to one instance. I also learned to double-check the review screen at the end of a multi-tab console workflow — switching tabs to create a security group and target group mid-flow caused one AZ's subnet selection to silently revert to a private subnet, which I only caught by reading the final review screenshot closely.
