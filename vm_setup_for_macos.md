# VM Setup for MacOS

## How to se up a VM

1. Create a new VM instance [here](https://console.cloud.google.com/compute/instances). Be sure to tick `Allow HTTP` and `Allow HTTPS` and select correct service account.
2. Login with gcloud: `gcloud auth login`
3. Option A:
  - Generate an SSK key-pair: `ssh-keygen -t rsa -f ~/.ssh/google_compute_engine && cat ~/.ssh/google_compute_engine.pub`
  - Open the file `~/.ssh/google_compute_engine.pub` and copy its contents.
  - Add the SSH key [here](https://console.cloud.google.com/compute/metadata?tab=sshkeys).
4. Option B:
```bash
# Generate an RSA SSH key pair. If the files already exist, you might be prompted to overwrite them.
ssh-keygen -t rsa -f ~/.ssh/google_compute_engine

# Prefix the generated public key with the username 'robmllze:' and save the output 
# to a new file named 'google_compute_engine_modified.pub'.
echo "robmllze:$(cat $HOME/.ssh/google_compute_engine.pub)" > $HOME/.ssh/google_compute_engine_modified.pub

# Upload the modified public key to your GCP project's metadata.
gcloud compute project-info add-metadata --metadata-from-file ssh-keys=$HOME/.ssh/google_compute_engine_modified.pub --project xyz-app-example
```
6. Make the internal and external IP addesses static [here](https://console.cloud.google.com/networking/addresses/).
4. Open your VM's terminal via: `gcloud compute ssh <VM_NAME> --zone <ZONE> --project <FIREBASE_PROJECT_NAME>`, e.g. `gcloud compute ssh robmllze0 --zone australia-southeast2-a --project xyz-app-example`
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
sudo apt install ufw
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
- Get firewall status with `sudo ufw status`
- Disable firewall with `sudo ufw disable`
- Kill a port with `kill -9 $(lsof -t -i :<PORT>)` (e.g. `kill -9 $(lsof -t -i :4001)`)

## How to use a custom domain for your server

1. Register a domain at [https://namecheap.com/], for example `robmllze0.xyz`. Be sure to add Premium DNS.
1. Create two `A` records with hosts `@` and `wwww` and values `34.129.18.22` or whatever your server's public IP is. Delete all other records.
1. Generate a free SSL certificate using "Let's Encrypt" (see: [https://letsencrypt.org/]) for your domains to allow serving an HTTPS server.
```sh
sudo systemctl stop nginx
# Change robmllze0.xyz and www.robmllze0.xyz to your domains.
sudo certbot certonly --standalone -d robmllze0.xyz -d www.robmllze0.xyz --register-unsafely-without-email
sudo systemctl start nginx
```
2. Specify an email, e.g. `robmllze@gmail.com` (but you have to omit --register-unsafely-without-email in previous step).
3. This should generate two files, e.g. `/etc/letsencrypt/live/robmllze0.xyz/fullchain.pem` and `/etc/letsencrypt/live/robmllze0.xyz/privkey.pem`.
4. Create a conf file for nginx with `sudo nano /etc/nginx/conf.d/robmllze0.conf`. Modify it like `robmllze0.conf` included in this repo.
5. Save the file then test nginx for errors with `sudo nginx -t`.
6. Test that certbot is able to automatically renew the certificates when needed (every 90 days) `sudo certbot renew --dry-run`.
7. Start or reload nginx `sudo systemctl start nginx` or `sudo systemctl reload nginx` (stop with `sudo systemctl stop nginx`).

## How to set up a remote Firebase Emulator on the VM

1. Create a VM and open its console.
2. Ensure you have Node.js 16 and Firebase Tools installed.
3. Create an Ingress firewall rule called `allow-firestore-emulators` [here](https://console.cloud.google.com/net-security/firewall-manager/) to allow the ports for TCP used by Firestore Emulator (4000,4001,5001,5002,8080,8081,9099,9100,9199,9200). Be sure to target the service account and not a tag.
# ...
4. Enable port forwarding for all ports used by Firestore Emulator, to make them accessible, for example:
```sh
compute ssh robmllze2 --project xyz-app-example --zone europe-west2-a
sudo systemctl stop nginx
gcloud compute ssh robmllze0 --project xyz-app-example --zone australia-southeast2-a -- -L 4000:localhost:4000 -L 5001:localhost:5001 -L 8080:localhost:8080 -L 9099:localhost:9099 -L 9199:localhost:9199
sudo systemctl start nginx

gcloud compute ssh robmllze2 --project xyz-app-example --zone europe-west2-a -- -L 4000:localhost:4000 -L 5001:localhost:5001 -L 8080:localhost:8080 -L 9099:localhost:9099 -L 9199:localhost:9199
```
5. Create the Firebase project.
```sh
rm -r firebase_emulators
mkdir firebase_emulators && cd firebase_emulators
firebase init # select  "auth", "firestore", "functions" and "storage", set the UI port to 4000 and the functions language as Python
ls -l
nano firebase.json # modify firebase.json to the firebase.json included in this repo
```
7. Start the emulators with `firebase emulators:start`
8. Access the emulator dashboard with the VM's public IP or URL, like [http:\\34.129.161.164] [https:\\www.robmllze0.xyz:4001/] or [http:\\robmllze0.xyz:4000/].
9. Before running the app in Chrome, you must set the site to allow mixed content.
  - On Chrome on desktop or Android, go to `chrome://flags/#unsafely-treat-insecure-origin-as-secure`. Enable the flag `unsafely-treat-insecure-origin-as-secure` then add all the ports used by the app and emulator, e.g. `http://robmllze0.xyz:5001,https://robmllze0.xyz:5002,http://robmllze0.xyz:8080,https://robmllze0.xyz:8081,http://robmllze0.xyz:9099,https://robmllze0.xyz:9100,http://robmllze0.xyz:9199,https://robmllze0.xyz:9200,http://robmllze1.xyz:5001,https://robmllze1.xyz:5002,http://robmllze1.xyz:8080,https://robmllze1.xyz:8081,http://robmllze1.xyz:9099,https://robmllze1.xyz:9100,http://robmllze1.xyz:9199,https://robmllze1.xyz:9200,http://robmllze2.xyz:5001,https://robmllze2.xyz:5002,http://robmllze2.xyz:8080,https://robmllze2.xyz:8081,http://robmllze2.xyz:9099,https://robmllze2.xyz:9100,http://robmllze2.xyz:9199,https://robmllze2.xyz:9200`.
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

