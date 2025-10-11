## 🎯 SCENARIO 11: SELINUX CONFIGURATION (30 minutes)

### THE SITUATION:
Your organization is deploying web servers (Apache and Nginx) with custom configurations. SELinux is blocking various operations. You need to properly configure SELinux contexts, booleans, and port labels to allow legitimate operations while maintaining security.

### YOUR TASKS:
1. Verify SELinux is in enforcing mode
2. Configure Apache to serve from `/web` directory
3. Configure Nginx to serve from `/data/www`
4. Allow Apache to serve user home directories
5. Configure custom ports (8080 for Apache, 8081 for Nginx)
6. Enable Apache network connections and NFS access
7. Troubleshoot and fix AVC denials
8. Make all changes permanent

<details>
<summary><b>Part 1: Verify SELinux Status</b></summary>

### Part 1: Verify SELinux Status

```bash
# Check SELinux mode
getenforce                                     # Should show: Enforcing

# Detailed status
sestatus

# If disabled or permissive, enable it
sudo setenforce 1                              # Temporary

# For permanent (if needed)
sudo vi /etc/selinux/config
# Set: SELINUX=enforcing

# Reboot if you changed config file
```

</details>

---

<details>
<summary><b>Part 2: Apache - Custom Web Directory</b></summary>
### Part 2: Apache - Custom Web Directory

```bash
# Install Apache
sudo yum install httpd -y

# Create custom web directory
sudo mkdir -p /web/html
sudo mkdir -p /web/cgi-bin

# Create test files
echo "<h1>Apache Test from /web</h1>" | sudo tee /web/html/index.html
echo "<h2>This is a custom directory</h2>" | sudo tee /web/html/test.html

# Check current context (WRONG!)
ls -Z /web
# Shows: unconfined_u:object_r:default_t:s0
# Should be: httpd_sys_content_t
```

#### Fix Apache Directory Context:
```bash
# Method 1: Temporary fix with chcon (not persistent)
sudo chcon -R -t httpd_sys_content_t /web/html

# Verify
ls -Z /web/html

# Method 2: Permanent fix with semanage (CORRECT WAY)
# First restore to default
sudo restorecon -R -v /web

# Add persistent rule
sudo semanage fcontext -a -t httpd_sys_content_t '/web(/.*)?'
sudo semanage fcontext -a -t httpd_sys_script_exec_t '/web/cgi-bin(/.*)?'

# Apply the policy
sudo restorecon -R -v /web

# Verify
ls -Z /web
ls -Z /web/html
ls -Z /web/cgi-bin

# View the rule
sudo semanage fcontext -l | grep '/web'
```

#### Configure Apache:
```bash
# Edit Apache config
sudo vi /etc/httpd/conf/httpd.conf

# Change DocumentRoot
# Find and change:
# DocumentRoot "/var/www/html"
# To:
# DocumentRoot "/web/html"

# Add directory configuration
# Add after DocumentRoot:
```

```apache
<Directory "/web/html">
    AllowOverride None
    Require all granted
</Directory>

<Directory "/web/cgi-bin">
    AllowOverride None
    Options +ExecCGI
    Require all granted
</Directory>
```

```bash
# Test Apache config
sudo httpd -t

# Start Apache
sudo systemctl enable --now httpd
sudo systemctl status httpd
```

</details>

---

<details>
<summary><b>Part 3: Nginx - Custom Web Directory</b></summary>

### Part 3: Nginx - Custom Web Directory

```bash
# Install Nginx
sudo yum install nginx -y

# Create Nginx custom directory
sudo mkdir -p /data/www/html

# Create test files
echo "<h1>Nginx Test from /data/www</h1>" | sudo tee /data/www/html/index.html

# Check context (WRONG!)
ls -Z /data
```

#### Fix Nginx Directory Context:
```bash
# Add SELinux rule for Nginx
sudo semanage fcontext -a -t httpd_sys_content_t '/data/www(/.*)?'

# Apply
sudo restorecon -R -v /data/www

# Verify
ls -Z /data/www
```

#### Configure Nginx:
```bash
# Edit Nginx config
sudo vi /etc/nginx/nginx.conf

# Find 'server' block and change root:
# root         /usr/share/nginx/html;
# To:
# root         /data/www/html;
```

</details>

---

<details>
<summary><b>Part 4: Apache User Home Directories</b></summary>

### Part 4: Apache User Home Directories

```bash
# Create user for testing
sudo useradd webuser
echo "password123" | sudo passwd --stdin webuser

# Create user's public_html
sudo mkdir -p /home/webuser/public_html
echo "<h1>User Home Directory</h1>" | sudo tee /home/webuser/public_html/index.html
sudo chown -R webuser:webuser /home/webuser/public_html
sudo chmod 755 /home/webuser
sudo chmod 755 /home/webuser/public_html

# Check SELinux boolean for home directories
getsebool httpd_enable_homedirs
# Should be: off

# Enable the boolean PERMANENTLY
sudo setsebool -P httpd_enable_homedirs on

# Verify
getsebool httpd_enable_homedirs
# Should be: on

# List all httpd booleans
getsebool -a | grep httpd
```

#### Configure Apache for UserDir:
```bash
# Edit userdir config
sudo vi /etc/httpd/conf.d/userdir.conf

# Find and change:
# UserDir disabled
# To:
# UserDir public_html

# Uncomment:
# <Directory "/home/*/public_html">
#     AllowOverride FileInfo AuthConfig Limit Indexes
#     Options MultiViews Indexes SymLinksIfOwnerMatch IncludesNoExec
#     Require method GET POST OPTIONS
# </Directory>
```

```bash
# Reload Apache
sudo systemctl reload httpd

# Test (if DNS/hosts configured)
# curl http://localhost/~webuser/
```

</details>

---

<details>
<summary><b>Part 5: Custom Ports (8080 for Apache, 8081 for Nginx)</b></summary>

### Part 5: Custom Ports (8080 for Apache, 8081 for Nginx)

```bash
# Check current HTTP ports
sudo semanage port -l | grep http_port_t
# Default shows: 80, 81, 443, 488, 8008, 8009, 8443, 9000

# Apache will use 8080
# Edit Apache config
sudo vi /etc/httpd/conf/httpd.conf
# Change: Listen 80
# To: Listen 8080

# Add port 8080 to SELinux
sudo semanage port -a -t http_port_t -p tcp 8080

# If port already defined, modify it:
# sudo semanage port -m -t http_port_t -p tcp 8080

# Verify
sudo semanage port -l | grep http_port_t | grep 8080
```

```bash
# Nginx will use 8081
# Edit Nginx config
sudo vi /etc/nginx/nginx.conf
# Find: listen       80;
# Change to: listen       8081;

# Add port 8081 to SELinux
sudo semanage port -a -t http_port_t -p tcp 8081

# Verify
sudo semanage port -l | grep http_port_t | grep 8081
```

```bash
# Configure Firewall
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=8081/tcp
sudo firewall-cmd --reload

# Restart services
sudo systemctl restart httpd
sudo systemctl restart nginx

# Test
curl http://localhost:8080
curl http://localhost:8081
```

</details>

---

<details>
<summary><b>Part 6: Apache Network & NFS Booleans</b></summary>

### Part 6: Apache Network & NFS Booleans

```bash
# Scenario: Apache needs to connect to remote database
# Check boolean
getsebool httpd_can_network_connect
getsebool httpd_can_network_connect_db

# Enable network connections PERMANENTLY
sudo setsebool -P httpd_can_network_connect on
sudo setsebool -P httpd_can_network_connect_db on

# Verify
getsebool httpd_can_network_connect
getsebool httpd_can_network_connect_db
```

```bash
# Scenario: Apache serving files from NFS mount
# Check boolean
getsebool httpd_use_nfs

# Enable NFS access PERMANENTLY
sudo setsebool -P httpd_use_nfs on

# Verify
getsebool httpd_use_nfs
```

```bash
# Scenario: Apache executing scripts via CGI
getsebool httpd_enable_cgi

# Enable if needed
sudo setsebool -P httpd_enable_cgi on
```

#### Common Apache SELinux Booleans:
```bash
# View all httpd booleans
getsebool -a | grep httpd

# Common ones:
# httpd_enable_homedirs     - Serve user home directories
# httpd_can_network_connect - Connect to network (databases, APIs)
# httpd_can_network_connect_db - Connect to database ports
# httpd_use_nfs             - Access NFS mounted content
# httpd_use_cifs            - Access CIFS/SMB mounted content
# httpd_enable_cgi          - Execute CGI scripts
# httpd_unified             - Full read/write to all httpd content
# httpd_builtin_scripting   - Execute PHP, Python scripts

# Enable multiple at once
sudo setsebool -P httpd_enable_homedirs on \
                  httpd_can_network_connect on \
                  httpd_use_nfs on
```

</details>

---

<details>
<summary><b>Part 7: Troubleshooting SELinux Denials</b></summary>

### Part 7: Troubleshooting SELinux Denials

```bash
# Install troubleshooting tools
sudo yum install setroubleshoot-server policycoreutils-python-utils -y

# View recent denials
sudo ausearch -m avc -ts recent

# Detailed analysis
sudo sealert -a /var/log/audit/audit.log

# View with journalctl
sudo journalctl -t setroubleshoot --since today

# Real-time monitoring
sudo tail -f /var/log/audit/audit.log | grep denied
```

#### Example: Fix a Denial
```bash
# Simulate a problem
echo "<h1>Test</h1>" | sudo tee /web/test.html
sudo chcon -t default_t /web/test.html    # Wrong context

# Try to access via Apache (will fail)
curl http://localhost:8080/test.html

# Check for denials
sudo ausearch -m avc -ts recent | grep httpd

# Get recommendations
sudo sealert -a /var/log/audit/audit.log | grep -A 20 test.html

# Fix it
sudo restorecon -v /web/test.html

# Test again
curl http://localhost:8080/test.html
```

</details>

---

<details>
<summary><b>Part 8: Verification & Testing</b></summary>

### Part 8: Verification & Testing

```bash
# Check all SELinux settings
getenforce
sestatus

# Verify contexts
ls -Z /web
ls -Z /data/www
ls -Z /home/webuser/public_html

# Verify booleans (all should be ON)
getsebool httpd_enable_homedirs
getsebool httpd_can_network_connect
getsebool httpd_use_nfs
getsebool httpd_enable_cgi

# Verify ports
sudo semanage port -l | grep http_port_t | grep -E '8080|8081'

# Verify fcontext rules
sudo semanage fcontext -l | grep -E '/web|/data/www'

# Test Apache
curl http://localhost:8080
curl http://localhost:8080/test.html
curl http://localhost/~webuser/

# Test Nginx
curl http://localhost:8081

# Check for denials
sudo ausearch -m avc -ts today | grep denied
# Should be empty or only show irrelevant denials

# View all httpd-related rules
sudo semanage fcontext -l | grep httpd
sudo semanage port -l | grep http
getsebool -a | grep httpd
```

</details>

---

<details>
<summary><b>Part 9: Common RHCSA Exam Scenarios</b></summary>

### Part 9: Common RHCSA Exam Scenarios

```bash
# Scenario 1: Apache can't access /srv/website
sudo mkdir -p /srv/website
echo "Test" | sudo tee /srv/website/index.html
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/website(/.*)?'
sudo restorecon -R -v /srv/website

# Scenario 2: Apache needs port 8888
sudo semanage port -a -t http_port_t -p tcp 8888

# Scenario 3: Apache can't write to /var/log/webapp
sudo mkdir -p /var/log/webapp
sudo semanage fcontext -a -t httpd_log_t '/var/log/webapp(/.*)?'
sudo restorecon -R -v /var/log/webapp

# Scenario 4: Apache needs to send email
sudo setsebool -P httpd_can_sendmail on

# Scenario 5: Allow httpd to connect to LDAP
sudo setsebool -P httpd_can_connect_ldap on

# Scenario 6: PHP Upload Directory (writable content)
sudo mkdir -p /web/html/uploads
sudo chown apache:apache /web/html/uploads
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/web/html/uploads(/.*)?'
sudo restorecon -R -v /web/html/uploads

# Scenario 7: CGI Scripts
sudo mkdir -p /web/cgi-bin
echo '#!/bin/bash' | sudo tee /web/cgi-bin/test.cgi
echo 'echo "Content-type: text/html"' | sudo tee -a /web/cgi-bin/test.cgi
echo 'echo ""' | sudo tee -a /web/cgi-bin/test.cgi
echo 'echo "<h1>CGI Test</h1>"' | sudo tee -a /web/cgi-bin/test.cgi
sudo chmod +x /web/cgi-bin/test.cgi
sudo semanage fcontext -a -t httpd_sys_script_exec_t '/web/cgi-bin(/.*)?'
sudo restorecon -R -v /web/cgi-bin
sudo setsebool -P httpd_enable_cgi on
```

</details>

---

### Quick Reference Card:

```bash
# File Contexts
# View: ls -Z /path
# Set temporary: sudo chcon -t TYPE /path
# Set permanent: sudo semanage fcontext -a -t TYPE '/path(/.*)?'
# Apply: sudo restorecon -R -v /path
# List rules: sudo semanage fcontext -l | grep /path

# Common httpd contexts:
# httpd_sys_content_t         - Read-only web content
# httpd_sys_rw_content_t      - Read-write content (uploads)
# httpd_sys_script_exec_t     - Executable scripts (CGI)
# httpd_log_t                 - Log files

# Booleans
# View: getsebool BOOLEAN
# Set temporary: sudo setsebool BOOLEAN on
# Set permanent: sudo setsebool -P BOOLEAN on
# List all httpd: getsebool -a | grep httpd

# Ports
# View: sudo semanage port -l | grep http_port_t
# Add: sudo semanage port -a -t http_port_t -p tcp PORT
# Delete: sudo semanage port -d -t http_port_t -p tcp PORT

# Troubleshooting
# Recent denials: sudo ausearch -m avc -ts recent
# Detailed analysis: sudo sealert -a /var/log/audit/audit.log
# Monitor live: sudo tail -f /var/log/audit/audit.log | grep denied

# SELinux Modes
# Check: getenforce
# Temporary: sudo setenforce 1 (enforcing) or 0 (permissive)
# Permanent: Edit /etc/selinux/config
```

### Final Verification Checklist:

```bash
# [ ] SELinux is in Enforcing mode
getenforce

# [ ] Apache serves from /web
curl http://localhost:8080

# [ ] Nginx serves from /data/www
curl http://localhost:8081

# [ ] User home directories work
curl http://localhost/~webuser/

# [ ] Custom ports configured
sudo semanage port -l | grep -E '8080|8081'

# [ ] All booleans are permanent (-P flag used)
getsebool -a | grep httpd | grep " on"

# [ ] File contexts are permanent (semanage used, not chcon)
sudo semanage fcontext -l | grep -E '/web|/data'

# [ ] No SELinux denials for httpd/nginx
sudo ausearch -m avc -ts today | grep -E 'httpd|nginx' | grep denied

# [ ] Services running
sudo systemctl status httpd
sudo systemctl status nginx
```