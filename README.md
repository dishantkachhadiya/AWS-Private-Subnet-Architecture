# AWS Production-Grade VPC Project (Public/Private Subnet Architecture)

I built a production-style AWS architecture: a VPC with public and private subnets across two Availability Zones, a load-balanced application running in the private subnet with no public IP, and a Bastion host as the only way in for SSH access.

Reference: AWS public/private subnet architecture pattern.

## Architecture

```
Internet
   │
Internet Gateway
   │
┌──────────────── VPC ────────────────┐
│  Public Subnet (AZ-1)   Public Subnet (AZ-2)
│   - NAT Gateway           - (route to IGW)
│   - Bastion Host
│   - Application Load Balancer (spans both public subnets)
│
│  Private Subnet (AZ-1)  Private Subnet (AZ-2)
│   - EC2 instance (app)   - EC2 instance (app)
│   - no public IP         - no public IP
└───────────────────────────────────────┘
```

- **2 Availability Zones**, for redundancy
- **Public subnets**: NAT Gateway, Bastion host, Load Balancer
- **Private subnets**: application EC2 instances, no public IP addresses
- **Auto Scaling Group**: manages the two application instances
- **Target Group + ALB**: distributes incoming traffic across both instances and performs health checks

## What I used

- **Provider:** AWS (VPC, EC2, ALB, Auto Scaling, NAT Gateway)
- **AMI:** Ubuntu
- **Instance type:** t3.micro
- **App:** simple Python `http.server` on port 8000, serving a static HTML page

## Steps

### 1. Create the VPC

Used the AWS VPC wizard ("VPC and more") to create:
- 1 VPC
- 2 public subnets (one per AZ)
- 2 private subnets (one per AZ)
- 1 Internet Gateway, attached to the VPC
- 1 NAT Gateway (in a public subnet)
- Route tables: public subnets routed to the Internet Gateway; private subnets routed through the NAT Gateway
- No VPC endpoint (not needed for this project)

### 2. Create a Launch Template

Defined:
- AMI: Ubuntu
- Instance type: t3.micro
- Key pair for SSH
- Security group allowing:
  - Port 22 (SSH) from anywhere
  - Port 8000 (app) from anywhere

### 3. Create an Auto Scaling Group

- Used the Launch Template above
- Placed instances in the **two private subnets**
- Desired / min / max capacity: 2

This created two EC2 instances, one per AZ, with no public IP addresses.

### 4. Create a Bastion host

- Launched a separate EC2 instance (Ubuntu, t3.micro) in a **public subnet**
- Enabled a public IP
- Security group allowing SSH (port 22) from anywhere
- This is the only entry point for SSH into the private subnet instances

### 5. Access the private instances through the Bastion

Copied the SSH key onto the Bastion host, then SSH'd from the Bastion into each private instance using its private IP:

```bash
# From my laptop, copy the key to the Bastion
scp -i ~/.ssh/aws-prod-demo.pem ~/.ssh/aws-prod-demo.pem ubuntu@<bastion-public-ip>:/home/ubuntu

# SSH into the Bastion
ssh -i ~/.ssh/aws-prod-demo.pem ubuntu@<bastion-public-ip>

# From the Bastion, SSH into a private instance
ssh -i aws-prod-demo.pem ubuntu@<private-instance-ip>
```

### 6. Deploy a simple app on each private instance

On the **first** instance:

```bash
python3 -m http.server 8000
```
Serving a page titled *"My First AWS PROJECT to demonstrate apps in Private subnet."*

On the **second** instance, a different page:
```bash
python3 -m http.server 8000
```
Serving a page titled *"This is my second AWS Project."*

Using two different pages made it possible to visually confirm the load balancer is distributing traffic across both instances rather than always hitting the same one.

### 7. Create a Target Group

- Target type: Instance
- Protocol: HTTP, Port: 8000
- VPC: the one created above
- Registered both private EC2 instances
- Health check: HTTP

### 8. Create the Application Load Balancer

- Internet-facing
- Spans both public subnets (one per AZ)
- Listener on **port 80**, forwarding to the Target Group above
- Security group allowing inbound HTTP (port 80) from anywhere

Load balancer DNS name:
```
aws-prod-demo-2026018832.ap-south-1.elb.amazonaws.com
```

### 9. Test the result

```
http://aws-prod-demo-2026018832.ap-south-1.elb.amazonaws.com
```

Confirmed the app loads successfully through the load balancer, with both backend instances showing as **Healthy** in the Target Group.

### 10. Verify load balancing across both instances

A single browser tab tends to reuse the same connection, which can make it look like only one instance is ever responding. To properly verify traffic distribution, used `curl` in a loop, which opens a fresh connection each time:

```bash
for i in {1..10}; do curl -s aws-prod-demo-2026018832.ap-south-1.elb.amazonaws.com; echo; done
```

The output alternated between the two pages, confirming the Application Load Balancer and Target Group were correctly distributing requests across both private-subnet instances.

Also confirmed visually by opening two separate browser tabs to the same URL: one showed *"My First AWS PROJECT"* and the other showed *"This is my second AWS Project."*

## Problems I ran into

| Problem | Cause | Fix |
|---------|-------|-----|
| `Permission denied (publickey)` when copying the key with `scp` | `.pem` file was stored on a Windows-mounted drive (`/mnt/d/...`) in WSL, which always reports file permissions as `777` regardless of `chmod` | Copied the key into WSL's native filesystem (`~/.ssh/`) and ran `chmod 600` there, where permissions are respected |
| Load balancer returned `ERR_CONNECTION_REFUSED` on `http://<lb-dns>` | The ALB only had a listener on port 8000, not port 80 (the default port browsers use) | Added a second listener on port 80, forwarding to the same Target Group |
| Phone and incognito browser failed to load the site at all | Typing just the domain caused the browser to auto-upgrade the request to `https://`, but no HTTPS (443) listener was configured | Explicitly typed `http://` before the URL |
| Browser kept showing only the first instance's page | Browsers reuse the same TCP connection across reloads, so the ALB kept routing to the same backend instance | Verified true load balancing using a `curl` loop (new connection each request) and separate browser tabs |

## Notes

- The private instances have no public IP addresses and cannot be reached directly from the internet, only through the Bastion host (for SSH) or the Load Balancer (for HTTP).
- The NAT Gateway allows the private instances to make outbound internet requests (e.g. for package installs) while hiding their real IP.
- **Cost warning:** this project uses billable resources even when idle, the NAT Gateway and Application Load Balancer both incur hourly charges. Terminate the EC2 instances, delete the Load Balancer, delete the NAT Gateway, and release any unused Elastic IPs once finished.
