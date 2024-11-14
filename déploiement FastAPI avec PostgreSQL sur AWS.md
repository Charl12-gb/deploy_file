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
SMTP_USERNAME=oriontestorion@gmail.com
SMTP_PASSWORD=dszoyexnxznsaxfo
SENDER_EMAIL=oriontestorion@gmail.com

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

#### 4.1 Dépôt GitHub et Jenkins

1. **Dépôt GitHub** : Créez un dépôt pour votre projet et poussez-y votre code.
2. **Jenkins** : Configurez un pipeline dans Jenkins pour automatiser le processus de CI/CD.

#### 4.2 Jenkinsfile

Voici un exemple de fichier `Jenkinsfile` pour automatiser la construction et le déploiement avec Jenkins et Docker.

```groovy
pipeline {
    agent any
    environment {
        DOCKER_HUB_REPO = 'votre_nom_utilisateur/fastapi-aws-app'
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    }
    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_HUB_REPO}:${env.BUILD_ID}")
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', 'docker-hub-credentials') {
                        dockerImage.push()
                    }
                }
            }
        }
        stage('Deploy to AWS') {
            steps {
                sh 'aws ecs update-service --cluster my-cluster --service my-service --force-new-deployment'
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
```

Voici des instructions détaillées pour configurer et déployer votre application FastAPI sur AWS en utilisant Amazon RDS pour PostgreSQL et Amazon ECS pour héberger votre application Docker.

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
   - Choisissez une classe d’instance adaptée aux besoins de votre application (par exemple, `db.t3.micro` pour une petite application).

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

#### 2. Configurer Amazon ECS pour héberger l’application Docker

1. **Accédez à Amazon ECS** dans la console AWS.

2. **Créer un Cluster** :
   - Cliquez sur **Clusters** dans le menu latéral, puis sur **Create Cluster**.
   - Sélectionnez **EC2 Linux + Networking** pour un déploiement géré, ou **Fargate** pour un service entièrement managé sans serveur.
   - Suivez les étapes de configuration, puis cliquez sur **Create** pour finaliser.

3. **Configurer une Tâche ECS** (Task Definition) :
   - Accédez à **Task Definitions** et cliquez sur **Create New Task Definition**.
   - Choisissez **Fargate** ou **EC2** selon votre choix lors de la création du cluster.
   
4. **Configuration de la tâche** :
   - **Nom** : Donnez un nom significatif, comme `fastapi-task`.
   - **Roles IAM** : Attribuez un rôle IAM permettant à votre tâche d'accéder aux services nécessaires (comme RDS). Si vous n’avez pas de rôle IAM existant, créez-en un avec les autorisations `AmazonECSTaskExecutionRolePolicy` et `AmazonRDSFullAccess`.
   - **Conteneurs** :
     - Cliquez sur **Add container** pour configurer votre conteneur.
     - **Nom du conteneur** : `fastapi-app`.
     - **Image** : Saisissez le nom de l'image Docker (par exemple, `votre_nom_utilisateur/fastapi-aws-app:latest` depuis Docker Hub, ou l'URI ECR si vous utilisez Amazon ECR).
     - **Port mappings** : Configurez le port 8000 pour l'exposer aux autres services.
     - **Variables d'environnement** : Ajoutez vos variables d'environnement nécessaires (comme `DATABASE_URL`, `SECRET_KEY`, etc.), correspondant à celles de votre `.env`.

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
     SMTP_USERNAME=oriontestorion@gmail.com
     SMTP_PASSWORD=dszoyexnxznsaxfo
     SENDER_EMAIL=oriontestorion@gmail.com
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
