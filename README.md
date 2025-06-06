### kdo
---

## 🎯 OBJECTIF FINAL

Un utilisateur tape dans son navigateur :

* `wordpress.local` → Bind9 redirige vers 192.168.30.100 (reverse proxy), qui transmet à 192.168.30.20 (WordPress)
* `dashy.local` → vers 192.168.30.90:6000
* `gitea.local` → vers 192.168.30.90:3000

---

## ✅ 1. CONFIGURER LE DNS AVEC BIND9 (sur 192.168.30.10)

### 🔹 Installer Bind9

```bash
sudo apt update
sudo apt install bind9 -y
```

### 🔹 Créer une zone DNS pour `.local`

Dans `/etc/bind/named.conf.local`, ajoute :

```bash
zone "local" {
    type master;
    file "/etc/bind/db.local";
};
```

### 🔹 Créer le fichier de zone `/etc/bind/db.local`

```bash
$TTL 604800
@   IN  SOA dns.local. root.local. (
              2         ; Serial
         604800         ; Refresh
          86400         ; Retry
        2419200         ; Expire
         604800 )       ; Negative Cache TTL

@       IN  NS  dns.local.

dns     IN  A   192.168.30.10
wordpress IN A  192.168.30.100
dashy    IN A   192.168.30.100
gitea    IN A   192.168.30.100
```

> Tous les domaines pointent vers le reverse proxy (192.168.30.100)

### 🔹 Redémarrer Bind9

```bash
sudo named-checkconf
sudo named-checkzone local /etc/bind/db.local
sudo systemctl restart bind9
```

---

## ✅ 2. CONFIGURER LES CLIENTS POUR UTILISER TON DNS

Sur les PC ou VMs clientes, mets **192.168.30.10** comme **serveur DNS** dans les paramètres réseau.

Teste avec :

```bash
ping wordpress.local
```

---

## ✅ 3. CONFIGURER LE REVERSE PROXY NGINX (sur 192.168.30.100)

### 🔹 Installer Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### 🔹 Configurer les sites

Crée trois fichiers dans `/etc/nginx/sites-available` :

#### 🔸 `wordpress.local`

```nginx
server {
    listen 80;
    server_name wordpress.local;

    location / {
        proxy_pass http://192.168.30.20;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### 🔸 `dashy.local`

```nginx
server {
    listen 80;
    server_name dashy.local;

    location / {
        proxy_pass http://192.168.30.90:6000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### 🔸 `gitea.local`

```nginx
server {
    listen 80;
    server_name gitea.local;

    location / {
        proxy_pass http://192.168.30.90:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 🔹 Activer les sites

```bash
sudo ln -s /etc/nginx/sites-available/wordpress.local /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/dashy.local /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/gitea.local /etc/nginx/sites-enabled/
```

### 🔹 Redémarrer Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## ✅ 4. TESTER

Depuis un client configuré avec le DNS :

* Accède à `http://wordpress.local` → doit afficher WordPress
* Accède à `http://dashy.local` → Dashy
* Accède à `http://gitea.local` → Gitea

---

Si tu veux, je peux te faire un **schéma réseau clair** ou une **vérification des fichiers de conf** — tu veux ?























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
