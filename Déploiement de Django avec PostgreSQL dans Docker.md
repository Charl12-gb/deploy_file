# Documentation : Déploiement de Django avec PostgreSQL dans Docker

## Prérequis

- **Docker** et **Docker Compose** doivent être installés sur votre machine. Pour vérifier les installations, exécutez :
  ```bash
  docker --version
  docker-compose --version
  ```

## Structure du Projet

Votre projet Django doit inclure les fichiers suivants pour une intégration complète avec Docker et PostgreSQL :

- **Dockerfile** : Décrit l'image de votre application Django.
- **docker-compose.yml** : Configure et lie le service Django (web) au service PostgreSQL (db).
- **requirements.txt** : Liste des dépendances Python de votre projet, y compris `Django` et `psycopg2-binary`.
- **settings.py** : Configure la base de données PostgreSQL dans Django.

## Étape 1 : Configuration de Docker

### 1.1 Créer le fichier `Dockerfile`

Dans le répertoire racine de votre projet, créez un fichier `Dockerfile` :

```dockerfile
# Utilise une image Python 3.10
FROM python:3.10-slim

# Définir le répertoire de travail
WORKDIR /app

# Copier et installer les dépendances
COPY requirements.txt /app/
RUN pip install --no-cache-dir -r requirements.txt

# Copier le code de l'application
COPY . /app/

# Exposer le port par défaut de Django
EXPOSE 8000

# Démarrer le serveur Django
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

### 1.2 Créer le fichier `docker-compose.yml`

Le fichier `docker-compose.yml` lie le conteneur Django (`web`) avec le conteneur PostgreSQL (`db`). Voici un exemple de configuration :

```yaml
version: '3.8'

services:
  db:
    image: postgres:13
    environment:
      POSTGRES_DB: mydatabase
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - django_network

  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    environment:
      - DJANGO_SETTINGS_MODULE=monprojet.settings
    depends_on:
      - db
    networks:
      - django_network

volumes:
  postgres_data:

networks:
  django_network:
```

### 1.3 Mettre à jour les dépendances

Dans le fichier `requirements.txt`, ajoutez les dépendances nécessaires, notamment `Django` et `psycopg2-binary` :

```
Django>=3.2
psycopg2-binary
```

## Étape 2 : Configurer Django pour utiliser PostgreSQL

Dans `settings.py`, mettez à jour la configuration de la base de données pour utiliser PostgreSQL, comme indiqué ci-dessous :

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydatabase',
        'USER': 'myuser',
        'PASSWORD': 'mypassword',
        'HOST': 'db',
        'PORT': '5432',
    }
}
```

> **Note** : Les valeurs `NAME`, `USER`, `PASSWORD`, et `HOST` doivent correspondre aux valeurs du fichier `docker-compose.yml`.

## Étape 3 : Construction et Lancement des Conteneurs

Dans le répertoire contenant les fichiers `Dockerfile` et `docker-compose.yml`, exécutez les commandes suivantes :

### 3.1 Construire et démarrer les conteneurs

```bash
docker-compose up --build
```

Cela :
- Construit les images Docker définies dans `Dockerfile` et `docker-compose.yml`.
- Démarre les conteneurs.

### 3.2 Exécuter les migrations

Pour exécuter les migrations de la base de données, utilisez :

```bash
docker-compose run web python manage.py migrate
```

### 3.3 Ajouter des données initiales

Pour ajouter des données initiales, utilisez les commandes suivantes :

```bash
docker-compose run web python manage.py seed-app
docker-compose run web python manage.py seed-permissions
docker-compose run web python manage.py seed-superadmin
```

> **Note** : Remplacez `seed-app`, `seed-permissions`, et `seed-superadmin` par les commandes Django spécifiques à votre projet pour initialiser les données.

## Étape 4 : Accéder à l'API

Votre API sera accessible à l'adresse suivante :

```
http://localhost:8000
```

### Vérification des logs

Pour surveiller les logs de Django, exécutez la commande suivante :

```bash
docker-compose logs -f web
```

## Étape 5 : Arrêter les Conteneurs

Pour arrêter et supprimer les conteneurs, réseaux et volumes, exécutez :

```bash
docker-compose down
```

## Récapitulatif des Commandes

| Commande                                 | Description                                           |
|------------------------------------------|-------------------------------------------------------|
| `docker-compose up --build`              | Démarrer les conteneurs avec construction de l'image. |
| `docker-compose run web python manage.py migrate` | Exécuter les migrations Django.                |
| `docker-compose run web python manage.py <commande>` | Lancer une commande Django personnalisée.       |
| `docker-compose logs -f web`             | Afficher les logs en temps réel pour Django.          |
| `docker-compose down`                    | Arrêter et supprimer les conteneurs et volumes.       |

---

Ce document devrait vous guider à travers la configuration et le lancement de votre API Django dans Docker avec PostgreSQL.
