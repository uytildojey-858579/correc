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
