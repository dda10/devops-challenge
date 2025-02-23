# Troubleshooting High Memory Usage in NGINX Load Balancer

## Scenario

You are working as a DevOps Engineer on a cloud-based infrastructure where a virtual machine (VM), running Ubuntu 24.04, with 64GB of storage is under your management. Recently, your monitoring tools have reported that the VM is consistently running at 99% memory usage. This VM is responsible for only running one service - a NGINX load balancer as a traffic router for upstream services.

## Troubleshooting Steps

1. **Check Current Memory Usage**  
   ```sh
   free -h
   vmstat -s
   ```

2. **Identify Processes Consuming Memory**  
   ```sh
   htop
   ps aux --sort=-%mem | head -10
   ```

3. **Analyze NGINX Worker Processes**  
   ```sh
   htop
   ps -eo pid,comm,%mem,%cpu --sort=-%mem | grep nginx
   ```

4. **Check NGINX Logs for Anomalies**  
   ```sh
   tail -f /var/log/nginx/access.log
   tail -f /var/log/nginx/error.log
   ```

5. **Inspect Open Connections**  
   ```sh
   netstat -anp | grep nginx | wc -l
   ```

6. **Review NGINX Configuration**  
   ```sh
   cat /etc/nginx/sites-available/*.conf
   cat /etc/nginx/conf.d/nginx.conf
   cat /etc/nginx/nginx.conf
   ```

## Possible Causes and Solutions

### 1. **High Number of Open Connections**
   - **Cause**: A surge in traffic or slow upstream responses can cause a high number of active connections.
   - **Impact**: Leads to increased memory usage and potential crashes.
   - **Solution**: Tune NGINX parameters in `/etc/nginx/nginx.conf`:
     ```nginx
     worker_processes auto;
     worker_rlimit_nofile 100000;
     events {
         worker_connections 10000;
         multi_accept on;
     }
     ```

### 2. **Memory Leak in NGINX Modules or Configuration**
   - **Cause**: Custom modules or improper caching configurations can cause memory to be retained.
   - **Impact**: Gradual memory exhaustion leading to service degradation.
   - **Solution**: Restart NGINX and monitor:
     ```sh
     systemctl restart nginx
     ```
     If memory usage stabilizes, inspect custom modules.
     ```sh
     ls /etc/nginx/modules
     ```

### 3. **Logging Configuration Issues**
   - **Cause**: Excessive logging (debug mode enabled) can increase memory and I/O operations.
   - **Impact**: Increased disk and memory usage.
   - **Solution**: Reduce log levels in `/etc/nginx/nginx.conf`:
     ```nginx
     error_log /var/log/nginx/error.log warn;
     ```

### 4. **Insufficient System Resources**
   - **Cause**: The VM may not have enough resources to handle incoming traffic.
   - **Impact**: Performance degradation and increased swap usage.
   - **Solution**: Upgrade VM memory or optimize load balancing strategies.

### 5. **Slowloris Attack or DDOS Attack**
   - **Cause**: Malicious requests that keep connections open without sending data. If encountered DDOS attack, the attacker sends losts of connections and dirty traffic to the server, make it exausted from request processing
   - **Impact**: Exhausts worker processes and memory.
   - **Solution**: Enable rate limiting and timeouts in NGINX:
     ```nginx
     client_body_timeout 10;
     client_header_timeout 10;
     keepalive_timeout 5 5;
     limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
     ```

## Recovery Steps

1. **Restart NGINX**  
   ```sh
   systemctl restart nginx
   ```

2. **Clear Cache and Temporary Files**  
   ```sh
   rm -rf /var/cache/nginx/*
   ```

3. **Upgrade VM Resources if Needed**  
   ```sh
   aws ec2 modify-instance-attribute --instance-type m5.large --instance-id <instance-id>
   ```

4. **Implement Auto Scaling for Traffic Spikes**  
   - Deploy NGINX on multiple instances behind an **AWS ALB**, **AWS WAF**, **AWS Cloudfront**. It will spread traffic spikes through AWS CDN networks, make DDOS attacks less efficient 


