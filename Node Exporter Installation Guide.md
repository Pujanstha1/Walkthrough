# Installing Node Exporter on AWS EC2

## Table of Contents

- [What is Prometheus?](#what-is-prometheus)
- [What is Node Exporter?](#what-is-node-exporter)
- [When to Use Exporters](#when-to-use-exporters)
- [Prerequisites](#prerequisites)
- [Step 1: Create EC2 Instance with CloudFormation](#step-1-create-ec2-instance-with-cloudformation)
- [Step 2: Install Node Exporter](#step-2-install-node-exporter)
- [Step 3: Verify Installation](#step-3-verify-installation)
- [Troubleshooting](#troubleshooting)

---

## What is Prometheus?

**Prometheus** is an open-source monitoring and alerting toolkit widely used in DevOps and software engineering. It specializes in collecting and storing time-series data (metrics) from various systems and applications.

### Key Features of Prometheus

**Time-Series Database**: Stores metrics with timestamps, allowing you to track how values change over time

**Pull-Based Model**: Prometheus actively scrapes (pulls) metrics from configured targets at regular intervals

**PromQL**: Powerful query language for analyzing metrics and creating custom dashboards

**Alerting**: Built-in alertmanager to send notifications when metrics cross thresholds

**Service Discovery**: Automatically discovers targets in dynamic environments (Kubernetes, AWS, etc.)

### Use Cases in DevOps

**Infrastructure Monitoring**: CPU usage, memory, disk I/O, network traffic on servers

**Application Performance**: Response times, error rates, request counts

**Container Orchestration**: Kubernetes cluster health, pod resource usage

**Alerting & SLA Tracking**: Proactive notifications when systems degrade

**Capacity Planning**: Historical data analysis to predict future resource needs

### Why Prometheus?

Prometheus has become the de facto standard for cloud-native monitoring because it:

- Integrates seamlessly with Kubernetes and containerized environments
- Provides powerful visualization through Grafana dashboards
- Offers multi-dimensional data model (labels for flexible querying)
- Has a vibrant ecosystem with hundreds of exporters for different systems
- Is reliable, scalable, and battle-tested in production environments

---

## What is Node Exporter?

**Node Exporter** is a Prometheus agent (also called an "exporter") that exposes hardware and OS-level metrics from Linux/Unix systems. It runs as a lightweight daemon on your servers and makes system metrics available for Prometheus to scrape.

### Metrics Exposed by Node Exporter

Node Exporter provides detailed system metrics including:

**CPU Metrics**: Usage per core, idle time, system vs user time, CPU frequency

**Memory Metrics**: Total RAM, available memory, swap usage, buffers/cache

**Disk Metrics**: I/O operations, read/write bytes, disk utilization, filesystem usage

**Network Metrics**: Bytes sent/received, packet counts, errors, interface statistics

**System Load**: 1/5/15 minute load averages

**Uptime**: System boot time and uptime duration

**Process Metrics**: Number of processes, context switches, forks

### Default Port

Node Exporter listens on **port 9100** by default and exposes metrics at the `/metrics` endpoint.

---

## When to Use Exporters

Exporters are specialized agents that translate metrics from various systems into Prometheus-compatible format. Different exporters serve different purposes:

### Common Exporters and Use Cases

| Exporter | Purpose | When to Use |
|----------|---------|-------------|
| **Node Exporter** | System/hardware metrics | Monitor Linux/Unix server health (CPU, memory, disk, network) |
| **Blackbox Exporter** | Endpoint probing | Monitor HTTP/HTTPS endpoints, DNS, TCP/ICMP availability |
| **MySQL Exporter** | Database metrics | Track MySQL performance (queries, connections, replication lag) |
| **PostgreSQL Exporter** | Database metrics | Monitor PostgreSQL performance and health |
| **Redis Exporter** | Cache metrics | Track Redis memory usage, hit rates, key counts |
| **Nginx Exporter** | Web server metrics | Monitor Nginx connections, requests, response codes |
| **Apache Exporter** | Web server metrics | Track Apache performance and resource usage |
| **HAProxy Exporter** | Load balancer metrics | Monitor HAProxy backend health, request rates |
| **cAdvisor** | Container metrics | Track Docker container resource usage |
| **kube-state-metrics** | Kubernetes objects | Monitor K8s deployments, pods, nodes, services |
| **CloudWatch Exporter** | AWS metrics | Scrape AWS CloudWatch metrics into Prometheus |
| **JMX Exporter** | Java applications | Monitor JVM metrics (heap, threads, garbage collection) |

### Choosing the Right Exporter

**Infrastructure monitoring**: Start with Node Exporter on all servers

**Application-specific**: Use dedicated exporters (MySQL, Redis, etc.) for each service

**Black-box monitoring**: Use Blackbox Exporter to test endpoints from outside

**Container environments**: Combine cAdvisor + kube-state-metrics + Node Exporter

**Custom applications**: Write custom exporters or use client libraries (Python, Go, Java)

### Multiple Exporters on One Host

You can run multiple exporters on the same server. For example, a typical production server might run:

- Node Exporter (port 9100) - system metrics
- MySQL Exporter (port 9104) - database metrics
- Application-specific exporter (custom port) - business metrics

Each exporter runs independently and exposes metrics on its own port.

---

## Prerequisites

Before starting, ensure you have:

- AWS account with permissions to create EC2 instances and CloudFormation stacks
- An existing EC2 Key Pair for SSH access
- Basic understanding of Linux command line
- AWS CLI installed (optional, for CLI deployment)

---

## Step 1: Create EC2 Instance with CloudFormation

We'll use a CloudFormation template to create an EC2 instance with the necessary security group rules.

### CloudFormation Template

Create a file named `ec2-node-exporter.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 instance for Node Exporter monitoring

Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t3.micro
      - t3.small
    Description: EC2 instance type
  
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: SSH KeyPair name
  
  ImageName:
    Description: Select the Operating System
    Type: String
    Default: ubuntu-22.04
    AllowedValues:
      - ubuntu-22.04
      - amazon-linux-2
      - amazon-linux-2023

Mappings:
  AmiSSMMap:
    ubuntu-22.04:
      SSM: /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id
    amazon-linux-2:
      SSM: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
    amazon-linux-2023:
      SSM: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  # VPC Configuration
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: NodeExporter-VPC

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: NodeExporter-IGW

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - Key: Name
          Value: NodeExporter-PublicSubnet

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: NodeExporter-RouteTable

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

  # Security Group - SSH and Node Exporter
  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and Node Exporter access
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
          Description: SSH access
        - IpProtocol: tcp
          FromPort: 9100
          ToPort: 9100
          CidrIp: 0.0.0.0/0
          Description: Node Exporter metrics
      Tags:
        - Key: Name
          Value: NodeExporter-SG

  # EC2 Instance
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      ImageId: !Sub
        - "{{resolve:ssm:${SSMPath}}}"
        - SSMPath: !FindInMap
            - AmiSSMMap
            - !Ref ImageName
            - SSM
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref EC2SecurityGroup
      Tags:
        - Key: Name
          Value: NodeExporter-Instance

Outputs:
  InstancePublicIP:
    Description: Public IP of EC2 instance
    Value: !GetAtt EC2Instance.PublicIp
  
  InstanceId:
    Description: Instance ID
    Value: !Ref EC2Instance
  
  NodeExporterURL:
    Description: Node Exporter metrics endpoint
    Value: !Sub "http://${EC2Instance.PublicIp}:9100/metrics"
  
  SSHCommand:
    Description: SSH command to connect to instance
    Value: !Sub "ssh -i your-key.pem ubuntu@${EC2Instance.PublicIp}"
```

### Deploy the Stack

**Using AWS Console:**

1. Go to CloudFormation in AWS Console
2. Click **Create Stack** → **With new resources**
3. Upload the `ec2-node-exporter.yaml` template
4. Enter stack name: `node-exporter-stack`
5. Fill parameters (KeyName, InstanceType, ImageName)
6. Click **Next** → **Next** → **Create Stack**

**Using AWS CLI:**

```bash
aws cloudformation create-stack \
  --stack-name node-exporter-stack \
  --template-body file://ec2-node-exporter.yaml \
  --parameters \
    ParameterKey=KeyName,ParameterValue=your-key-name \
    ParameterKey=InstanceType,ParameterValue=t2.micro \
    ParameterKey=ImageName,ParameterValue=ubuntu-22.04
```

### Get the Instance IP

Once the stack is created, retrieve the public IP:

```bash
aws cloudformation describe-stacks \
  --stack-name node-exporter-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`InstancePublicIP`].OutputValue' \
  --output text
```

---

## Step 2: Install Node Exporter

SSH into your EC2 instance and follow these steps:

### 1. Connect to Your Instance

```bash
ssh -i /path/to/your-key.pem ubuntu@<INSTANCE_PUBLIC_IP>
```

For Amazon Linux, use `ec2-user` instead of `ubuntu`.

### 2. Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

For Amazon Linux:

```bash
sudo yum update -y
```

### 3. Download Node Exporter

Navigate to `/tmp` and download Node Exporter version 1.10.2:

```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
```

### 4. Extract the Archive

```bash
tar xvf node_exporter-1.10.2.linux-amd64.tar.gz
```

### 5. Move Binary to System Path

```bash
sudo mv node_exporter-1.10.2.linux-amd64/node_exporter /usr/local/bin/
```

Verify the binary is executable:

```bash
which node_exporter
# Should output: /usr/local/bin/node_exporter

node_exporter --version
# Should output: node_exporter, version 1.10.2 (...)
```

### 6. Create a Dedicated User

Create a system user to run Node Exporter (for security):

```bash
sudo useradd -rs /bin/false node_exporter
```

This creates a user with:
- `-r`: system user (no home directory by default)
- `-s /bin/false`: no shell access (cannot login)

### 7. Create Systemd Service

Create the systemd service file:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Paste the following configuration:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

**Service Configuration Explained:**

- `Wants=network-online.target`: Ensures network is ready before starting
- `After=network-online.target`: Starts after network is online
- `User=node_exporter`: Runs as the unprivileged node_exporter user
- `Type=simple`: Standard service type for long-running processes
- `ExecStart`: Command to start Node Exporter
- `WantedBy=multi-user.target`: Enables service at system boot

Save and exit (Ctrl+O, Enter, Ctrl+X in nano).

### 8. Enable and Start the Service

Reload systemd to recognize the new service:

```bash
sudo systemctl daemon-reload
```

Start Node Exporter:

```bash
sudo systemctl start node_exporter
```

Enable it to start automatically on boot:

```bash
sudo systemctl enable node_exporter
```

### 9. Check Service Status

```bash
sudo systemctl status node_exporter
```

You should see output like:

```
● node_exporter.service - Node Exporter
     Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2026-01-29 10:30:45 UTC; 5s ago
   Main PID: 1234 (node_exporter)
      Tasks: 4 (limit: 1234)
     Memory: 12.0M
        CPU: 100ms
     CGroup: /system.slice/node_exporter.service
             └─1234 /usr/local/bin/node_exporter
```

If the status shows "active (running)", Node Exporter is successfully running.

---

## Step 3: Verify Installation

### Test Locally on the Instance

From within the EC2 instance, test the metrics endpoint:

```bash
curl http://localhost:9100/metrics
```

You should see a large output of metrics like:

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_cpu_seconds_total{cpu="0",mode="system"} 123.45
...
```

### Test from Your Local Machine

From your local computer, test the public endpoint:

```bash
curl http://<INSTANCE_PUBLIC_IP>:9100/metrics
```

Or open in a browser:

```
http://<INSTANCE_PUBLIC_IP>:9100/metrics
```

### View Node Exporter Web UI

Node Exporter also provides a simple web interface:

```
http://<INSTANCE_PUBLIC_IP>:9100/
```

This page shows links to:
- `/metrics` - Prometheus metrics
- Configuration and build info

---

## Troubleshooting

### Service Won't Start

Check the service status for errors:

```bash
sudo systemctl status node_exporter
sudo journalctl -u node_exporter -n 50
```

Common issues:
- Binary not found: Verify `/usr/local/bin/node_exporter` exists
- Permission issues: Ensure binary is executable (`sudo chmod +x /usr/local/bin/node_exporter`)
- Port already in use: Check if another process is using port 9100 (`sudo lsof -i :9100`)

### Can't Access Metrics from Browser

1. Verify security group allows port 9100:

```bash
aws ec2 describe-security-groups \
  --group-ids <SECURITY_GROUP_ID> \
  --query 'SecurityGroups[0].IpPermissions'
```

2. Check if Node Exporter is listening:

```bash
sudo netstat -tlnp | grep 9100
```

3. Test locally first:

```bash
curl http://localhost:9100/metrics
```

If localhost works but public IP doesn't, it's a firewall/security group issue.

### Permission Denied Errors

If you see permission errors in the logs, check file ownership:

```bash
ls -l /usr/local/bin/node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### High Memory Usage

Node Exporter is lightweight and typically uses 10-20MB of RAM. If usage is high:

```bash
ps aux | grep node_exporter
```

Restart the service:

```bash
sudo systemctl restart node_exporter
```

---

## Next Steps

### Configure Prometheus to Scrape This Instance

On your Prometheus server, add this target to `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<INSTANCE_PUBLIC_IP>:9100']
        labels:
          instance: 'my-ec2-server'
          environment: 'production'
```

Restart Prometheus:

```bash
sudo systemctl restart prometheus
```

### Create Grafana Dashboard

Import a pre-built Node Exporter dashboard in Grafana:

1. Go to Grafana → Dashboards → Import
2. Enter dashboard ID: **1860** (Node Exporter Full)
3. Select your Prometheus data source
4. Click Import

### Set Up Alerting

Create alerts in Prometheus for critical metrics:

```yaml
groups:
  - name: node_exporter
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
```

---

## Security Considerations

**Restrict Port 9100 Access**: The CloudFormation template opens port 9100 to the world (0.0.0.0/0). For production:

- Restrict to your Prometheus server IP only
- Use VPC peering or VPN for private network access
- Consider using AWS Systems Manager Session Manager instead of SSH

**Update Security Group:**

```bash
# Remove public access
aws ec2 revoke-security-group-ingress \
  --group-id <SECURITY_GROUP_ID> \
  --protocol tcp \
  --port 9100 \
  --cidr 0.0.0.0/0

# Add specific Prometheus server IP
aws ec2 authorize-security-group-ingress \
  --group-id <SECURITY_GROUP_ID> \
  --protocol tcp \
  --port 9100 \
  --cidr <PROMETHEUS_SERVER_IP>/32
```

**Run Behind a Reverse Proxy**: Use Nginx with TLS and authentication for production.

---

## Cleanup

To delete all resources:

```bash
aws cloudformation delete-stack --stack-name node-exporter-stack
```

To stop Node Exporter without deleting the instance:

```bash
sudo systemctl stop node_exporter
sudo systemctl disable node_exporter
```

---

## Additional Resources

- [Prometheus Official Documentation](https://prometheus.io/docs/)
- [Node Exporter GitHub](https://github.com/prometheus/node_exporter)
- [Prometheus Exporters List](https://prometheus.io/docs/instrumenting/exporters/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)

---

**Last Updated**: January 2026
