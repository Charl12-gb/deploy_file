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

#### 4.3 Déploiement sur AWS ECS et RDS

1. **Configurer RDS PostgreSQL** : Créez une instance PostgreSQL dans RDS et mettez à jour l’URL de la base de données dans `.env`.
2. **Configurer ECS** : Créez un cluster et une tâche ECS pour héberger l’application Docker.

---

### 5. Tests et Validation

1. **Lancement des Tests** : Déclenchez un build Jenkins en poussant du code sur GitHub et vérifiez que le pipeline CI/CD fonctionne.
2. **Vérification des Migrations** : Confirmez que les migrations Alembic s’appliquent correctement dans la base de données RDS.
3. **Contrôle d'accès CORS et SMTP** : Testez les variables CORS et l’envoi de mail avec les paramètres SMTP.

---

En suivant ces étapes, vous pouvez déployer votre application FastAPI sur AWS avec PostgreSQL, Docker, Jenkins et GitHub intégrés.
