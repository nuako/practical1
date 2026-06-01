## Operations Runbook
Project: Cloud VPS Automation Platform
Owner: Operations Lead
Scope: Day-to-day operations, troubleshooting, and recovery

0. Quick Architecture Map
Customer → FOSSBilling → Webhook (/webhook/order-paid) → create_vm.py → Proxmox API → VM
                                                        ↘ logs (/var/log/cloud-provisioning.log)
Monitoring: Netdata (http://<host>:19999)

1. Pre-Flight Health Check (run before demos/changes)
Commands
# Proxmox API
curl -k https://127.0.0.1:8006/api2/json/version

# Services
systemctl is-active pveproxy pvedaemon pvestatd apache2 docker

# Webhook
curl http://127.0.0.1:5000/health

# Disk / RAM / CPU
df -h
free -h
uptime

# Expected
- All services active
- Disk < 90% used
- Webhook returns OK/healthy


2. Start / Stop / Restart Services
- Proxmox services:         systemctl restart pveproxy pvedaemon pvestatd
- Web server (FOSSBilling): systemctl restart apache2
- Webhook (Flask):          pkill -f webhook_listener.py
                            python3 /opt/cloud-automation/webhook_listener.py &

3. Provision a VPS (Manual Test)
# Command (Python):             cd /opt/cloud-automation
                                python3 create_vm.py
# Expected
- New VM appears in Proxmox
- Log entry created:            tail -f /var/log/cloud-provisioning.log


4. Test Webhook End-to-End
# Trigger webhook manually
curl -X POST http://127.0.0.1:5000/webhook/order-paid \
-H "Content-Type: application/json" \
-d '{
  "order_id": 1001,
  "product_slug": "small-vps",
  "config": {"os": "ubuntu"},
  "customer_email": "test@example.com"
}'

# Expected
- VM created
- Log updated
- No errors in console

5. Verify FOSSBilling → Webhook
- In FOSSBilling
- Complete a test order
- Check:        tail -f /var/log/cloud-provisioning.log
- If nothing happens
- Verify webhook URL in FOSSBilling
- Check port 5000 is open:  ss -tulnp | grep 5000

6. Retrieve VM IP Address
- Using Proxmox agent: qm agent <vmid> network-get-interfaces
# Requirement
- QEMU Guest Agent installed in template

7. SSH into VPS (Verification)
ssh ubuntu@<vm-ip>
If using password: Use generated password from logs

8. Monitor System (Netdata)
- Open dashboard
- http://<server-ip>:19999
- Watch
    CPU spikes
    RAM usage
    Disk usage
    Network

9. Check Logs
Provisioning logs: tail -f /var/log/cloud-provisioning.log
System logs:       journalctl -xe

10. Backup Operations
Manual backup:      tar -czvf /backup/backup_$(date +%F).tar.gz /opt/cloud-automation
Database backup:    mysqldump -u root -p db_name > backup.sql

11. Log Rotation (Verify)
Force rotation test:    logrotate -f /etc/logrotate.d/cloud-automation
Check rotated files:    ls /var/log | grep cloud-provisioning

12. Performance / Load Test
ab -n 100 -c 10 http://<fossbilling-ip>/
- Metrics to note
    Requests/sec
    Time per request
    Failed requests

13. Common Issues & Fixes
- Proxmox API Timeout:  systemctl restart pveproxy pvedaemon pvestatd
- Webhook Not Working:  curl http://127.0.0.1:5000/health
                        ss -tulnp | grep 5000
- VM Created but No IP: qm config <vmid> | grep agent
    If missing:         qm set <vmid> --agent 1
- Script Syntax Error:  python3 -m py_compile create_vm.py
- Disk Full
    df -h
    rm -rf /var/log/*.gz
- High CPU
    top
    kill -9 <pid>

14. Recovery Checklist
- Restart core services
- Verify Proxmox API
- Test webhook
- Run manual provisioning
- Validate full flow
 
15. Daily Operations Checklist
- Proxmox services running
- Webhook reachable
- Disk usage < 90%
- Logs updating
- Netdata healthy
- Test provisioning works

16. Shutdown Procedure (if needed)
systemctl stop apache2
systemctl stop docker

Shutdown host: shutdown now