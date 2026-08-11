AdGuard home setup

Part 1 – Create the folder structure
mkdir -p ~/homecloud/adguard/{work,conf}

Part 2 – Open firewall ports
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 3000/tcp
sudo ufw status

Part 3 – Check if port 53 is already in use
Ubuntu uses a built-in DNS service, systemd-resolved, which by default occupies port 53. AdGuard
needs that port. Check if it is running:
sudo lsof -i :53  

If you see systemd-r in the output, disable it:
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf 


Part 4 – Update Docker Compose file
Use the nano command to update the file. The green-highlighted lines are the new AdGuard block
being added. Replace all changeme_* values with your actual passwords:

nano ~/homecloud/docker-compose.yml
nextcloud-db:
image: mariadb:10.11
container_name: nextcloud-db
restart: unless-stopped
environment:
MYSQL_ROOT_PASSWORD: changeme_root
MYSQL_DATABASE: nextcloud
MYSQL_USER: nextcloud
MYSQL_PASSWORD: changeme_nc
volumes:
- nextcloud-db:/var/lib/mysql
nextcloud:
image: nextcloud:latest
container_name: nextcloud
restart: unless-stopped
depends_on:
- nextcloud-db
environment:
MYSQL_HOST: nextcloud-db
MYSQL_DATABASE: nextcloud
MYSQL_USER: nextcloud
MYSQL_PASSWORD: changeme_nc
NEXTCLOUD_ADMIN_USER: admin
NEXTCLOUD_ADMIN_PASSWORD: changeme_admin
volumes:
- nextcloud-data:/var/www/html
caddy:
image: caddy:latest
container_name: caddy
restart: unless-stopped
ports:
- "80:80"
- "443:443"
volumes:
- ~/homecloud/caddy/Caddyfile:/etc/caddy/Caddyfile
- caddy-data:/data
- caddy-config:/config

# NEW — AdGuard Home 
adguard:
image: adguard/adguardhome:latest
container_name: adguard
restart: unless-stopped
ports:
- "53:53/tcp"
- "53:53/udp"
- "3000:3000/tcp"
volumes:
- ~/homecloud/adguard/work:/opt/adguardhome/work
- ~/homecloud/adguard/conf:/opt/adguardhome/conf
# 

volumes:
nextcloud-db:
nextcloud-data:
caddy-data:
caddy-config:
EOF

Part 5 – Start and configure AdGuard
Restart all containers: 
cd ~/homecloud
docker compose down
docker compose up -d
docker compose ps 

Part 6 – Run the Adguard setup wizard
Open a browser and go to: 
http://”YOUR_IP_ADDRESS”:3000 

Walk through the setup wizard:
• Click get started
• Leave the web interface port as 3000
• Leave the DNS server port as 53
• Create your admin username and password
• Click Next, then open the dashboard

After setting up, your AdGuard dashboard will always be at:
http://”YOUR_IP_ADDRESS”:3000 

Part 7 – Add DNS block list
In the dashboard, go to Filters → DNS Blocklists → Add Blocklist → Choose from the list.
AdGuard DNS filter
OISD Blocklist Small
Steven Black's List
Peter Lowe's Blocklist
Anti Push Notifications
Apple Tracker Blocklist
Windows/Office Tracker
Phishing URL Blocklist
Anti-Malware List
Malicious URL (URLHaus)
DNS Rebind Protection

Part 8 – Connect to your network
You can first verify by using your phone, and then change the DNS address. Always test on a single device before changing your router's DNS. If something goes wrong, you can revert just that one device without affecting anyone else connected to the Wi-Fi. 
Once you test, you can point your router to your DNS (Your_ip_address) and add Google’s DNS as a secondary DNS (8.8.8.8) as a fallback, just in case your lab/server is unreachable. 


