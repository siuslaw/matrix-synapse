# matrix-synapse
Guide on how to create a matrix synapse.
## Requirements:
1. Debian 10/11
2. NGINX (installed via ```sudo apt install nginx```)
3. COTURN (installed via ```sudo apt install coturn```)
4. PostgreSQL (installed via ```sudo apt install postgresql```)
5. Port-forwarding: 80, 443, 8448 (on your router/modem if you're self hosting)
6. Domain and a publicly set DNS entry towards your server/computer (IPV4)
7. Basic Linux knowledge
#
## NGINX config:
Create new file in ```/etc/nginx/sites-available/YOUR_DOMAIN```  (of course replace YOUR_DOMAIN with your actual domain)
Then paste the following:
``` server {
        server_name YOUR_DOMAIN;

        listen 80;
        listen [::]:80;

        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server;
       
        listen 8448 ssl http2 default_server;
        listen [::]:8448 ssl http2 default_server; 
	
	location ~* ^(\/_matrix|\/_synapse\/client) {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
        client_max_body_size 50M;
        }

	location /.well-known/matrix/client {
        return 200 '{"m.homeserver": {"base_url": "https://YOUR_DOMAIN"}}';
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
        }

        location /.well-known/matrix/server {
        return 200 '{"m.server": "YOUR_DOMAIN:443"}';
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
        }
}
```
#
## COTURN config:
Paste the following in  ```/etc/turnserver.conf```
```
use-auth-secret
static-auth-secret=CHANGEME
realm=YOUR_DOMAIN.COM
listening-port=3478
tls-listening-port=5349
min-port=49160
max-port=49200
verbose
allow-loopback-peers
cli-password=CHANGEME
external-ip=EXTERNALIP
```

Then in your synapse config homeserver.yaml you just need to add these following lines:
```
turn_uris:
  - "turn:YOURDOMAIN.COM:3478?transport=udp"
  - "turn:YOURDOMAIN.COM:3478?transport=tcp"
  - "turns:YOURDOMAIN.COM:3478?transport=udp"
  - "turns:YOURDOMAIN.COM:3478?transport=tcp"
turn_shared_secret: "CHANGEME"
turn_user_lifetime: 1h
turn_allow_guests: true
```
(make sure you only change YOURDOMAIN.COM)
#
## Enabling your site:
``` sudo ln -s /etc/nginx/sites-available/YOUR_DOMAIN /etc/nginx/sites-enabled```
#
## Installing packages:
```
sudo apt install -y lsb-release wget apt-transport-https
sudo wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] https://packages.matrix.org/debian/ $(lsb_release -cs) main" |
    sudo tee /etc/apt/sources.list.d/matrix-org.list
```
```
sudo apt update
sudo apt install matrix-synapse-py3
```
#
## Restart services:
```sudo systemctl restart nginx matrix-synapse```
#
## Encryption with Certbot
```sudo apt install python3-certbot python3-certbot-nginx```
```sudo certbot --nginx -d YOUR_DOMAIN```

#
## Enabling PostgreSQL
```sudo systemctl start postgresql```
```sudo su - postgres```
```sudo passwd postgres``` (if users has no password)
```createuser --pwprompt synapse_user```
```
psql
CREATE DATABASE synapse
 ENCODING 'UTF8'
 LC_COLLATE='C'
 LC_CTYPE='C'
 template=template0
 OWNER synapse_user;
```
Edit the db config: ```/etc/matrix-synapse/homeserver.yaml``` and **comment** out the following:
```
# database:
  # name: sqlite3
  # args:
    # database: DATADIR/homeserver.db
```
Then, uncomment the following configuration above, and replace the credentials correctly:
```
database:
  name: psycopg2
  args:
    user: synapse_user
    password: secretpassword
    database: synapse
    host: localhost
    cp_min: 5
    cp_max: 10
```
Time to restart the synapse: ```sudo systemctl restart matrix-synapse```
#
## Synapse customization:

#
### Enabling registration:
```enable_registration: true```

To be able to create users from your homeserver, you also need to uncomment this line in your matrix config:
```registration_shared_secret: CHANGME```

Now you can create users, to create a new admin user on your server:
```sudo register_new_matrix_user -u YOUR_USERNAME -p YOUR_PASSWORD -a -c /etc/matrix-synapse/homeserver.yaml```

#
### Federation: (optional)
```allow_public_rooms_over_federation: true```
```allow_public_rooms_without_auth: true```
#
### URL Previews (optional)
```url_preview_enabled: true```

#
## Congrats, now you should have a fully functionating matrix synapse.



### WIP:
(X) automated bash script








