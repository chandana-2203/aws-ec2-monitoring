# COMPLETE SETUP GUIDE  
### Grafana + CloudWatch Agent Monitoring on AWS EC2  
**Steps 1 → 16 with uniform formatting**

This guide explains how to install Grafana, configure CloudWatch Agent, set up dashboards, alarms, and monitor EC2 instance metrics.

---

## **1. Launch EC2 Instance**

- Go to AWS Console → EC2 → Launch Instance  
- Name: `grafana-server`  
- AMI: **Amazon Linux 2**  
- Instance Type: `t2.micro`  
- Key Pair: Create/Select `.pem`  
- Security Group:
  - SSH → 22
  - Custom TCP → 3000  
- Launch instance  
- Note Public IPv4 + Instance ID  

---

## **2. Create & Attach IAM Role**

- IAM → Roles → Create role  
- Use case: EC2  
- Policies:
  - CloudWatchAgentServerPolicy  
  - AmazonSSMManagedInstanceCore  
  - (Optional) CloudWatchFullAccess  
- Name: `grafana-role`  
- Attach to instance via:  
  EC2 → Instance → Actions → Security → Modify IAM role  

---

## **3. Connect to EC2**

```bash
ssh -i /path/to/key.pem ec2-user@<EC2_PUBLIC_IP>
```

Switch to root:

```bash
sudo -i
```

Update system:

```bash
sudo yum update -y
```

---

## **4. Install Grafana**

Install Grafana:

```bash
sudo yum install -y https://dl.grafana.com/oss/release/grafana-11.0.0-1.x86_64.rpm
```

Start and enable:

```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Check:

```bash
sudo systemctl status grafana-server --no-pager
```

Verify port:

```bash
sudo ss -tulpen | grep 3000
```

Open browser:

`http://<EC2_PUBLIC_IP>:3000`

Login:  
- admin / admin  

---

## **5. Install CloudWatch Agent**

```bash
sudo yum install -y amazon-cloudwatch-agent
```

---

## **6. Create CloudWatch Agent Configuration File**

```bash
nano cloudwatch-config.json
```

Paste:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent", "mem_available"],
        "metrics_collection_interval": 60
      },
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_user",
          "cpu_usage_system"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

Save & exit.

---

## **7. Start CloudWatch Agent**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
 -a fetch-config -m ec2 -c file:/home/ec2-user/cloudwatch-config.json -s
```

Check:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

Logs:

```bash
sudo tail -n 200 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

---

## **8. Store Config in SSM (Optional)**

Install jq:

```bash
sudo yum install -y jq
```

Compact JSON:

```bash
cat cloudwatch-config.json | jq -c . > cloudwatch-config-compact.json
```

Upload:

```bash
aws ssm put-parameter --name "/cwagent/config" --type "String" \
 --value "`cat cloudwatch-config-compact.json`" --overwrite
```

Apply:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
 -a fetch-config -m ec2 -c ssm:/cwagent/config -s
```

---

## **9. Configure CloudWatch Data Source in Grafana**

- Grafana → Configuration → Data Sources  
- Add **CloudWatch**  
- Authentication: IAM Role  
- Region: `ap-south-1`  
- Save & Test  

---

## **10. Import Grafana Dashboards**

- Dashboards → Import  
- Dashboard IDs:
  - 1265  
  - 617  
- Data Source: CloudWatch  
- Region: ap-south-1  

---

## **11. Create CloudWatch Alarm (Optional)**

```bash
aws cloudwatch put-metric-alarm \
 --alarm-name "High-CPU-80" \
 --metric-name CPUUtilization \
 --namespace AWS/EC2 \
 --statistic Average \
 --period 60 \
 --threshold 80 \
 --comparison-operator GreaterThanThreshold \
 --evaluation-periods 2 \
 --dimensions Name=InstanceId,Value=<INSTANCE_ID> \
 --alarm-actions <SNS_TOPIC_ARN>
```

---

## **12. Run Stress Test (Optional)**

Install:

```bash
sudo yum install -y stress
```

Run:

```bash
stress --cpu 2 --vm 1 --vm-bytes 500M --timeout 120
```

---

## **13. Verify Metrics**

- CloudWatch → Metrics → CWAgent  
- Grafana dashboards  
- Refresh every 5s  

---

## **14. Reboot & Verify Auto-Start**

Reboot:

```bash
sudo reboot
```

Check services:

```bash
sudo systemctl status grafana-server
sudo systemctl status amazon-cloudwatch-agent
```

---

## **15. Troubleshooting**

### Grafana not loading
- Check port 3000 inbound  
- Check service status  

### No data in Grafana
- Region wrong  
- Instance wrong  
- Agent not running  

### CloudWatch Agent errors
Check logs:

```bash
sudo tail -n 200 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

---

## **16. Cleanup**

Stop services:

```bash
sudo systemctl stop grafana-server
sudo systemctl stop amazon-cloudwatch-agent
```

Terminate EC2 if needed.


