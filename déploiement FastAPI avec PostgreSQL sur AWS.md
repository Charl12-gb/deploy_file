## Étapes de déploiement FastAPI avec PostgreSQL sur AWS

### Prérequis

1. **Compte AWS** et **AWS CLI configuré**.
2. **Serveur Jenkins** pour CI/CD.
3. **Dépôt GitHub** pour le code source.
4. **Docker et Docker Compose**.

---

### 1. Structure et Configuration du Projet FastAPI

Organisez votre projet FastAPI comme suit :

```
fastapi-aws-app/
│
├── app/
│   ├── main.py
│   ├── models/
│   ├── api/
│   ├── core/
│   ├── db.py
│
├── alembic/
│
├── Dockerfile
├── docker-compose.yml
├── entrypoint.sh
├── .env
├── requirements.txt
└── alembic.ini
```

---

### 2. Configuration des Fichiers de Déploiement

#### 2.1 Dockerfile

Voici votre `Dockerfile` mis à jour pour PostgreSQL et MySQL (si nécessaire).

```Dockerfile
# Utilisation de l'image de base Python 3.12
FROM python:3.12.2-slim

# Installer les dépendances système pour mysqlclient et netcat (en utilisant netcat-openbsd)
RUN apt-get update && apt-get install -y \
    pkg-config \
    libpq-dev \
    gcc \
    g++ \
    apache2 \
    netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*

# Configurer le ServerName pour éviter le warning d'Apache
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

# Créer et définir le répertoire de travail
WORKDIR /app

# Copier le fichier requirements.txt et installer les dépendances Python
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copier le reste du code de l'application
COPY . .

# Copier le script d'entrée
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

# Exposer le port par défaut de FastAPI
EXPOSE 8000

# Utiliser entrypoint.sh comme point d'entrée
ENTRYPOINT ["/app/entrypoint.sh"]
```

#### 2.2 docker-compose.yml

Ce fichier `docker-compose.yml` est configuré pour PostgreSQL.

```yaml
version: '3.8'

services:
  app:
    build: .
    container_name: fastapi_app
    env_file: .env
    ports:
      - "8000:8000"
    volumes:
      - .:/app
    depends_on:
      - db
    restart: always

  db:
    image: postgres:13
    container_name: ${DB_HOST}
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  db_data:
```

#### 2.3 entrypoint.sh

Le fichier `entrypoint.sh` vérifie la disponibilité de la base de données avant de démarrer l’application.

```bash
#!/bin/sh

# Attente que la base de données PostgreSQL soit prête
echo "Waiting for PostgreSQL to be ready..."
until nc -z -v -w30 $DB_HOST $DB_PORT
do
  echo "Waiting for PostgreSQL..."
  sleep 10
done

# Appliquer les migrations avec Alembic
echo "Applying database migrations..."
alembic upgrade head

# Exécuter le script de seed
echo "Seeding the database..."
python seed.py seed_default_user --email admin@example.com --password password

# Démarrer l'application avec Uvicorn en mode rechargement
echo "Starting FastAPI server with auto-reload..."
exec uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

#### 2.4 .env

Votre fichier `.env` contient les configurations pour PostgreSQL, le SMTP et d’autres variables d’environnement.

```env
# Connexion à la base de données
DATABASE_URL=postgresql+asyncpg://postgres:root@localhost:5432/cica_db
TYPE_DB=postgres

# Variables de sécurité
SECRET_KEY="ad12g-34ghe-f5g6-h7890-jklmno"
ALGORITHM="HS256"
ACCESS_TOKEN_EXPIRE_MINUTES=1440

# Variables CORS
ORIGIN_CORS="http://localhost:8080,http://127.0.0.1:8080,http://localhost:8000,http://127.0.0.1:8000"

# Envoi de mail
EMAIL_PROVIDER=smtp
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=
SMTP_PASSWORD=
SENDER_EMAIL=

# Variables de la base de données PostgreSQL
DB_USER=postgres
DB_PASSWORD=root
DB_NAME=cica_db
DB_HOST=localhost
DB_PORT=5432
```

---

### 3. Initialisation et Configuration d’Alembic

#### 3.1 Initialisation d'Alembic

Dans le terminal, initialisez Alembic :

```bash
alembic init alembic
```

#### 3.2 Configuration de `alembic.ini`

Dans le fichier `alembic.ini`, modifiez `sqlalchemy.url` avec l'URL de votre base de données :

```ini
sqlalchemy.url = postgresql+asyncpg://postgres:root@localhost:5432/cica_db
```

---

### 4. Configuration de GitHub, Jenkins et AWS pour CI/CD

#### **4.1 Configuration du Dépôt GitHub et Jenkins**

1. **Dépôt GitHub** :  
   - Créez un dépôt pour votre projet sur GitHub.  
   - Poussez votre code source, y compris le fichier `Dockerfile` et le fichier `Jenkinsfile`.  

2. **Configuration Jenkins** :  
   - Installez Jenkins sur une machine (locale ou distante).  
   - Configurez les plugins nécessaires, comme :
     - **Docker Pipeline** (pour gérer Docker dans les pipelines).
     - **SSH Agent** (pour les connexions SSH).  
   - Ajoutez les credentials nécessaires :  
     - **Docker Hub Credentials** : Identifiants pour se connecter à Docker Hub.  
     - **SSH Credentials** : Clé privée utilisée pour se connecter à l'instance EC2.  

---

#### **4.2 Fichier Jenkinsfile**

Voici un exemple de fichier `Jenkinsfile` adapté pour déployer une application Docker sur une instance EC2 :

```groovy
    pipeline {
        agent any
        environment {
            DOCKER_HUB_REPO = 'votre_nom_utilisateur/fastapi-aws-app' // Nom du dépôt Docker Hub
            EC2_IP = 'your-ec2-public-ip' // Adresse IP publique de l'instance EC2
            EC2_USER = 'ec2-user' // Utilisateur par défaut pour Amazon Linux
            DOCKER_IMAGE_TAG = "${env.BUILD_ID}" // Identifiant unique pour chaque build
        }
        stages {
            stage('Checkout Code') {
                steps {
                    // Récupération du code source depuis GitHub
                    checkout scm
                }
            }
            stage('Build Docker Image') {
                steps {
                    script {
                        // Construction de l'image Docker avec un tag unique
                        dockerImage = docker.build("${DOCKER_HUB_REPO}:${DOCKER_IMAGE_TAG}")
                    }
                }
            }
            stage('Push Docker Image') {
                steps {
                    script {
                        // Connexion manuelle à Docker Hub avec les secrets GitHub
                        withCredentials([
                            string(credentialsId: 'DOCKER_HUB_USERNAME', variable: 'DOCKER_USERNAME'),
                            string(credentialsId: 'DOCKER_HUB_PASSWORD', variable: 'DOCKER_PASSWORD')
                        ]) {
                            // Connexion à Docker Hub
                            sh """
                            echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin
                            """
                            
                            // Push de l'image Docker
                            dockerImage.push() // Push avec le tag BUILD_ID
                            dockerImage.push('latest') // Ajout d'un tag "latest" pour les déploiements récents
                        }
                    }
                }
            }
            stage('Deploy to EC2') {
                steps {
                    script {
                        // Connexion en SSH à l'instance EC2 et déploiement
                        sshagent(['ec2-ssh-credentials']) {
                            sh """
                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} << EOF
                            docker pull ${DOCKER_HUB_REPO}:${DOCKER_IMAGE_TAG}
                            docker stop fastapi-container || true
                            docker rm fastapi-container || true
                            docker run -d --name fastapi-container -p 8000:8000 ${DOCKER_HUB_REPO}:${DOCKER_IMAGE_TAG}
                            EOF
                            """
                        }
                    }
                }
            }
        }
        post {
            always {
                // Nettoyage de l'espace de travail Jenkins
                cleanWs()
            }
        }
    }
```

---

#### **4.3 Étapes de Configuration Supplémentaires**

1. **Configuration SSH pour l'Instance EC2** :  
   - Ajoutez la clé privée SSH utilisée pour accéder à l’instance dans **Jenkins > Manage Credentials**.
   - Donnez un identifiant à ces credentials, comme `ec2-ssh-credentials`.  

2. **Ouverture des Ports dans le Groupe de Sécurité EC2** :  
   - **Port 8000** : Autorisez l'accès public ou restreint pour accéder à l'application FastAPI.  
   - **Port 22 (SSH)** : Assurez-vous que Jenkins peut se connecter via SSH à l'instance EC2.

3. **Connexion Docker Hub** :  
   - Configurez les credentials Docker Hub dans **Jenkins > Manage Credentials**.  
   - Donnez un identifiant à ces credentials, comme `docker-hub-credentials`.  

---

#### **Résumé**

Ce pipeline Jenkins vous permet de :  
- Construire une image Docker avec votre application FastAPI.  
- Publier cette image sur Docker Hub.  
- Déployer automatiquement l'application sur une instance EC2 via SSH.  

En cas de modification du code source dans GitHub, le pipeline peut être déclenché automatiquement pour redéployer l'application.

---

### Étape 4.3 : Déploiement sur AWS ECS et RDS

#### 1. Configurer Amazon RDS pour PostgreSQL

1. **Connectez-vous à la console AWS** et accédez à **Amazon RDS**.
2. Cliquez sur **Créer une base de données**.

3. **Choix de la méthode de création** :
   - Sélectionnez **Mode standard** pour une configuration manuelle.
   
4. **Type de moteur** :
   - Choisissez **PostgreSQL** comme moteur de base de données.

5. **Version** :
   - Sélectionnez la dernière version compatible avec votre application (par exemple, PostgreSQL 13 ou 14).

6. **Paramètres de l’instance** :
   - Choisissez un nom pour votre instance, comme `fastapi-postgres-db`.

7. **Authentification** :
   - Définissez le nom d'utilisateur principal (ex. `postgres`) et un mot de passe sécurisé pour l'instance RDS.
   - Enregistrez ces informations, car elles seront nécessaires pour configurer la connexion.

8. **Paramètres de connectivité** :
   - **VPC** : Sélectionnez le VPC où vos services ECS seront également déployés, ou laissez le paramètre par défaut.
   - **Sous-réseau** : Utilisez les sous-réseaux par défaut ou ceux de votre VPC.
   - **Public accessibility** : Si vous avez besoin d'un accès public à votre base de données, sélectionnez **Yes**. Sinon, laissez sur **No** pour une configuration plus sécurisée.
   - **Groupe de sécurité** : Sélectionnez un groupe de sécurité qui permet les connexions entrantes sur le port 5432 (port par défaut de PostgreSQL) depuis votre application ECS.

9. **Création de la base de données** :
   - Cliquez sur **Créer la base de données**. Cela peut prendre quelques minutes.

10. **Notez l’URL de connexion** :
    - Une fois la base de données créée, accédez à l’onglet **Configuration** de votre instance RDS.
    - Notez l'**Endpoint** de la base de données, qui devrait ressembler à : `fastapi-postgres-db.cluster-xxxxxxxxxx.us-west-2.rds.amazonaws.com`.
    - Mettez à jour le fichier `.env` avec cette URL, en utilisant le format suivant pour `DATABASE_URL` :
      ```plaintext
      DATABASE_URL=postgresql://postgres:<password>@fastapi-postgres-db.cluster-xxxxxxxxxx.us-west-2.rds.amazonaws.com:5432/<database_name>
      ```

---

### 2. Configurer Amazon ECS et EC2 pour héberger l'application Docker

1. **Créer une Instance EC2** :  
   - Accédez à la console AWS EC2 et cliquez sur **Lancer une Instance**.  
   - **Nom de l'Instance** : Donnez un nom descriptif à votre instance, comme `fastapi-ec2-instance`.  
   - **Choix de l’Image AMI** : Sélectionnez **Amazon Linux 2 AMI** (gratuitement éligible au niveau gratuit AWS).  
   - **Type d’Instance** : Choisissez un type adapté à votre charge, comme `t2.micro` pour des tests légers.  
   - **Configuration des Clés SSH** :
     - **Option 1 (avec paire de clés)** : Si vous avez une paire de clés SSH existante, sélectionnez-la. Sinon, créez-en une nouvelle pour vous connecter via SSH.
     - **Option 2 (sans paire de clés)** : Si vous continuez sans paire de clés, assurez-vous que votre instance est accessible par d'autres moyens.  
   - **Créer un Groupe de Sécurité** : 
     - **SSH (port 22)** : Autorisez uniquement votre adresse IP locale ou une plage sécurisée.  
     - **HTTP (port 80)** et/ou **HTTPS (port 443)** : Si l'application doit être accessible publiquement.  
     - **Custom TCP Rule (port 8000)** : Pour accéder directement à FastAPI si elle n'est pas déployée derrière un serveur proxy.  
   - Cliquez sur **Lancer l'Instance**.  

   Une fois l’instance lancée, notez son **ID**, son **Nom**, et son **Adresse IP publique**.

---

2. **Créer un Cluster ECS** :  
   - Accédez à la console AWS ECS.  
   - Cliquez sur **Clusters** > **Créer un Cluster**.  
   - Choisissez **EC2 Linux + Networking** (compatible avec votre instance EC2).  
   - **Configuration du Cluster** : 
     - **Nom** : Donnez un nom, comme `fastapi-ec2-cluster`.  
     - Configurez le type d'instance et le stockage selon vos besoins, puis cliquez sur **Créer**.  

---

3. **Créer une Définition de Tâche ECS** :  
   - Accédez à **Task Definitions** > **Créer une Nouvelle Tâche**.  
   - Sélectionnez **EC2** comme mode d’exécution.  
   - **Configuration de la Définition** :  
     - **Nom** : `fastapi-task`.  
     - **Roles IAM** : Attribuez un rôle avec les permissions suivantes :  
       - **`AmazonECSTaskExecutionRolePolicy`** (exécution des tâches ECS).  
       - **`AmazonRDSFullAccess`** (accès à la base de données RDS, si utilisé).  
       - Si vous n'avez pas de rôle existant, créez-en un via **IAM > Roles > Create Role**.  

   - **Ajouter un Conteneur** :  
     - Cliquez sur **Add Container** :  
       - **Nom du Conteneur** : `fastapi-app`.  
       - **Image** : Entrez l’image Docker que vous avez pushée, par exemple :  
         ```
         gboyou12/fastapi-aws-app:latest
         ```
       - **Port Mappings** : Configurez le port `8000` (ou selon les spécifications de votre application).  
       - **Variables d'Environnement** : Ajoutez les variables essentielles (`DATABASE_URL`, `SECRET_KEY`, etc.). Exemple :  
         ```
         DATABASE_URL=postgresql://user:password@rds-instance:5432/dbname  
         SECRET_KEY=your_secret_key  
         ```
       - Cliquez sur **Save** pour valider.  

---

4. **Déployer la Tâche ECS** :  
   - Retournez dans votre **Cluster ECS**.  
   - Cliquez sur l’onglet **Tasks**, puis sur **Run New Task**.  
   - Sélectionnez la définition de tâche créée (`fastapi-task`).  
   - Assurez-vous d’utiliser le bon réseau (VPC et sous-réseaux associés à votre instance EC2).  
   - Cliquez sur **Run Task**.  

---

5. **Accéder à l’Application** :  
   - **Vérifiez l’état de votre tâche** dans l’onglet **Tasks** du cluster ECS pour vous assurer qu'elle est bien en cours d’exécution.  
   - Allez dans la console EC2, sélectionnez votre instance, et copiez son **IP Publique** ou **Nom DNS**.  
   - Testez l’application FastAPI via le navigateur ou un outil comme `curl` :  
     ```
     http://<EC2-PUBLIC-IP>:8000/docs
     ```
     Remplacez `<EC2-PUBLIC-IP>` par l’adresse publique ou le DNS de votre instance.

---

5. **Déploiement de la tâche** :
   - Une fois la définition de la tâche créée, accédez à votre **Cluster ECS**.
   - Dans **Services**, cliquez sur **Create**.
   - **Service name** : Donnez un nom au service, par exemple `fastapi-ecs-service`.
   - **Nombre de tâches** : Choisissez le nombre d'instances (commencez par 1 pour tester).
   - **Déploiement** : Choisissez les options de déploiement selon votre besoin (par exemple, déploiement Blue/Green pour les environnements de production).
   
6. **Configurer le Load Balancer (optionnel)** :
   - Si vous souhaitez exposer votre application via un Load Balancer, sélectionnez **Application Load Balancer** pour acheminer le trafic vers vos instances ECS.
   - Configurez le port et les règles de routage pour rediriger le trafic vers le conteneur FastAPI sur le port 8000.

7. **Exécuter le Service** :
   - Lancez le service. ECS déploiera votre tâche et commencera à lancer le conteneur FastAPI.
   - Dans l'onglet **Tasks**, vous pourrez voir si le conteneur est lancé correctement.

---

### 3. Accéder à l'Application

- **Vérification des logs** : Consultez les logs de votre application dans la console ECS pour valider que l'application FastAPI démarre et se connecte à la base de données RDS.
- **URL du Load Balancer** : Si vous avez configuré un Load Balancer, utilisez son URL pour accéder à l’application.
- **Connexion directe** : Si vous avez activé l'accès public, utilisez l'adresse IP publique de la tâche ECS pour tester directement.

---

## 5. Tests et Validation

Après avoir déployé votre application FastAPI sur AWS, il est crucial de réaliser une série de tests et de validations pour s'assurer que tout fonctionne comme prévu. Cette étape garantit la fiabilité, la sécurité et la performance de votre application en production.

### 5.1 Lancement des Tests

#### 5.1.1 Déclencher un Build Jenkins via GitHub

1. **Configurer le Webhook GitHub pour Jenkins** :
   - Accédez à votre dépôt GitHub.
   - Allez dans **Settings** > **Webhooks** > **Add webhook**.
   - **Payload URL** : Entrez l'URL de votre serveur Jenkins, par exemple `http://your-jenkins-server/github-webhook/`.
   - **Content type** : Sélectionnez `application/json`.
   - **Which events would you like to trigger this webhook?** : Choisissez **Just the push event**.
   - Cliquez sur **Add webhook**.

2. **Configurer Jenkins pour Recevoir les Webhooks** :
   - Accédez à votre interface Jenkins.
   - Naviguez vers votre **Job Pipeline**.
   - Cliquez sur **Configurer**.
   - Dans la section **Build Triggers**, cochez **GitHub hook trigger for GITScm polling**.
   - Sauvegardez les modifications.

3. **Pousser du Code sur GitHub** :
   - Faites des modifications dans votre code localement.
   - Committez et poussez les changements :
     ```bash
     git add .
     git commit -m "Ajout de nouvelles fonctionnalités"
     git push origin main
     ```
   - Cette action déclenchera automatiquement le pipeline Jenkins via le webhook configuré.

#### 5.1.2 Vérifier le Pipeline CI/CD dans Jenkins

1. **Observer le Build en Cours** :
   - Accédez à votre job Jenkins.
   - Vous verrez un nouveau build démarrer automatiquement suite au push.
   - Cliquez sur le build en cours pour voir les logs en temps réel.

2. **Analyser les Logs** :
   - Assurez-vous que chaque étape (Build Docker Image, Push Docker Image, Deploy to AWS) se termine sans erreurs.
   - En cas d’échec, consultez les logs pour identifier et corriger les problèmes.

3. **Vérifier le Déploiement** :
   - Une fois le pipeline terminé, vérifiez que votre application est déployée sur AWS.
   - Accédez à l’URL de votre API pour confirmer que la nouvelle version est active.

### 5.2 Vérification des Migrations Alembic

#### 5.2.1 Appliquer les Migrations

1. **Vérifier le Script `entrypoint.sh`** :
   - Assurez-vous que votre script `entrypoint.sh` contient bien la commande pour appliquer les migrations Alembic :
     ```bash
     alembic upgrade head
     ```

2. **Déclencher le Déploiement** :
   - Poussez un commit qui inclut une nouvelle migration Alembic.
   - Par exemple, ajoutez une nouvelle colonne à un modèle :
     ```python
     # app/models/user.py
     from sqlalchemy import Column, Integer, String
     
     class User(Base):
         __tablename__ = 'users'
         id = Column(Integer, primary_key=True, index=True)
         name = Column(String, index=True)
         email = Column(String, unique=True, index=True)
         # Nouvelle colonne
         age = Column(Integer, nullable=True)
     ```
   - Générez une nouvelle migration :
     ```bash
     alembic revision --autogenerate -m "Ajout de la colonne age à User"
     git add alembic/versions/
     git commit -m "Ajout de la colonne age à User"
     git push origin main
     ```

3. **Vérifier l'Application des Migrations** :
   - Sur Jenkins, observez que le pipeline déclenche le script `entrypoint.sh` qui applique les migrations.
   - Accédez à votre base de données RDS PostgreSQL via un client SQL (comme pgAdmin ou psql) et vérifiez que la nouvelle colonne `age` a bien été ajoutée :
     ```sql
     \c cica_db
     \d users
     ```
   - Vous devriez voir la colonne `age` dans la table `users`.

#### 5.2.2 Gestion des Erreurs de Migration

1. **Analyser les Logs de Migration** :
   - Si une migration échoue, consultez les logs dans Jenkins pour identifier la cause.
   - Corrigez les scripts de migration ou le code source en conséquence et poussez les modifications à nouveau.

2. **Rollback des Migrations (si nécessaire)** :
   - En cas de problème critique, vous pouvez annuler la dernière migration :
     ```bash
     alembic downgrade -1
     ```
   - Ajustez votre code et vos migrations, puis redéployez.

### 5.3 Contrôle d'Accès CORS et SMTP

#### 5.3.1 Tester les Configurations CORS

1. **Vérifier les Origines CORS** :
   - Assurez-vous que les origines autorisées dans votre fichier `.env` sont correctes :
     ```env
     ORIGIN_CORS="http://localhost:8080,http://127.0.0.1:8080,http://localhost:8000,http://127.0.0.1:8000"
     ```

2. **Tester les Requêtes CORS** :
   - Utilisez un client HTTP (comme Postman ou un navigateur web) pour envoyer des requêtes à votre API depuis une origine autorisée et une origine non autorisée.
   - Exemple avec **curl** depuis une origine autorisée :
     ```bash
     curl -H "Origin: http://localhost:8080" -H "Access-Control-Request-Method: GET" -X OPTIONS --verbose http://your-api-endpoint.com
     ```
   - Vous devriez recevoir des en-têtes CORS appropriés (`Access-Control-Allow-Origin`).

   - Exemple depuis une origine non autorisée :
     ```bash
     curl -H "Origin: http://unauthorized-domain.com" -H "Access-Control-Request-Method: GET" -X OPTIONS --verbose http://your-api-endpoint.com
     ```
   - Vous ne devriez pas recevoir les en-têtes CORS autorisant la requête.

3. **Résoudre les Problèmes CORS** :
   - Si des requêtes légitimes sont bloquées, vérifiez et ajustez la variable `ORIGIN_CORS` dans votre fichier `.env`.
   - Redéployez l’application via Jenkins pour appliquer les modifications.

#### 5.3.2 Tester l'Envoi de Mails via SMTP

1. **Configurer un Endpoint de Test pour l'Envoi de Mail** :
   - Créez une route temporaire dans votre application FastAPI pour tester l'envoi de mail.
     ```python
     # app/api/test_mail.py
     from fastapi import APIRouter, Depends
     from app.core.mail import send_test_email
     
     router = APIRouter()
     
     @router.get("/test-mail")
     async def test_mail():
         await send_test_email(
             subject="Test Email",
             recipient="admin@example.com",
             body="Ceci est un email de test."
         )
         return {"message": "Email envoyé avec succès"}
     ```
   - Incluez cette route dans votre `main.py` :
     ```python
     from fastapi import FastAPI
     from app.api import router
     from app.api.test_mail import router as test_mail_router
     
     app = FastAPI()
     
     app.include_router(router)
     app.include_router(test_mail_router)
     ```

2. **Redéployer l'Application** :
   - Poussez les modifications sur GitHub pour déclencher le pipeline Jenkins et déployer la nouvelle version.

3. **Accéder à l'Endpoint de Test** :
   - Ouvrez un navigateur ou utilisez **curl** pour accéder à l’endpoint de test :
     ```bash
     curl http://your-api-endpoint.com/test-mail
     ```
   - Vérifiez que la réponse indique que l'email a été envoyé avec succès.

4. **Vérifier la Réception de l'Email** :
   - Connectez-vous à l'adresse email du destinataire (`admin@example.com`) et confirmez la réception de l'email de test.

5. **Analyser les Logs en Cas d'Échec** :
   - Si l'email n'est pas reçu, consultez les logs de l'application via la console ECS ou Jenkins pour identifier les erreurs liées à SMTP.
   - Assurez-vous que les variables SMTP dans le fichier `.env` sont correctes :
     ```env
     SMTP_SERVER=smtp.gmail.com
     SMTP_PORT=587
     SMTP_USERNAME=
     SMTP_PASSWORD=
     SENDER_EMAIL=
     ```
   - **Note** : Si vous utilisez Gmail, assurez-vous que les paramètres de sécurité permettent l'accès via SMTP (par exemple, activer les applications moins sécurisées ou utiliser un mot de passe d'application).

6. **Résoudre les Problèmes SMTP** :
   - Corrigez les configurations SMTP en fonction des erreurs trouvées dans les logs.
   - Redéployez l’application après avoir apporté les modifications nécessaires.

### 5.4 Surveillance et Monitoring (Optionnel mais Recommandé)

Pour garantir la fiabilité continue de votre application, il est recommandé de mettre en place des outils de surveillance et de monitoring.

1. **Configurer Amazon CloudWatch** :
   - Utilisez **CloudWatch Logs** pour collecter et analyser les logs de votre application ECS.
   - Configurez des métriques personnalisées pour surveiller la performance de l’application (CPU, mémoire, latence des requêtes, etc.).

2. **Configurer des Alertes** :
   - Définissez des alarmes CloudWatch pour être alerté en cas de comportements anormaux (par exemple, une augmentation soudaine des erreurs 500).

3. **Utiliser des Outils de Monitoring Externes** :
   - Intégrez des outils comme **Prometheus** et **Grafana** pour une surveillance avancée et des tableaux de bord personnalisés.
   - Utilisez **Sentry** pour le suivi des erreurs et des exceptions dans votre application FastAPI.

### 5.5 Validation Finale

1. **Effectuer des Tests de Charge** :
   - Utilisez des outils comme **Apache JMeter** ou **Locust** pour simuler des charges de trafic et évaluer la performance de votre application sous pression.

2. **Tester les Fonctionnalités de l'Application** :
   - Effectuez des tests fonctionnels manuels ou automatisés (avec **pytest** et **Selenium**) pour vérifier que toutes les fonctionnalités de votre application fonctionnent correctement.

3. **Vérifier la Sécurité** :
   - Assurez-vous que votre application est sécurisée en vérifiant les configurations CORS, les politiques de sécurité, et en utilisant des outils comme **OWASP ZAP** pour des scans de sécurité.

---

En suivant ces étapes, vous pouvez déployer votre application FastAPI sur AWS avec PostgreSQL, Docker, Jenkins et GitHub intégrés.

---

Pour obtenir le lien (URL) de votre API FastAPI déployée sur AWS, voici les étapes à suivre, selon votre configuration :

### 1. Si vous utilisez un Load Balancer

Si vous avez configuré un **Load Balancer** (comme un **Application Load Balancer**), c’est la méthode recommandée pour accéder à votre application :

1. **Accédez à la console Amazon EC2** :
   - Dans le menu de gauche, allez dans **Load Balancers**.
   - Sélectionnez votre Load Balancer, qui devrait être associé à votre cluster ECS.

2. **Copiez le DNS du Load Balancer** :
   - Dans la section **Description**, repérez le champ **DNS name**.
   - Ce DNS représente l'URL publique de votre application. Par exemple :
     ```
     http://my-load-balancer-1234567890.us-west-2.elb.amazonaws.com
     ```

3. **Ajoutez le port de votre application si nécessaire** :
   - Si votre application FastAPI écoute sur le port 8000, l'URL finale serait :
     ```
     http://my-load-balancer-1234567890.us-west-2.elb.amazonaws.com:8000
     ```

---

### 2. Si vous n’utilisez pas de Load Balancer (Accès direct aux tâches ECS)

1. **Accédez à la console Amazon ECS** :
   - Dans le menu de gauche, cliquez sur **Clusters**.
   - Sélectionnez votre cluster ECS, puis allez dans l’onglet **Tasks**.

2. **Vérifiez l’adresse IP publique de la tâche** :
   - Cliquez sur la tâche active pour voir ses détails.
   - Dans **Network** (Réseau), si votre tâche est dans un sous-réseau public, vous verrez une **adresse IP publique**.

3. **Utilisez l’adresse IP publique** :
   - Vous pouvez accéder à l’application en utilisant cette adresse IP, en ajoutant le port si nécessaire :
     ```
     http://<public-ip>:8000
     ```

> ⚠️ **Remarque** : Assurez-vous que le **groupe de sécurité** associé à la tâche ECS autorise le trafic entrant sur le port 8000 pour permettre l'accès externe.

---

### 3. Accès via un Domaine Personnalisé (Optionnel)

Si vous avez un domaine personnalisé :

1. **Configurez un enregistrement CNAME ou A Record** dans votre service de domaine (comme Route 53, si vous utilisez AWS).
2. Pointez ce domaine vers le DNS de votre Load Balancer ou l’adresse IP publique de votre tâche.
3. Vous pourrez ensuite accéder à votre API avec votre propre domaine :
   ```
   http://api.mondomaine.com:8000
   ```
