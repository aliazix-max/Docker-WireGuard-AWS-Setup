# Docker WireGuard Setup

This repository contains the setup for running WireGuard VPN in a Docker container with dynamic IP updating using No-IP and iptables on an AWS EC2 instance.

## Prerequisites
- AWS EC2 instance (This has been tested on Amazon Linux 2)
- Docker and Docker Compose installed
- No-IP account for dynamic DNS

## Setup Instructions

### Step 1: Install Docker on AWS EC2

1. **Update the package list and install Docker**:
    ```sh
    sudo yum update -y
    sudo amazon-linux-extras install docker -y
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

2. **Verify Docker installation**:
    ```sh
    docker --version
    ```

### Step 2: Setup WireGuard in Docker

1. **Clone this repository and navigate to the WireGuard directory**:
    ```sh
    git clone https://github.com/username/docker-wireguard-setup.git
    cd docker-wireguard-setup/wireguard
    ```

2. **Create a `docker-compose.yml` file with the following content**:
    ```yaml
    version: '3'

    services:
      wireguard:
        image: linuxserver/wireguard
        container_name: wireguard
        cap_add:
          - NET_ADMIN
          - SYS_MODULE
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Etc/UTC
          - SERVERURL=myvpn.ddns.net # Replace with your No-IP hostname
          - SERVERPORT=51820
          - PEERS=1
          - PEERDNS=auto
        volumes:
          - ./config:/config
        ports:
          - 51820:51820/udp
        sysctls:
          - net.ipv4.conf.all.src_valid_mark=1
        restart: unless-stopped
    ```

3. **Start the WireGuard container**:
    ```sh
    sudo docker-compose up -d
    ```

4. **Verify that the container is running**:
    ```sh
    sudo docker ps
    ```

### Step 3: Configure Dynamic DNS (No-IP)

1. **Register for a No-IP account and create a hostname**:
    - Go to [No-IP](https://www.noip.com/) and sign up for an account.
    - Create a hostname (e.g., `myvpn.ddns.net`).

2. **Install the No-IP client on your EC2 instance**:
    ```sh
    cd /usr/local/src/
    sudo wget https://www.noip.com/client/linux/noip-duc-linux.tar.gz
    sudo tar xf noip-duc-linux.tar.gz
    cd noip-2.1.9-1/
    sudo make
    sudo make install
    ```

3. **Configure the No-IP client**:
    ```sh
    sudo /usr/local/bin/noip2 -C
    ```

4. **Start the No-IP client**:
    ```sh
    sudo /usr/local/bin/noip2
    ```

### Step 4: Script to Update iptables

1. **Create the script file**:
    ```sh
    sudo nano /usr/local/bin/update-iptables.sh
    ```

2. **Add the following content to the script**:
    ```sh
    #!/bin/bash
    echo "Script started at $(date)" >> /var/log/update-iptables-cron.log

    HOSTNAME="myvpn.ddns.net"
    IPTABLES=/sbin/iptables
    IP=$(dig +short $HOSTNAME)

    # Remove old iptables rule (if any)
    if [ -f /tmp/current_ip ]; then
        OLD_IP=$(cat /tmp/current_ip)
        $IPTABLES -D INPUT -p udp --dport 51820 -s $OLD_IP -j ACCEPT
        logger "Removed old iptables rule for IP: $OLD_IP"
        echo "Removed old iptables rule for IP: $OLD_IP" >> /var/log/update-iptables-cron.log
    fi

    # Add new iptables rule
    $IPTABLES -A INPUT -p udp --dport 51820 -s $IP -j ACCEPT
    logger "Added new iptables rule for IP: $IP"
    echo "Added new iptables rule for IP: $IP" >> /var/log/update-iptables-cron.log

    # Save new IP to file for future use
    echo $IP > /tmp/current_ip
    echo "Script completed at $(date)" >> /var/log/update-iptables-cron.log
    ```

3. **Make the script executable**:
    ```sh
    sudo chmod +x /usr/local/bin/update-iptables.sh
    ```

### Step 5: Setup Crontab

1. **Edit the crontab for root**:
    ```sh
    sudo crontab -e
    ```

2. **Add the following line to the crontab to run the script every 5 minutes**:
    ```sh
    */5 * * * * /usr/local/bin/update-iptables.sh >> /var/log/update-iptables-cron.log 2>&1
    ```

3. **Save and exit the crontab editor**.

### Verify the Setup

1. **Check the log file for script execution details**:
    ```sh
    sudo cat /var/log/update-iptables-cron.log
    ```

2. **Ensure the iptables rules are updated**:
    ```sh
    sudo iptables -L -v -n
    ```

3. **Verify VPN connectivity**:
    - Ensure that your VPN client can connect using the No-IP hostname.

## License

MIT License
