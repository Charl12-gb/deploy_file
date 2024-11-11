Il est tout à fait possible d'utiliser Docker pour déployer une application FastAPI sur Heroku, y compris avec une base de données PostgreSQL et des migrations gérées par Alembic. L'utilisation de Docker permet d'isoler l'application et de garantir que toutes les dépendances sont installées et configurées correctement. Voici comment procéder.

### Étapes de déploiement avec Docker

#### 1. Configurer Docker pour FastAPI
Créez un fichier `Dockerfile` à la racine de votre projet pour définir l'image Docker de votre application :

```dockerfile
# Dockerfile
# Utiliser une image officielle Python pour construire l'image de base
FROM python:3.10-slim

# Définir le répertoire de travail
WORKDIR /app

# Copier les fichiers de l'application dans l'image Docker
COPY . .

# Installer les dépendances
RUN pip install --no-cache-dir -r requirements.txt

# Exposer le port sur lequel Uvicorn sera disponible
EXPOSE 8000

# Commande de démarrage pour Uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### 2. Configurer le fichier `docker-compose.yml`
Ce fichier est utile pour orchestrer le conteneur de l’application et celui de PostgreSQL. Créez un fichier `docker-compose.yml` :

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:13
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydatabase
    volumes:
      - postgres_data:/var/lib/postgresql/data

  web:
    build: .
    command: ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://myuser:mypassword@db:5432/mydatabase
    depends_on:
      - db

volumes:
  postgres_data:
```

Le fichier `docker-compose.yml` définit deux services :
- `db` : un service PostgreSQL pour la base de données.
- `web` : l’application FastAPI qui dépend de PostgreSQL pour se lancer.

> **Remarque :** La variable d’environnement `DATABASE_URL` est définie pour connecter SQLAlchemy à PostgreSQL. Cette variable est ensuite utilisée par Alembic pour les migrations.

#### 3. Configurer Alembic pour les migrations
Dans `alembic.ini`, assurez-vous d’utiliser la variable `DATABASE_URL` pour la connexion :

```ini
# alembic.ini
sqlalchemy.url = ${DATABASE_URL}
```

#### 4. Créer l’image Docker et tester en local
Avant de déployer, testez votre configuration Docker en local. Exécutez la commande suivante :

```bash
docker-compose up --build
```

L’application devrait maintenant être accessible sur `http://localhost:8000`.

#### 5. Connecter l’application à Heroku
1. **Connexion à Heroku** : Si ce n’est pas déjà fait, connectez-vous à Heroku avec `heroku login`.
2. **Créer une application Heroku** : Utilisez `heroku create <app_name>` pour créer une nouvelle application.
3. **Ajouter un add-on PostgreSQL** : Sur Heroku, ajoutez une base de données PostgreSQL avec cette commande :

   ```bash
   heroku addons:create heroku-postgresql:hobby-dev -a <app_name>
   ```

   Cette commande fournit également une variable d’environnement `DATABASE_URL` qui sera utilisée par l’application et Alembic.

#### 6. Déployer sur Heroku
Heroku permet de déployer une application Docker en utilisant un fichier `Dockerfile` directement, ou en poussant l’image Docker construite.

1. **Pousser l’image Docker vers Heroku** :

   ```bash
   heroku container:push web -a <app_name>
   ```

2. **Déployer l’image** :

   ```bash
   heroku container:release web -a <app_name>
   ```

3. **Migrer la base de données** : Utilisez Alembic pour appliquer les migrations de la base de données PostgreSQL en accédant au conteneur :

   ```bash
   heroku run alembic upgrade head -a <app_name>
   ```

#### 7. Accéder à votre application
Votre application FastAPI devrait maintenant être disponible via l’URL fournie par Heroku (par exemple, `https://<app_name>.herokuapp.com`).

### Résumé des fichiers importants

- `Dockerfile` : Définit l'image Docker de l’application.
- `docker-compose.yml` : Orchestration des conteneurs PostgreSQL et FastAPI en local.
- `alembic.ini` : Configure Alembic pour se connecter à la base de données via `DATABASE_URL`.

Avec cette configuration, vous pouvez déployer facilement votre application FastAPI sur Heroku en utilisant Docker, Alembic pour les migrations, et PostgreSQL comme base de données.
