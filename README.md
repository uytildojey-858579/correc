- docker pull authelia/authelia
- sudo mkdir -p /etc/authelia
- cd /etc/authelia

```bash
nano /etc/authelia/configuration.yml
```

```yml
server:
  host: 0.0.0.0
  port: 9091

authentication_backend:
  file:
    path: /config/users_database.yml

access_control:
  default_policy: deny
  rules:
    - domain: "*.auth.local"
      policy: one_factor

session:
  name: authelia_session
  secret: "Jev78444d4d4ddddd44s5qd568qd745c6x4114eff4d9vcx"
  expiration: 1h

storage:
  local:
    path: /config/db.sqlite3

notifier:
  filesystem:
    directory: /config/notification
```
- pour générer un secret
```bash
openssl rand -hex 32
```

```bash
nano /etc/authelia/users_database.yml
```

- on peut ajouter ou modifier les utilisateurs 
```yml
users:
  admin:
    password: "#le mdp généré"
    displayname: "Administrateur"
    email: admin@admin.com
```


```bash
docker run authelia/authelia:latest authelia  '0000'
```

```bash
docker run -d \
  --name authelia \
  -p 9091:9091 \
  -v /etc/authelia:/config \
  authelia/authelia
```


```bash
/etc/nginx/sites-available/proxy
```


```
# HTTP vers HTTPS (redirection)
server {
    listen 80;
    server_name wordpress.auth.local gitea.auth.local dashy.auth.local;
    return 301 https://$host$request_uri;
}

# HTTPS pour WordPress
server {
    listen 443 ssl;
    server_name wordpress.auth.local;
    
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
    ssl_trusted_certificate /etc/nginx/ssl/ca_root.crt;

    location / {
        auth_request /authelia;
        auth_request_set $target_url $scheme://$http_host$request_uri;
        error_page 401 =302 https://wordpress.auth.local:9091/?rd=$target_url;

        proxy_pass http://192.168.30.20:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location = /authelia {
        internal;
        proxy_pass http://127.0.0.1:9091/api/verify;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTPS pour Gitea
server {
    listen 443 ssl;
    server_name gitea.auth.local;
    
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
    ssl_trusted_certificate /etc/nginx/ssl/ca_root.crt;

    location / {
        auth_request /authelia;
        auth_request_set $target_url $scheme://$http_host$request_uri;
        error_page 401 =302 https://gitea.auth.local:9091/?rd=$target_url;

        proxy_pass http://192.168.30.90:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location = /authelia {
        internal;
        proxy_pass http://127.0.0.1:9091/api/verify;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTPS pour Dashy
server {
    listen 80;
    server_name wordpress.auth.local gitea.auth.local dashy.auth.local;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name dashy.auth.local;
    
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
    ssl_trusted_certificate /etc/nginx/ssl/ca_root.crt;

    location / {
        auth_request /authelia;
        auth_request_set $target_url $scheme://$http_host$request_uri;
        error_page 401 =302 https://dashy.auth.local:9091/?rd=$target_url;

        proxy_pass http://192.168.30.90:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    location = /authelia {
        internal;
        proxy_pass http://127.0.0.1:9091/api/verify;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```



































### Ya pas le san, dcp il faut le rajouter

 Sur le conteneur PKI (création de l’autorité racine)
bash

# Générer la clé privée de la CA
openssl genrsa -out ca_root.key 4096

# Générer le certificat auto-signé de la CA
openssl req -x509 -new -nodes -key ca_root.key -sha256 -days 3650 -out ca_root.crt \
  -subj "/C=FR/ST=France/L=Paris/O=MonOrganisation/CN=MonCA"

2. Sur le conteneur test_apache (générer la clé et la CSR du serveur)
bash

# Générer la clé privée du serveur
openssl genrsa -out server.key 2048

# Générer la CSR (remplis bien le CN et les SAN si besoin)
openssl req -new -key server.key -out server.csr \
  -subj "/C=FR/ST=France/L=Paris/O=MonOrganisation/CN=mon-site.local"

3. Transfert de la CSR vers le PKI

Transfère le fichier server.csr du conteneur test_apache vers le PKI (copie manuelle ou via un volume partagé Docker).
4. Sur le PKI (signature de la CSR)
bash

# Signer la CSR pour générer le certificat du serveur
openssl x509 -req -in server.csr -CA ca_root.crt -CAkey ca_root.key -CAcreateserial \
  -out server.crt -days 365 -sha256

5. Transfert des fichiers vers test_apache

Transfère les fichiers suivants sur test_apache :

    server.key (déjà présent)
    server.crt (nouveau, signé par la CA)
    ca_root.crt (pour la chaîne de confiance)

6. Configurer Apache pour le HTTPS

Dans la configuration SSL d’Apache (ex : /etc/apache2/sites-available/default-ssl.conf), ajoute/modifie :
Code

SSLEngine on
SSLCertificateFile      /chemin/vers/server.crt
SSLCertificateKeyFile   /chemin/vers/server.key
SSLCertificateChainFile /chemin/vers/ca_root.crt

Puis active le module SSL et le site :
bash

a2enmod ssl
a2ensite default-ssl
service apache2 reload
