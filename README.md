Voici une procédure complète et claire pour installer et configurer **Authelia v4.37.5** en Docker, adaptée à ce que nous venons de faire. Cette version est stable et bien documentée.

---

## 1. Prérequis

- Docker installé sur ton serveur.
- Un dossier de configuration pour Authelia (ex : `/etc/authelia`).
- Un nom de domaine local ou public pour l’authentification (ex : `auth.local`).
- Un fichier d’utilisateurs : `/etc/authelia/users_database.yml` (tu peux l’ajouter après, ce n’est pas bloquant pour démarrer le service).

---

## 2. Génère tes secrets

Utilise ces commandes pour générer des secrets (remplace-les dans la config ci-dessous) :

```bash
openssl rand -hex 32  # pour 'secret', 'jwt_secret', 'encryption_key'
```

---

## 3. Crée le fichier de configuration

Crée le fichier `/etc/authelia/configuration.yml` avec ce contenu :

```yaml name=/etc/authelia/configuration.yml
server:
  host: 0.0.0.0
  port: 9091

authentication_backend:
  file:
    path: /config/users_database.yml

access_control:
  default_policy: deny
  rules:
    - domain: "auth.local"
      policy: one_factor
    - domain: "*.auth.local"
      policy: one_factor

session:
  name: authelia_session
  secret: "COLLE_ICI_TA_CLE_SECRET"
  expiration: 1h
  domain: auth.local

storage:
  local:
    path: /config/db.sqlite3
  encryption_key: "COLLE_ICI_TA_CLE_ENCRYPTION"

notifier:
  filesystem:
    filename: /config/notification.txt

jwt_secret: "COLLE_ICI_TA_CLE_JWT"
```

**Remplace les valeurs des secrets par celles générées.**

---

## 4. Lance le conteneur Docker

```bash
docker stop authelia || true
docker rm authelia || true

docker run -d \
  --name authelia \
  -p 9091:9091 \
  -v /etc/authelia:/config \
  authelia/authelia:4.37.5
```

---

## 5. Vérification

1. **Vérifie les logs** pour t’assurer qu’il n’y a pas d’erreur :
   ```bash
   docker logs -f authelia
   ```
   Tu dois voir un message comme `Authelia is running`.

2. **Accède à l’interface d’Authelia** avec :  
   [http://localhost:9091](http://localhost:9091)  
   (ou adapte l’URL selon ton reverse proxy et ton domaine)

---

## 6. Ajoute des utilisateurs

Crée le fichier `/etc/authelia/users_database.yml` avec un exemple d’utilisateur (à adapter à tes besoins) :

```yaml name=/etc/authelia/users_database.yml
users:
  alice:
    displayname: "Alice Doe"
    password: "$argon2id$v=19$m=65536,t=2,p=1$...$..."  # Hash généré avec authelia hash-password
    email: alice@example.com
    groups:
      - admins
```

Pour générer un hash de mot de passe :
```bash
docker run --rm authelia/authelia:4.37.5 authelia hash-password 'ton_mot_de_passe'
```

---

## 7. Redémarre le conteneur si tu modifies la config

```bash
docker restart authelia
```

---

## 8. Points d’attention

- **Nom de domaine** : Mets bien le même domaine partout (`auth.local` ou autre).
- **Reverse proxy** : Si tu utilises Nginx, Traefik, etc., configure-le pour rediriger vers le port 9091 de ce conteneur.
- **Sécurité** : Ne partage jamais tes secrets et garde-les dans un endroit sûr.

---

### **Résumé rapide**

1. Place le fichier de config corrigé dans `/etc/authelia`.
2. Génére les secrets.
3. Lance le conteneur avec la bonne version.
4. Vérifie les logs.
5. Ajoute tes utilisateurs.

---

Si tu veux une procédure personnalisée avec reverse proxy, LDAP ou autre, dis-le moi !  
Bravo d’avoir persévéré 👏





















-
-
-
-
-
-
-
-
-
-
-
-
-
-
-




- docker pull authelia/authelia
- sudo mkdir -p /etc/authelia
- cd /etc/authelia

```bash
nano /etc/authelia/configuration.yml
```

```yml
server:
  address: 0.0.0.0
  port: 9091

authentication_backend:
  file:
    path: /config/users_database.yml

access_control:
  default_policy: deny
  rules:
    - domain: "auth.local"
      policy: one_factor
    - domain: "*.auth.local"
      policy: one_factor

session:
  name: authelia_session
  secret: "remplace_ceci_par_un_hhhhhhhh77777778888888888secret_long"
  jwt_secret: "remplace_ceci_par_un_autre_se88888888888cret_lonhhhhhhhhhhhhhhhhhhhhhhhhg"
  expiration: 1h
  cookies:
    domain: auth.local  # PAS de point devant

storage:
  local:
    path: /config/db.sqlite3
    encryption_key: "remplace_ceci_par9999999999999999999999999999_une_cle_32caracteres"

notifier:
  filesystem:
    filename: /config/notification.txt
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
