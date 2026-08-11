Build Your Own Home Cloud



Part 1 – Prepare your Linux

Step 1 – Update the system 
Open a terminal or SSH into your machine and run:
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y

Step 2 – Install Essential Tools 
sudo apt install -y curl wget git htop net-tools ufw

Part 2 – Install Docker

Step 3– Install Docker 
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
 
Step 4– Add your user to the Docker group
sudo usermod -aG docker $USER
newgrp docker
 
Step 5– Verify the installation
docker –-version
docker compose version
Step 6– Configure Firewall and Open ports for Web traffic
   
sudo ufw enable
Type y when asked to confirm. Verify the firewall is active:
sudo ufw status

sudo ufw allow 80
sudo ufw allow 443
sudo ufw status




Part 3 – Install Nextcloud
Step 7– Create folder structure 
mkdir -p ~/homecloud/caddy
mkdir -p ~/homecloud/nextcloud
cd ~/homecloud

Step 8– Create the Docker Compose file
cat > ~/homecloud/docker-compose.yml << 'EOF'
services:
 
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
 
volumes:
  nextcloud-db:
  nextcloud-data:
  caddy-data:
  caddy-config:
EOF

 
Step 9– Start the containers
docker compose up -d
This pulls the images and starts Nextcloud, MariaDB, and Caddy. The first run takes a few minutes. Verify everything is running:
docker compose ps
All three containers — nextcloud, nextcloud-db, and caddy — should show a status of Up.

At this point, we should be able to access our Nextcloud locally. We will finish setting up the domain so we can access it from anywhere at any time. 




Part 4 – Set up DuckDNS (Free domain)
Step 10– Create a DuckDNS Account 
• Go to https://www.duckdns.org
• Sign in with Google or GitHub
• Choose a subdomain name — for example: yourhome.duckdns.org
• Click Add Domain
• Copy your token — you will need it in the next step

Step 11– Create the DuckDNS Update Script  
Replace YOUR_TOKEN and YOUR_SUBDOMAIN with your actual values:
mkdir -p ~/duckdns
cat > ~/duckdns/duck.sh << 'EOF'
echo url="https://www.duckdns.org/update?domains=YOUR_SUBDOMAIN&token=YOUR_TOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
EOF
chmod 700 ~/duckdns/duck.sh
Test that it works:
~/duckdns/duck.sh
cat ~/duckdns/duck.log
It should print OK. If it does, the domain is pointing to your public IP.


Step 12– Schedule automatic updates   
This keeps your domain updated every 5 minutes in case your home IP changes:
EDITOR=nano crontab -e
Add this line at the bottom of the file:
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
Save with Ctrl+O, Enter, Ctrl+X. Verify it saved:
crontab -l
I had an issue when adding the line for this part and I had used another command. I will leave it below, just in case you run into the same issue. It was just not saving/detecting the line once I added it. 
(crontab -l 2>/dev/null; echo "*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1") | crontab -



Step 13– Create caddy file 
Replace YOUR_DOMAIN with your actual DuckDNS domain (set up in Part 4):
cat > ~/homecloud/caddy/Caddyfile << 'EOF'
YOUR_DOMAIN.duckdns.org {
    reverse_proxy nextcloud:80
}
EOF
In the video, I update the Docker Compose file again to add Caddy. When we set up earlier, we did it all, so you don’t need to update it again. 

Step 14– Restart/start containers 
docker compose down
docker compose up -d
This pulls the images and starts Nextcloud, MariaDB, and Caddy. The first run takes a few minutes. Verify everything is running:
docker compose ps
All three containers — nextcloud, nextcloud-db, and caddy — should show a status of Up.


I had an issue here where I had “Nextcloud” before my DNS address. I show in the video how to fix it, but make sure that you have the correct DNS. 
 
Part 5– Final configuration
Step 15– Add your domain as Trusted in NextCloud
Replace YOUR_DOMAIN with your actual DuckDNS subdomain:
docker exec -it nextcloud php occ config:system:set trusted_domains 1 --value=YOUR_LOCAL_IP
docker exec -it nextcloud php occ config:system:set trusted_domains 2 --value=YOUR_DOMAIN.duckdns.org

Part 6– Router Port Forwarding
Step 16– Forward your ports in your router
You need to forward ports 80 and 443 to your server's local IP address. The steps vary by router brand:

I have an Eero router, so I did this using the app on my phone: 
• Open the Eero app
• Tap the menu icon (☰) top left
• Tap Network settings
• Tap Reservations & port forwarding
• Tap your server's IP reservation
• Tap Add port forward
• Add rule 1: Name = HTTP, Internal port = 80, External port = 80, Protocol = TCP
• Add rule 2: Name = HTTPS, Internal port = 443, External port = 443, Protocol = TCP
Most other routers: 
• Open a browser and go to 192.168.1.1 or 192.168.0.1
• Log in with your router admin credentials
• Find the Port Forwarding section
• Add the same two rules pointing to your server's local IP

💡 If you have set up a DHCP reservation for your server's IP address on your router, port forwarding will always work even after reboots.

Part 7– Access NextCloud
Step 17– Access NextCloud
Open a browser from anywhere in the world and go to:
https://YOUR_DOMAIN.duckdns.org
Log in with the admin username and password you set in the Docker Compose file. Caddy automatically handles HTTPS — the first time it may take 1-2 minutes to get the certificate from Let's Encrypt.
💡 The HTTPS certificate from Let's Encrypt is completely free and renews automatically. You never need to manage it manually. You will also see a “SUCCESS” part when you go back to DUCKDNS.





Troubleshooting
Access through an untrusted domain error
Run the trusted domain command from Step 15 with your correct IP or domain name.

Caddy container keeps restarting 
Check the Caddy logs for errors:
docker logs caddy
Most common cause: the Caddyfile has a typo in the domain name. Verify it with:
cat ~/homecloud/caddy/Caddyfile
The domain must be plain, no http:// or https:// prefix. Just the domain name followed by the block.

“This site can’t be reached” from outside
• Check that duck.log says OK after running the script
• Verify your public IP matches what DuckDNS shows on their website
• Test ports 80 and 443 are open at yougetsignal.com/tools/open-ports

