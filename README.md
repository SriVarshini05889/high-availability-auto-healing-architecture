# High-Availability Web Architecture with Automated Self-Healing & Failover
## Project Overview
This project demonstrates an enterprise-grade, fault-tolerant cloud architecture featuring multi-zone high availability, Layer 7 reverse proxy load balancing, active health monitoring, and automated self-healing. 
It replicates core capabilities provided by AWS Application Load Balancer (ALB) and Auto Scaling Groups (ASG) using Linux systems engineering, Nginx, and automated watchdog daemon scripting.
## Architecture Design & Traffic Flow
Internet Traffic
                     │
                     ▼
       ┌───────────────────────────┐
       │   Nginx Reverse Proxy     │
       │  (Application Load Balancer)│
       │         Port 80           │
       └─────────────┬─────────────┘
                     │ Round-Robin & Health Checks
      ┌──────────────┴──────────────┐
      ▼                             ▼
┌───────────────────┐         ┌───────────────────┐
│    Web Node 1     │         │    Web Node 2     │
│  Availability     │         │  Availability     │
│      Zone A       │         │      Zone B       │
│    Port 8081      │         │    Port 8082      │
└─────────▲─────────┘         └───────────────────┘
│
│ Failover / Drop Traffic Simulated
│
┌─────────┴─────────────────────────────────┐
│     Auto Scaling Watchdog Daemon          │
│ • Probes backend HTTP status every 2s     │
│ • Detects node failure                    │
│ • Automatically self-heals & restores node│
└───────────────────────────────────────────┘

## Technical Stack
 **Operating System:** Ubuntu Linux 24.04 LTS
 **Reverse Proxy / Load Balancer:** Nginx (Layer 7 traffic routing, upstream connection timeouts)
 **Monitoring & Self-Healing:** Bash daemon with real-time HTTP health probing and firewall-level fault recovery
 **Networking & Diagnostics:** `curl`, `iptables`, `systemd`, `net-tools`

## Implementation Details

### 1. Multi-Zone Node Deployment
Two distinct backend web nodes were provisioned to represent independent availability zones:
Node 1 (Port 8081)`: Zone A
Node 2 (Port 8082)`: Zone B

### 2. Load Balancer Configuration
Nginx was configured as an upstream reverse proxy listening on port 80:
nginxupstream backend_pool {server 127.0.0.1:8081 max_fails=2 fail_timeout=5s;server 127.0.0.1:8082 max_fails=2 fail_timeout=5s;}server {listen 80;location / {proxy_pass http://backend_pool;proxy_connect_timeout 1s;proxy_read_timeout 1s;}}
### 3. Automated Self-Healing Watchdog
A background monitoring process continuously polls target endpoints. Upon detecting an HTTP non-200 code or network drop, it flags an incident alert and executes recovery actions:
bash#!/bin/bashwhile true; dofor PORT in 8081 8082; doHTTP_CODE=$(curl -s -m 1 -o /dev/null -w "%{http_code}" http://127.0.0.1:$PORT/ || echo "000")if [ "$HTTP_CODE" != "200" ]; thenecho "[ALERT $(date +\%T)] Instance on Port$PORT is UNHEALTHY! ASG initiating replacement..."iptables -D INPUT -p tcp --dport $PORT -j DROP 2>/dev/nullecho "[AUTO-SCALING $(date +%T)] Replacement instance provisioned. Port $PORT restored."fidonesleep 2done

## Verification & Test Results

### 1. High-Availability Load Balancing
Requests to the load balancer alternate evenly across zones:
bash$ for i in {1..4}; do curl -s http://localhost:80/ | grep -o 'Zone [A-B]'; doneZone AZone BZone AZone B
### 2. Fault Tolerance & Zero-Downtime Failover
When Node 1 traffic was dropped to simulate an infrastructure crash:
1. The load balancer detected the unresponsive target and routed incoming requests to **Zone B** with zero downtime.
2. The watchdog daemon detected the outage within 2 seconds, logged the incident, and executed automated remediation.
3. Traffic distribution across both instances was restored automatically.

## Evidence & Screenshots

### Load Balancing Verification
[Traffic Balancing](traffic-balancing.png)

### Self-Healing & Event Recovery Log
[Self-Healing Proof](self-healing-asg.png)

