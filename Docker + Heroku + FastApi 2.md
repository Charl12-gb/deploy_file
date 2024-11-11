Déployer une application FastAPI sur Heroku avec Docker en utilisant une base de données PostgreSQL, Alembic pour les migrations, et Uvicorn pour le serveur peut se faire en suivant ces étapes. Nous utiliserons Docker pour construire et déployer l'application sur Heroku, en tirant parti de son intégration Docker.

### Prérequis
1. **Compte Heroku** et **Heroku CLI**.
2. **Docker** et **Docker Compose** installés localement.
3. Une application FastAPI avec une base de données PostgreSQL configurée.

### Étapes de déploiement

#### 1. Configurer l'application FastAPI
Assurez-vous que votre application a la structure suivante :

- `main.py`: Contient votre code FastAPI.
- `Dockerfile`: Configuration Docker pour le conteneur.
- `docker-compose.yml`: Fichier de composition pour les services.
- `entrypoint.sh`: Script pour le démarrage du conteneur.
- `alembic.ini` et dossier `alembic`: Pour les migrations de la base de données.
- `.env`: Variables d’environnement pour la configuration.

#### 2. Créer le Dockerfile

Voici le `Dockerfile` configuré pour utiliser Uvicorn et PostgreSQL :

```dockerfile
# Dockerfile

# Utilisation de l'image de base Python slim
FROM python:3.12-slim

# Installation des dépendances système
RUN apt-get update && apt-get install -y \
    libpq-dev \
    gcc \
    netcat-openbsd \
    && rm -rf /var/lib/apt/lists/*

# Définir le répertoire de travail
WORKDIR /app

# Copier les dépendances Python
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copier le reste du code de l'application
COPY . .

# Copier le script d'entrée
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

# Exposer le port 8000
EXPOSE 8000

# Utiliser entrypoint.sh comme point d'entrée
ENTRYPOINT ["/app/entrypoint.sh"]
```

#### 3. Créer le fichier `docker-compose.yml`

Le fichier `docker-compose.yml` configure deux services : l'application FastAPI et la base de données PostgreSQL.

```yaml
# docker-compose.yml

version: '3.8'

services:
  app:
    build: .
    container_name: fastapi_app
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      - db
    restart: always
    command: ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

  db:
    image: postgres:13
    container_name: postgres_db
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

#### 4. Créer le script `entrypoint.sh`

Le script `entrypoint.sh` vérifie que la base de données est prête avant de démarrer Uvicorn :

```bash
#!/bin/sh

# Attendre que la base de données soit prête
echo "Waiting for PostgreSQL to start..."
while ! nc -z db 5432; do
  sleep 0.1
done

echo "PostgreSQL started"

# Lancer les migrations Alembic
alembic upgrade head

# Démarrer l'application avec Uvicorn
exec "$@"
```

#### 5. Configurer les migrations avec Alembic

Initialisez Alembic et configurez `alembic.ini` pour pointer vers PostgreSQL en utilisant les variables d’environnement. Par exemple :

```ini
# alembic.ini

sqlalchemy.url = postgresql+asyncpg://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
```

#### 6. Créer le fichier `.env`

Le fichier `.env` contiendra vos variables d’environnement :

```env
DB_USER=your_postgres_user
DB_PASSWORD=your_postgres_password
DB_NAME=your_database_name
```

#### 7. Construire et tester en local

Utilisez Docker Compose pour démarrer l'application localement :

```bash
docker-compose up --build
```

#### 8. Préparer le déploiement sur Heroku

1. Connectez-vous à Heroku et créez une nouvelle application.
   
2. Activez l’intégration Docker dans Heroku :
   
   ```bash
   heroku stack:set container -a <nom_de_votre_application>
   ```

3. Poussez votre code vers Heroku en utilisant Docker :

   ```bash
   heroku container:push web -a <nom_de_votre_application>
   ```

4. Relâchez l'image Docker vers Heroku :

   ```bash
   heroku container:release web -a <nom_de_votre_application>
   ```

#### 9. Configurer la base de données sur Heroku

Heroku propose un addon pour PostgreSQL que vous pouvez ajouter en utilisant :

```bash
heroku addons:create heroku-postgresql:hobby-dev -a <nom_de_votre_application>
```

Cela définira automatiquement les variables d’environnement `DATABASE_URL` dans l’environnement Heroku. Mettez à jour `alembic.ini` pour utiliser cette variable d’environnement :

```ini
sqlalchemy.url = ${DATABASE_URL}
```

#### 10. Exécuter les migrations Alembic sur Heroku

Pour exécuter les migrations sur la base de données PostgreSQL de Heroku :

```bash
heroku run "alembic upgrade head" -a <nom_de_votre_application>
```

#### 11. Accéder à l'application

Votre application FastAPI est maintenant en cours d'exécution sur Heroku, et vous pouvez y accéder en ouvrant l'URL de l'application fournie par Heroku.
