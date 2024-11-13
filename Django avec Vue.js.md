Voici un guide pour déployer une application Django avec Vue.js sur un serveur Linux (par exemple Ubuntu), avec Nginx pour le serveur web, PostgreSQL et MySQL pour la gestion des bases de données.

### 1. Prérequis

- Serveur Linux (Ubuntu 20.04 ou plus récent)
- Accès SSH à votre serveur
- Accès root ou droits `sudo`

### 2. Installation des composants nécessaires

1. **Mettre à jour le serveur** :
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Installer les dépendances** :
   ```bash
   sudo apt install python3-pip python3-venv nginx git
   ```

3. **Installer PostgreSQL et MySQL** :
   ```bash
   sudo apt install postgresql postgresql-contrib
   sudo apt install mysql-server
   ```

### 3. Configuration des bases de données

#### PostgreSQL

1. Connectez-vous à PostgreSQL et créez un utilisateur et une base de données pour votre application Django :
   ```bash
   sudo -u postgres psql
   ```
   ```sql
   CREATE DATABASE nom_base_postgresql;
   CREATE USER nom_utilisateur_postgresql WITH PASSWORD 'votre_mot_de_passe';
   ALTER ROLE nom_utilisateur_postgresql SET client_encoding TO 'utf8';
   ALTER ROLE nom_utilisateur_postgresql SET default_transaction_isolation TO 'read committed';
   ALTER ROLE nom_utilisateur_postgresql SET timezone TO 'UTC';
   GRANT ALL PRIVILEGES ON DATABASE nom_base_postgresql TO nom_utilisateur_postgresql;
   \q
   ```

2. **Configurer PostgreSQL pour accepter les connexions locales** :
   ```bash
   sudo nano /etc/postgresql/12/main/pg_hba.conf
   ```
   - Remplacez `peer` par `md5` pour `local` et `host`.

3. Redémarrez PostgreSQL :
   ```bash
   sudo systemctl restart postgresql
   ```

#### MySQL

1. Connectez-vous à MySQL et créez une base de données et un utilisateur :
   ```bash
   sudo mysql
   ```
   ```sql
   CREATE DATABASE nom_base_mysql;
   CREATE USER 'nom_utilisateur_mysql'@'localhost' IDENTIFIED BY 'votre_mot_de_passe';
   GRANT ALL PRIVILEGES ON nom_base_mysql.* TO 'nom_utilisateur_mysql'@'localhost';
   FLUSH PRIVILEGES;
   EXIT;
   ```

2. Redémarrez MySQL :
   ```bash
   sudo systemctl restart mysql
   ```

### 4. Configuration de l'application Django

1. **Cloner le projet Django** :
   ```bash
   git clone https://github.com/votre_repo.git
   cd votre_repo
   ```

2. **Configurer l'environnement virtuel** :
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```

3. **Configurer les paramètres de la base de données** dans le fichier `settings.py` :
   ```python
   DATABASES = {
       'default': {
           'ENGINE': 'django.db.backends.postgresql',  # ou 'django.db.backends.mysql' pour MySQL
           'NAME': 'nom_base',
           'USER': 'nom_utilisateur',
           'PASSWORD': 'votre_mot_de_passe',
           'HOST': 'localhost',
           'PORT': '',
       }
   }
   ```

4. **Appliquer les migrations et collecter les fichiers statiques** :
   ```bash
   python manage.py migrate
   python manage.py collectstatic
   ```

### 5. Configuration de Vue.js

1. **Installer Node.js et npm** :
   ```bash
   sudo apt install nodejs npm
   ```

2. **Aller dans le répertoire de votre projet Vue.js** et installer les dépendances :
   ```bash
   cd frontend  # ou le nom de votre répertoire Vue.js
   npm install
   ```

3. **Construire le projet** :
   ```bash
   npm run build
   ```

4. **Copier les fichiers construits vers Django** (le répertoire `dist` de Vue.js) :
   ```bash
   cp -r dist/* ../votre_projet_django/static/
   ```

### 6. Configurer Gunicorn pour servir Django

1. **Installer Gunicorn** :
   ```bash
   pip install gunicorn
   ```

2. **Créer un service systemd pour Gunicorn** :
   ```bash
   sudo nano /etc/systemd/system/gunicorn.service
   ```
   Contenu du fichier :
   ```ini
   [Unit]
   Description=gunicorn daemon
   After=network.target

   [Service]
   User=your_user
   Group=www-data
   WorkingDirectory=/chemin/vers/votre_projet_django
   ExecStart=/chemin/vers/votre_projet_django/venv/bin/gunicorn --workers 3 --bind unix:/chemin/vers/votre_projet_django.sock votre_projet.wsgi:application

   [Install]
   WantedBy=multi-user.target
   ```

3. **Démarrer et activer Gunicorn** :
   ```bash
   sudo systemctl start gunicorn
   sudo systemctl enable gunicorn
   ```

### 7. Configuration de Nginx

1. **Créer un bloc de configuration pour Nginx** :
   ```bash
   sudo nano /etc/nginx/sites-available/votre_projet
   ```
   Contenu du fichier :
   ```nginx
   server {
       listen 80;
       server_name votre_domaine.com;

       location / {
           proxy_pass http://unix:/chemin/vers/votre_projet_django.sock;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }

       location /static/ {
           alias /chemin/vers/votre_projet_django/static/;
       }

       location /media/ {
           alias /chemin/vers/votre_projet_django/media/;
       }
   }
   ```

   - Exemple :
   ```
   server {
       listen 8080;
       server_name alertedgped.finances.bj;
   
       location / {
           root /var/www/html/alertedgped/;
           try_files $uri $uri/ /index.html;
       }
   
   
       location /api/ {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }

   ```

2. **Activer le site et redémarrer Nginx** :
   ```bash
   sudo ln -s /etc/nginx/sites-available/votre_projet /etc/nginx/sites-enabled
   sudo systemctl restart nginx
   ```

### 8. Configurer le pare-feu

Autorisez Nginx sur le pare-feu :
```bash
sudo ufw allow 'Nginx Full'
```

### 9. Tester le déploiement

Ouvrez votre navigateur et accédez à `http://votre_domaine.com` pour tester si l'application fonctionne.

### 10. Gérer les services

Pour redémarrer les services si nécessaire :
```bash
sudo systemctl restart gunicorn
sudo systemctl restart nginx
```

Votre application Django avec Vue.js devrait maintenant être déployée sur le serveur Linux avec Nginx, configurée pour utiliser PostgreSQL et MySQL.
