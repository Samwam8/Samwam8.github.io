---
layout: subpage
title: Docker Application Setup Guide Using Uptime Kuma
---
# Docker Application Setup Guide Using Uptime Kuma

## 1. Introduction

This guide goes over the installation and configuration of Docker and Uptime Kuma on an Ubuntu Server.

Uptime Kuma is a self-hosted monitoring application.

## 2. Installing Docker

### 2.1 Install Dependencies

First, update the package lists and install the required dependencies.

```bash
sudo apt update && sudo apt install -y ca-certificates curl
```
### 2.2 Configure the Docker Repository

Create a directory for storing repository signing keys.
```bash
sudo install -m 0755 -d /etc/apt/keyrings
```
Download Docker's official GPG signing key.
```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```
Add the Docker repository to the system's package sources.
```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```
This command automatically identifies the system architecture and Ubuntu release codename to configure the appropriate Docker repository.

Update the package lists again to include the new repository.
```bash
sudo apt update
```
### 2.3 Install Docker

Install Docker Engine, its command-line interface, containerd, and Docker Compose.
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Enable Docker to start automatically and start the service immediately.
```bash
sudo systemctl enable --now docker
```
Verify that Docker is enabled.
```bash
systemctl is-enabled docker
```
The expected output is:
```bash
enabled
```
Test the Docker installation by running the Hello World container.
```bash
sudo docker run --rm hello-world
```
The successful execution of this confirms that Docker is correctly installed.

Verify the Docker Compose installation.
```bash
docker compose version
```
### 2.4 Optional: Run Docker Without sudo

Add the current user to the Docker group.
```bash
sudo usermod -aG docker $USER
```
This allows the user to run Docker commands without using sudo.

Log out and back in for the group membership change to take effect.
## 3. Installing Uptime Kuma

### 3.1 Create the Application Directory

Create a directory to store the Uptime Kuma configuration and application data.
```bash
sudo mkdir -p /opt/uptime-kuma
```
Give the current user ownership of the directory.
```bash
sudo chown -R $USER:$USER /opt/uptime-kuma
```
Navigate to the application directory.
```bash
cd /opt/uptime-kuma
```
### 3.2 Configure Docker Compose

Create the Docker Compose configuration file.
```bash
nano compose.yaml
```
Add the following configuration:
```bash
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "127.0.0.1:3001:3001"
    volumes:
      - ./uptime-kuma-data:/app/data
```
This uses the official uptime-kuma image, makes a container called uptime-kuma, sets the application to automatically restart unless stopped manually, maps the port 3001 only to localhost, and specifies a volume to store persistent application data

The persistent volume ensures that application data remains available even if the container is removed or recreated.

Binding the application to 127.0.0.1 prevents direct access through the VM's external network interfaces. Access is provided through an SSH tunnel instead.

### 3.3 Deploy Uptime Kuma

Start the application in the background.
```bash
docker compose up -d
```
Docker downloads the required image if it is not already available and creates the container.

Check the container status.
```bash
docker compose ps
```
This displays the running containers and their current status.

View application logs.
```bash
docker compose logs -f
```
This displays live application logs. Press Ctrl+C to stop viewing the logs without stopping the container.
### 3.4 Access Uptime Kuma Through SSH
Open Git Bash and run:
```bash
ssh -i ~/.ssh/id_ed25519 -L 3001:127.0.0.1:3001 $USER@YOUR_SERVER_IP
```
The -i option specifies the private SSH key, while -L forwards local port 3001 to port 3001 on the Ubuntu server.

Open the following address in a web browser:

[http://localhost:3001](http://localhost:3001)

Create an administrator account to finish the initial Uptime Kuma setup.

## 4. Securing the Ubuntu VM

### 4.1 SSH Hardening

SSH key authentication is used instead of password authentication to improve remote access security.

Create a separate SSH configuration file.
```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf

```
Add the following settings:
```bash
PubkeyAuthentication yes
PasswordAuthentication no
PermitRootLogin no
```
These settings:

- Enable public key authentication.
    
- Disable password-based SSH authentication.
    
- Prevent direct SSH login as root.
    

Validate the SSH configuration.
```bash
sudo sshd -t
```

This checks for configuration errors. No output indicates that the configuration passed validation.

Check the effective SSH settings.
```bash
sudo sshd -T | grep -E 'pubkeyauthentication|passwordauthentication|permitrootlogin'
```
Output should be same as what you entered above.

Reload the SSH service to apply the changes.
```bash
sudo systemctl reload ssh
```
Test that password authentication is refused.
```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no $USER@YOUR_SERVER_IP
```
The connection should be refused because password authentication has been disabled.

Do not close your current SSH until you confirm the login works.

### 4.2 Configure UFW Firewall

Install UFW, Ubuntu's uncomplicated firewall.
```bash
sudo apt install ufw -y
```
Set the default incoming and outgoing firewall policies.
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Blocks unsolicited incoming connections.

Allows outgoing connections, including application monitoring requests and software updates.

Allow SSH connections.
```bash
sudo ufw allow OpenSSH
```
This ensures that remote administration remains possible through SSH.

Enable the firewall.
```bash
sudo ufw enable
```
Check the firewall configuration.
```bash
sudo ufw status verbose
```
The expected configuration includes:

- Firewall status: Active.
    
- Incoming connections: Denied by default.
    
- Outgoing connections: Allowed by default.
    
- OpenSSH: Allowed.
    

Uptime Kuma does not require an additional public firewall rule for port 3001 because Docker is configured to bind the application to 127.0.0.1.
### 4.3 Configure Automatic Security Updates
Install unattended-upgrades.
```bash
sudo apt install unattended-upgrades -y
```
This package automatically installs eligible updates, including security updates.

Run the configuration utility.
```bash
sudo dpkg-reconfigure unattended-upgrades
```
Select Yes when prompted to enable automatic updates.

Open the automatic updates configuration file.
```bash
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```
Ensure the following settings are present:
```bash
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```
These settings configure daily package-list updates and automatic installation of eligible unattended upgrades.

Check the service status.
```bash
systemctl status unattended-upgrades
```
Test the automatic update process without installing updates.
```bash
sudo unattended-upgrade --dry-run --debug
```
This simulates the update process and displays debugging information.

## 5. Configuring Application Backups
A backup script is used to create compressed archives of Uptime Kuma's persistent application data.

The script stops the application before backing up its data to avoid copying an actively changing database. It then restarts the application and retains only the seven newest archives.
### 5.1 Create the Backup Directory
Create a directory for storing backup archives.
```bash
sudo mkdir -p /var/backups/uptime-kuma
```
### 5.2 Create the Backup Script
Create the backup script.
```bash
sudo nano /usr/local/bin/backup-uptime-kuma.sh
```
Add the following contents:
```bash
#!/bin/bash

set -Eeuo pipefail

APP_DIR="/opt/uptime-kuma"
BACKUP_DIR="/var/backups/uptime-kuma"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S-%N")
ARCHIVE="$BACKUP_DIR/uptime-kuma-$TIMESTAMP.tar.gz"

mkdir -p "$BACKUP_DIR"

docker compose -f "$APP_DIR/compose.yaml" stop uptime-kuma

restart_app() {
    docker compose -f "$APP_DIR/compose.yaml" start uptime-kuma
}
trap restart_app EXIT

tar -czf "$ARCHIVE" -C "$APP_DIR" uptime-kuma-data

find "$BACKUP_DIR" -maxdepth 1 -type f \
    -name 'uptime-kuma-*.tar.gz' \
    -printf '%f\n' | sort -r | tail -n +8 |
    while IFS= read -r file; do
        rm -f -- "$BACKUP_DIR/$file"
    done

echo "Backup completed: $ARCHIVE"
```
The script performs the following operations:

1. Defines the application and backup directories.
    
2. Generates a timestamp for the backup filename.
    
3. Stops the Uptime Kuma container.
    
4. Registers a command to restart the application when the script exits.
    
5. Creates a compressed archive of the persistent application data.
    
6. Identifies older backup archives and removes everything except the seven newest.
    
7. Displays the backup location when finished.
    
### 5.3 Make the Script Executable
Allow the script to execute.
```bash
sudo chmod +x /usr/local/bin/backup-uptime-kuma.sh
```
### 5.4 Execute and Verify Backups
Run the backup script.
```bash
sudo /usr/local/bin/backup-uptime-kuma.sh
```
Run it a second time to create another backup archive.

List the backup directory and verify that the archives were created.
```bash
sudo ls -lh /var/backups/uptime-kuma/
```
The directory should contain at least two compressed .tar.gz archives.

Verify that Uptime Kuma is running after the backup operations.
```bash
sudo docker compose -f /opt/uptime-kuma/compose.yaml ps
```
The container should show a running status.
## 6. Conclusion

Docker and Uptime Kuma have been installed and configured on a Ubuntu Server. Uptime Kuma runs in a Docker container with persistent application storage and is accessed remotely through an SSH tunnel. Additional security measures include SSH key authentication, disabled password authentication, UFW firewall policies, and automatic security updates. An application backup script has also been configured to create compressed backups of Uptime Kuma's persistent data, restart the application after backups, and retain the seven newest archives.
## 7. Sources

- [Docker Engine Installation on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
    
- [Docker Compose Documentation](https://docs.docker.com/compose/)
    
- [Uptime Kuma GitHub Repository](https://github.com/louislam/uptime-kuma)
    
- [Ubuntu Automatic Updates Documentation](https://documentation.ubuntu.com/server/how-to/software/automatic-updates/)
    
- [Ubuntu Firewall Documentation](https://documentation.ubuntu.com/server/how-to/security/firewalls/)
    
- [Ubuntu OpenSSH Server Documentation](https://documentation.ubuntu.com/server/how-to/security/openssh-server/)
    
- [Proxmox VE Documentation](https://pve.proxmox.com/pve-docs/)
    
