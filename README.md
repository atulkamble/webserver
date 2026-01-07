# Webserver Nginx
# nginx on EC2
```
// updating a ec2 instanceb
sudo yum update -y

// install nginx server on ec2 
sudo yum install nginx -y

// check for nginx version
sudo nginx -v

// start nginx
sudo nginx

sudo systemctl enable nginx

sudo systemctl start nginx

sudo systemctl status nginx

```


# Install and Run Nginx with Docker on EC2
```
// install docker
sudo yum install docker

// check docker version
docker --version

// start docker engine
sudo systemctl start docker

// pull docker image 
sudo docker pull nginx:latest 

// list docker images
sudo docker images

// run nginx docker image 
sudo docker run -p 80:80 nginx:latest
   
// Check web-browser for nginx server

http://public-ip-of-ec2-instance
```

# Webserver Apache
Basic EC2 Webserver Script. Apache Server | EC2 Advanced | Terraform Script.

// Webserver | Configuration
```
EC2 | amazon-linux-2023 | t2 micro
Security Group: http:80 , https:443, ssh:22
ssh >> update instance, httpd, /var/www/html/index.html
<h1> This is webserver </h1>
public ip
```

// Commands after ssh

```
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo usermod -a -G apache ec2-user
sudo chmod 777 /var/www/html
cd /var/www/html
touch index.html
sudo nano index.html
cat index.html
history
```

Paste following Code to
EC2 - Advanced Details - user data

```
#!/bin/bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
if ! getent group apache >/dev/null; then
    groupadd apache
fi
usermod -aG apache ec2-user
chmod 755 /var/www/html
echo "<h1>Hello from $(hostname -f) webserver</h1>" > /var/www/html/index.html
```
// AzureVM User Data Script 
```
#!/bin/bash
apt update -y
apt install apache2 -y

systemctl start apache2
systemctl enable apache2

cat <<EOF > /var/www/html/index.html
<html>
<head>
<title>Ubuntu Web Server</title>
</head>
<body>
<h1>Apache Web Server Running</h1>
<p>Access this server using the URL printed in logs</p>
</body>
</html>
EOF

HOST_IP=$(hostname -I | awk '{print $1}')

echo "----------------------------------"
echo "Web Server URL:"
echo "http://$HOST_IP"
echo "----------------------------------"

```
// Deploy Webserver on Azure VM
// Task: Configuration of Webserver on Ubuntu Server 2022
```
sudo apt update -y
sudo apt install apache2
sudo apt install mini-httpd -y
sudo systemctl start mini-httpd
sudo systemctl enable mini-httpd
sudo chmod 777 /var/www/html
cd /var/www/html
sudo touch index.html
nano --version
nano index.html
```
// Copy following code to index.html
```
<!DOCTYPE html>
<html>
<body>

<h1> This is a Webserver. </h1>
<p> Hello form EC2 </p>

</body>
</html>
```

Check public-ip of machine
```
http://vm-public-ip/index.html
```

// Launch website from git to EC2 directly via terraform
```
sudo yum update -y
sudo yum install git -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo usermod -a -G apache ec2-user
sudo chmod 777 /var/www/html
git clone github-repository-URL
mv * source-folder /var/www/html
```
// Check public instance ip

// ASG - Template 
```
#!/bin/bash
yum update -y
yum install httpd git -y

systemctl start httpd
systemctl enable httpd

usermod -a -G apache ec2-user
chmod 775 /var/www/html
chown -R ec2-user:apache /var/www/html

cd /var/www/html
git clone https://github.com/atulkamble/pong-game.git
mv pong-game/* .
rm -rf pong-game
```

kit for deploying multiple web servers on **Azure VMs** for **Linux (Ubuntu)** and **Windows Server**—with Azure CLI, one-shot provisioning scripts (cloud-init/PowerShell), NSG rules, and quick validation steps.

---

# 1) Create VMs + Open Ports (Azure CLI)

```bash
# Variables
RG=websrv-rg
LOC=eastus
LINUXVM=ubuntux-web
WINVM=win-web
ADMINUSER=azureuser

az group create -n $RG -l $LOC

# NSG allowing 22, 80, 443, 3389 (RDP optional), 8080 (Tomcat), 3000 (Node demo)
az network nsg create -g $RG -n web-nsg
for p in 22 80 443 3389 8080 3000; do
  az network nsg rule create -g $RG --nsg-name web-nsg -n allow-${p} \
    --priority $((1000 + p)) --access Allow --protocol Tcp --direction Inbound \
    --source-address-prefixes '*' --source-port-ranges '*' \
    --destination-address-prefixes '*' --destination-port-ranges $p
done

# Linux VM (Ubuntu 22.04 LTS) with cloud-init (filled below)
az vm create -g $RG -n $LINUXVM \
  --image Ubuntu2204 \
  --admin-username $ADMINUSER \
  --generate-ssh-keys \
  --nsg web-nsg \
  --custom-data cloud-init-linux.yaml

# Windows Server 2022 with Custom Script Extension (we attach the script next)
az vm create -g $RG -n $WINVM \
  --image MicrosoftWindowsServer:WindowsServer:2022-datacenter-azure-edition:latest \
  --admin-username $ADMINUSER --admin-password 'P@ssw0rd!123456' \
  --nsg web-nsg
```

Grab the public IPs to test later:

```bash
az vm list-ip-addresses -g $RG -n $LINUXVM -o table
az vm list-ip-addresses -g $RG -n $WINVM -o table
```

---

# 2) Linux: cloud-init (Nginx, Apache, Tomcat, Node.js+PM2)

Save as `cloud-init-linux.yaml` before you run `az vm create`.

```yaml
#cloud-config
package_update: true
package_upgrade: true

packages:
  - nginx
  - apache2
  - openjdk-17-jre
  - curl
  - git
  - ufw

write_files:
  - path: /var/www/html/index.nginx-deploy.html
    permissions: "0644"
    content: |
      <h1>Nginx up on Ubuntu</h1>
  - path: /var/www/html/index.apache-deploy.html
    permissions: "0644"
    content: |
      <h1>Apache2 up on Ubuntu</h1>
  - path: /opt/node-demo/server.js
    permissions: "0644"
    content: |
      const http = require('http');
      const port = 3000;
      const server = http.createServer((req,res)=>{
        res.writeHead(200, {'Content-Type':'text/plain'});
        res.end('Node.js app via PM2 is running\n');
      });
      server.listen(port, ()=>console.log(`Listening on ${port}`));
  - path: /etc/systemd/system/tomcat.service
    permissions: "0644"
    content: |
      [Unit]
      Description=Apache Tomcat 10
      After=network.target
      [Service]
      Type=forking
      Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
      Environment=CATALINA_HOME=/opt/tomcat
      Environment=CATALINA_BASE=/opt/tomcat
      ExecStart=/opt/tomcat/bin/startup.sh
      ExecStop=/opt/tomcat/bin/shutdown.sh
      User=tomcat
      Group=tomcat
      Restart=always
      [Install]
      WantedBy=multi-user.target

runcmd:
  # Basic hardening & firewall
  - ufw allow 22/tcp
  - ufw allow 80/tcp
  - ufw allow 443/tcp
  - ufw allow 8080/tcp
  - ufw allow 3000/tcp
  - ufw --force enable

  # --- NGINX ---
  - systemctl enable nginx --now
  - bash -lc 'echo "server { listen 80 default_server; root /var/www/html; index index.nginx-deploy.html; }" > /etc/nginx/sites-available/default'
  - nginx -t && systemctl reload nginx

  # --- APACHE2 (on port 8081 via reverse-proxy path or just keep as alt index) ---
  - bash -lc 'echo "<VirtualHost *:80> ServerName _ default_ ServerAlias _ DocumentRoot /var/www/html </VirtualHost>" > /etc/apache2/sites-available/000-default.conf'
  - a2enmod proxy proxy_http
  - systemctl enable apache2 --now
  # keep apache to serve /index.apache-deploy.html (can be switched later if desired)

  # --- Node.js + PM2 ---
  - curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
  - apt-get install -y nodejs
  - npm i -g pm2
  - bash -lc 'cd /opt/node-demo && pm2 start server.js --name node-demo && pm2 startup systemd -u root --hp /root && pm2 save'

  # --- Tomcat 10 ---
  - useradd -m -U -d /opt/tomcat -s /bin/false tomcat
  - bash -lc 'curl -L -o /tmp/tomcat.tar.gz https://downloads.apache.org/tomcat/tomcat-10/v10.1.29/bin/apache-tomcat-10.1.29.tar.gz'
  - mkdir -p /opt/tomcat
  - tar -xzf /tmp/tomcat.tar.gz -C /opt/tomcat --strip-components=1
  - chown -R tomcat:tomcat /opt/tomcat
  - chmod +x /opt/tomcat/bin/*.sh
  - systemctl daemon-reload
  - systemctl enable tomcat --now

  # --- Optional: Nginx reverse-proxy for Node and Tomcat paths ---
  - bash -lc 'cat > /etc/nginx/conf.d/upstreams.conf <<EOF
    upstream node_demo { server 127.0.0.1:3000; }
    upstream tomcat_svc { server 127.0.0.1:8080; }
    server {
      listen 80;
      server_name _;
      location / { root /var/www/html; index index.nginx-deploy.html; }
      location /node/ { proxy_pass http://node_demo/; }
      location /tomcat/ { proxy_pass http://tomcat_svc/; }
    }
    EOF'
  - rm -f /etc/nginx/sites-enabled/default
  - nginx -t && systemctl reload nginx
```

### Test (Linux)

* Nginx default page: `http://<linux-public-ip>/`
* Node app: `http://<linux-public-ip>/node/`
* Tomcat Manager (if enabled): `http://<linux-public-ip>/tomcat/` (serves Tomcat root)
* Check services:

  ```bash
  ssh azureuser@<linux-public-ip>
  systemctl status nginx apache2 tomcat
  pm2 status
  ```

> **TLS (Let’s Encrypt)** (optional):

```bash
sudo apt-get install -y snapd
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
# Point your DNS to VM first, then:
sudo certbot --nginx -d your.domain.tld
```

---

# 3) Windows Server: PowerShell (IIS, .NET Core, Node, Tomcat)

## 3a) Upload & run the script via Custom Script Extension

Save as `win-web-setup.ps1`, then:

```bash
# Copy script to a storage account or Git raw URL. Example using GitHub raw:
SCRIPT_URI="https://raw.githubusercontent.com/<you>/scripts/main/win-web-setup.ps1"

az vm extension set \
  --resource-group $RG \
  --vm-name $WINVM \
  --publisher Microsoft.Compute \
  --name CustomScriptExtension \
  --version 1.10 \
  --settings "{'fileUris': ['$SCRIPT_URI']}" \
  --protected-settings "{'commandToExecute': 'powershell -ExecutionPolicy Bypass -File win-web-setup.ps1'}"
```

## 3b) `win-web-setup.ps1`

```powershell
# Install IIS
Install-WindowsFeature Web-Server -IncludeManagementTools

# Simple IIS landing page
"<!doctype html><h1>IIS on Windows Server is running</h1>" | Out-File -Encoding utf8 C:\inetpub\wwwroot\index.html

# Enable key IIS features (static, default doc, compression, logging)
Install-WindowsFeature Web-Http-Errors,Web-Static-Content,Web-Default-Doc,Web-Http-Logging,Web-Http-Redirect,Web-Request-Monitor

# .NET Hosting Bundle (for ASP.NET Core apps)
$dotnetUrl = "https://download.visualstudio.microsoft.com/download/pr/9b2a7fdb-7e7b-4e17-8c2e-0b0cdbd61c40/5b9d8c7b6b2b1f27fcb0d6d1b53b86a0/dotnet-hosting-8.0.7-win.exe"
$dotnetExe = "$env:TEMP\dotnet-hosting.exe"
Invoke-WebRequest $dotnetUrl -OutFile $dotnetExe
Start-Process $dotnetExe -ArgumentList "/quiet" -Wait

# Install Node.js (LTS) silently
$nodeUrl = "https://nodejs.org/dist/v20.16.0/node-v20.16.0-x64.msi"
$nodeMsi = "$env:TEMP\node.msi"
Invoke-WebRequest $nodeUrl -OutFile $nodeMsi
Start-Process msiexec.exe -ArgumentList "/i `"$nodeMsi`" /qn" -Wait
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine")

# Create a simple Node app and run it as a Windows service with NSSM
$nodeDir = "C:\node-demo"
New-Item -ItemType Directory -Force -Path $nodeDir | Out-Null
@"
const http = require('http');
const port = 3000;
http.createServer((req,res)=>{res.end('Node.js on Windows via NSSM\n');}).listen(port);
"@ | Out-File -FilePath "$nodeDir\server.js" -Encoding utf8

# Install NSSM (Non-Sucking Service Manager)
$nssmZip = "$env:TEMP\nssm.zip"
Invoke-WebRequest "https://nssm.cc/release/nssm-2.24.zip" -OutFile $nssmZip
Expand-Archive $nssmZip -DestinationPath $env:TEMP -Force
Copy-Item "$env:TEMP\nssm-2.24\win64\nssm.exe" "C:\Windows\System32\" -Force

nssm install node-demo "C:\Program Files\nodejs\node.exe" "C:\node-demo\server.js"
nssm set node-demo Start SERVICE_AUTO_START
nssm start node-demo

# Install Tomcat 10 as a service
$tomcatUrl = "https://downloads.apache.org/tomcat/tomcat-10/v10.1.29/bin/apache-tomcat-10.1.29-windows-x64.zip"
$tomcatZip = "$env:TEMP\tomcat.zip"
$tomcatDir = "C:\Tomcat"
Invoke-WebRequest $tomcatUrl -OutFile $tomcatZip
Expand-Archive $tomcatZip -DestinationPath $tomcatDir -Force
# The expanded folder name may vary; move contents up one level if needed:
$nested = Get-ChildItem $tomcatDir | Where-Object {$_.PSIsContainer} | Select-Object -First 1
if ($nested) { Move-Item "$($nested.FullName)\*" $tomcatDir -Force; Remove-Item $nested.FullName -Recurse -Force }

& "$tomcatDir\bin\service.bat" install Tomcat10
Start-Service Tomcat10
Set-Service Tomcat10 -StartupType Automatic

# Optional: IIS reverse proxy for Node/Tomcat via URL Rewrite + ARR
# Install Web Platform Installer cmdline to fetch URL Rewrite/ARR (if desired)
# (Skipping here for brevity; most use cases: hit Node at :3000, Tomcat at :8080)

Write-Host "IIS, Node (service), and Tomcat are ready."
```

### Test (Windows)

* IIS: `http://<windows-public-ip>/`
* Node app: `http://<windows-public-ip>:3000/`
* Tomcat: `http://<windows-public-ip>:8080/`

> **TLS on Windows (optional):** use [win-acme](https://www.win-acme.com/) (ACME client) to auto-fetch/renew Let’s Encrypt certs and bind to IIS.

---

# 4) Pick-and-Choose Deployment Matrix

| Stack          | Linux (Ubuntu)                                             | Windows Server                              |
| -------------- | ---------------------------------------------------------- | ------------------------------------------- |
| Static/Reverse | **Nginx** (default site; reverse proxy `/node`, `/tomcat`) | **IIS** (Default Web Site)                  |
| PHP            | `apt install php-fpm` + Nginx `fastcgi_pass`               | IIS + PHP Manager (Web PI)                  |
| Java           | Tomcat 10 service (systemd)                                | Tomcat 10 service (`service.bat`)           |
| Node.js        | Node 20 + **PM2**                                          | Node 20 + **NSSM**                          |
| .NET           | (Container or Kestrel behind Nginx)                        | **.NET Hosting Bundle** + IIS reverse proxy |
| TLS            | **certbot --nginx**                                        | **win-acme** for IIS                        |

---

# 5) Quick Ops Commands

**Linux**

```bash
# Nginx
sudo nginx -t && sudo systemctl reload nginx
sudo tail -n 100 /var/log/nginx/error.log

# Apache
sudo systemctl status apache2
sudo tail -n 100 /var/log/apache2/error.log

# Node (PM2)
pm2 status
pm2 logs node-demo

# Tomcat
sudo journalctl -u tomcat -e --no-pager
sudo tail -n 200 /opt/tomcat/logs/catalina.out
```

**Windows (PowerShell)**

```powershell
Get-Service W3SVC, Tomcat10, node-demo
Get-Content -Tail 100 C:\inetpub\logs\LogFiles\W3SVC1\u_ex*.log
Get-Content -Tail 200 C:\Tomcat\logs\catalina.out
nssm status node-demo
```

---

# 6) Optional: Terraform skeleton (1 Linux + 1 Windows)

If you want this IaC-ready, say the word and I’ll drop a minimal Terraform with:

* `azurerm_linux_virtual_machine` + `custom_data` (cloud-init)
* `azurerm_windows_virtual_machine` + Custom Script Extension
* NSG, NICs, public IPs, and variables/outputs

---

# 7) Common Pitfalls & Fixes

* **Port blocked** → Confirm NSG *and* OS firewall (UFW/Windows Defender Firewall).
* **DNS/TLS** → Point A record to VM IP before running ACME clients (Let’s Encrypt).
* **Tomcat not starting** → Check Java version (OpenJDK 17 for Tomcat 10), logs.
* **Node not persistent** → Ensure PM2 `pm2 save` (Linux) or NSSM (Windows service).
* **IIS 500 errors** → Install the right ASP.NET feature/Hosting Bundle; check `Event Viewer → Windows Logs → Application`.
