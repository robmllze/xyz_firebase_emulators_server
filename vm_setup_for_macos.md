# VM Setup for MacOS

## How to se up a VM

1. Create a new VM instance [here](https://console.cloud.google.com/compute/instances). Be sure to tick `Allow HTTP` and `Allow HTTPS` and select correct service account.
2. Login with gcloud: `gcloud auth login`
3. Generate an SSK key-pair: `ssh-keygen -t rsa -f ~/.ssh/google_compute_engine && cat ~/.ssh/google_compute_engine.pub`
4. Open the file `~/.ssh/google_compute_engine.pub` and copy its contents.
5. Add the SSH key [here](https://console.cloud.google.com/compute/metadata?tab=sshkeys).
6. Make the internal and external IP addesses static [here](https://console.cloud.google.com/networking/addresses/l).
4. Open your VM's terminal via: `gcloud compute ssh <VM_NAME> --zone <ZONE> --project <FIREBASE_PROJECT_NAME>`, e.g. `gcloud compute ssh test-firebase-au --zone australia-southeast2-a --project xyz-app-example`
5. Install necessary packages:
```sh
# Update and upgrade existing packages.
sudo apt update
sudo apt upgrade
# Install essentials.
sudo apt install openjdk-11-jdk && java -version
sudo apt install nodejs && sudo apt install npm
sudo apt install nginx
sudo apt install certbot python3-certbot-nginx
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb && sudo dpkg -i google-chrome-stable_current_amd64.deb
# Install NVM. See: https://github.com/nvm-sh/nvm.
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash && source ~/.bashrc
# Install Node.js 16.
nvm install 16 && nvm use 16
sudo npm install -g firebase-tools
# Download an install the Chrome Remote Desktop package.
wget https://dl.google.com/linux/direct/chrome-remote-desktop_current_amd64.deb
sudo dpkg -i chrome-remote-desktop_current_amd64.deb
sudo apt install -f
sudo dpkg-reconfigure chrome-remote-desktop
```

### Notes

- Run Chrome with `google-chrome-stable`
- Fix broken dependencies with `apt --fix-broken install`
- Get firewall status with `sudo ufw status` (must install `sudo apt install ufw`)
- Disable firewall with `sudo ufw disable` (must install `sudo apt install ufw`)

## How to use a custom domain for your server

1. Register a domain at [https://namecheap.com/], for example `robmllze0.xyz`. Be sure to add Premium DNS.
1. Create two `A` records with hosts `@` and `wwww` and values `34.129.18.22` or whatever your server's public IP is. Delete all other records.
1. Generate a free SSL certificate using "Let's Encrypt" (see: [https://letsencrypt.org/]) for your domains to allow serving an HTTPS server.
```sh
# Change robmllze0.xyz and www.robmllze0.xyz to your domains.
sudo certbot certonly --standalone -d robmllze0.xyz -d www.robmllze0.xyz
```
2. Specify an email, e.g. `robmllze@gmail.com`
3. This should generate two files, e.g. `/etc/letsencrypt/live/robmllze0.xyz/fullchain.pem` and `/etc/letsencrypt/live/robmllze0.xyz/privkey.pem`.
4. Create a conf file for nginx with `sudo nano /etc/nginx/conf.d/robmllze0.conf` and write the following (modify as needed):
```conf
# Define an upstream block for the backend service on port 4100.
upstream backend_4100 {
    server localhost:4100;
    keepalive 32;
}

# Define an upstream block for the backend service on port 8180.
upstream backend_8180 {
    server localhost:8180;
    keepalive 32;
}

# Define an upstream block for the backend service on port 9199.
upstream backend_9199 {
    server localhost:9199;
    keepalive 32;
}

# Server block for HTTPS on port 4000.
server {
    listen 4000 ssl;
    server_name robmllze0.xyz;

    # SSL certificate and key paths.
    ssl_certificate /etc/letsencrypt/live/robmllze0.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/robmllze0.xyz/privkey.pem;

    # SSL configuration.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';

    # Location block for handling requests.
    location / {
        # Proxy requests to backend service on port 4100.
        proxy_pass http://backend_4100;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Server block for HTTPS on port 8080.
server {
    listen 8080 ssl;
    server_name robmllze0.xyz;

    # SSL certificate and key paths.
    ssl_certificate /etc/letsencrypt/live/robmllze0.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/robmllze0.xyz/privkey.pem;

    # SSL configuration.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';

    # Location block for handling requests.
    location / {
        # Proxy requests to backend service on port 8180.
        proxy_pass http://backend_8180;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Server block for HTTPS on port 9099.
server {
    listen 9099 ssl;
    server_name robmllze0.xyz;

    # SSL certificate and key paths.
    ssl_certificate /etc/letsencrypt/live/robmllze0.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/robmllze0.xyz/privkey.pem;

    # SSL configuration.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';

    # Location block for handling requests.
    location / {
        # Proxy requests to backend service on port 9199.
        proxy_pass http://backend_9199;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
5. Save the file then test nginx for errors with `sudo nginx -t`.
6. Test that certbot is able to automatically renew the certificates when needed (every 90 days) `sudo certbot renew --dry-run`.
7. Start or reload nginx `sudo systemctl start nginx` or `sudo systemctl reload nginx` (stop with `sudo systemctl stop nginx`).

## How to set up a remote Firebase Emulator on the VM

1. Create a VM and open its console.
2. Ensure you have Node.js 16 and Firebase Tools installed.
3. Create an Ingress firewall rule [here](https://console.cloud.google.com/net-security/firewall-manager/) to allow the ports used by Firestore Emulator (TCP 4000, 4100, 8080, 8180, 9099, 9199, etc.)
4. Enable port forwarding for all ports used by Firestore Emulator, for example:
```sh
gcloud compute ssh firebase-emulator -- -L 4000:localhost:4000
gcloud compute ssh firebase-emulator -- -L 8080:localhost:8080
gcloud compute ssh firebase-emulator -- -L 9099:localhost:9099
gcloud compute ssh firebase-emulator -- -L 4100:localhost:4100
gcloud compute ssh firebase-emulator -- -L 8180:localhost:8180
gcloud compute ssh firebase-emulator -- -L 9199:localhost:9199
# ...
```
5. Create the Firebase project.
```sh
rm -r firebase_emulator
mkdir firebase_emulator && cd firebase_emulator
firebase init
```
6. Run `nano firebase.json` to edit the file:
```json
{
  "emulators": {
    "auth": {
      "port": 9099,
      "host": "0.0.0.0"
    },
    "firestore": {
      "port": 8080,
      "host": "0.0.0.0"
    },
    "ui": {
     "enabled": true,
      "port": 4000,
      "host": "0.0.0.0"
    },
    "singleProjectMode": true
  }
}
```
7. Start the emulators with `firebase emulators:start`
8. Access the emulator dashboard with the VM's public IP, like [http://www.robmllze0.xyz:4100/] or [robmllze0.xyz:4000/].
9. Before running the app in Chrome, you must set the site to allow mixed content.
  - On Chrome on desktop or Android, go to `chrome://flags/#unsafely-treat-insecure-origin-as-secure`. Enable the flag `unsafely-treat-insecure-origin-as-secure` then add all the ports used by the app and emulator, e.g. `http://robmllze0.xyz:8180,http://robmllze0.xyz:9199`.
  - On Chrome
  - Another option on Chrome on desktop, is to go to `chrome://settings/content/all` then select the target such as `xyz-app-example--preview-s1b8b4ab.web.app` and set `Insecure content` to `true`. This will allow mixed http/https content for the target only.
  - TODO: Not sure how to do this on iOS.

### References

- [https://firebase.google.com/docs/emulator-suite/install_and_configure]

## Remote desktop

1. Create a VM and open its console.
2. Go [here](https://remotedesktop.google.com/headless) then `Begin` -> `Next` -> `Authorize` and get the setup code for Debian (Linux). It should look something like this.
```sh
DISPLAY= /opt/google/chrome-remote-desktop/start-host --code="<YOUR CODE HERE>" --redirect-url="https://remotedesktop.google.com/_/oauthredirect" --name=$(hostname)
```
4. Run this code on your VM and set a PIN.
5. Go [here](https://remotedesktop.google.com/access) and access the VM with the PIN.

### References

- [https://cloud.google.com/architecture/chrome-desktop-remote-on-compute-engine]