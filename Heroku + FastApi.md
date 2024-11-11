Déployer une application FastAPI sur Heroku avec une base de données PostgreSQL en utilisant Alembic pour la gestion des migrations et Uvicorn comme serveur d’application peut être réalisé en suivant ces étapes :

### Prérequis
- Avoir un compte Heroku
- Installer le [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)
- Avoir une application FastAPI prête à être déployée
- Avoir `uvicorn`, `alembic`, et `asyncpg` (si PostgreSQL) installés dans votre projet

### Étapes de déploiement

#### 1. Créer un projet FastAPI
Assurez-vous d’avoir un fichier `main.py` contenant votre application FastAPI :

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello, world!"}
```

#### 2. Installer les dépendances
Dans votre environnement virtuel, installez les dépendances nécessaires avec `pip` :

```bash
pip install fastapi uvicorn[standard] sqlalchemy asyncpg alembic psycopg2-binary
```

Ajoutez-les au fichier `requirements.txt` :

```bash
pip freeze > requirements.txt
```

#### 3. Configurer Alembic pour les migrations
Initialisez Alembic avec la commande suivante pour créer un répertoire `alembic` avec les fichiers de configuration de migration :

```bash
alembic init alembic
```

Dans le fichier `alembic.ini`, configurez la chaîne de connexion de votre base de données :

```ini
# alembic.ini
sqlalchemy.url = postgresql+asyncpg://<username>:<password>@<hostname>:<port>/<database>
```

Vous pouvez également gérer la chaîne de connexion via des variables d’environnement pour éviter de la rendre publique dans le code source.

#### 4. Configurer votre fichier `Procfile`
Dans le fichier `Procfile` (à la racine du projet), indiquez à Heroku comment exécuter l’application avec Uvicorn :

```bash
web: uvicorn main:app --host=0.0.0.0 --port=${PORT:-5000}
```

Heroku attribue le port via la variable d’environnement `PORT`, d'où l'utilisation de `--port=${PORT:-5000}`.

#### 5. Configurer les variables d’environnement dans `.env`
Créez un fichier `.env` (optionnel) pour stocker les configurations sensibles comme la chaîne de connexion de votre base de données et toute autre variable :

```env
DATABASE_URL=postgresql+asyncpg://<username>:<password>@<hostname>:<port>/<database>
```

#### 6. Initialiser un dépôt Git et créer l’application sur Heroku

1. Initialisez un dépôt Git si ce n’est pas déjà fait :
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   ```

2. Connectez-vous à Heroku via le CLI et créez une nouvelle application :
   ```bash
   heroku login
   heroku create <your-app-name>
   ```

3. Ajoutez l’add-on PostgreSQL fourni par Heroku :
   ```bash
   heroku addons:create heroku-postgresql:hobby-dev
   ```

4. Récupérez l’URL de la base de données configurée automatiquement par Heroku :
   ```bash
   heroku config:get DATABASE_URL
   ```

5. Dans le fichier `alembic.ini` ou dans votre code de configuration, remplacez la chaîne de connexion par la valeur `DATABASE_URL` obtenue de Heroku.

#### 7. Migrer la base de données avec Alembic

1. Créez un modèle pour votre base de données dans votre projet.
   
2. Générez une migration Alembic pour ce modèle :
   ```bash
   alembic revision --autogenerate -m "Initial migration"
   ```

3. Appliquez la migration :
   ```bash
   alembic upgrade head
   ```

#### 8. Déployer sur Heroku

1. Poussez le code sur Heroku :
   ```bash
   git push heroku main
   ```

2. Assurez-vous que tout fonctionne correctement en vérifiant les logs :
   ```bash
   heroku logs --tail
   ```

#### 9. Accéder à votre application
Votre application FastAPI est maintenant déployée sur Heroku ! Vous pouvez accéder à l’URL de l’application générée par Heroku : `https://<your-app-name>.herokuapp.com`.

### Résumé des fichiers importants

- `main.py` : votre application FastAPI.
- `Procfile` : fichier pour spécifier la commande de démarrage avec Uvicorn.
- `requirements.txt` : les dépendances Python.
- `.env` ou configuration des variables d’environnement sur Heroku (ex. `DATABASE_URL`).
- `alembic.ini` et les fichiers dans le dossier `alembic` : pour les migrations de base de données.

Heroku prendra en charge automatiquement la configuration des serveurs et la connexion à la base de données PostgreSQL grâce à l’add-on.
