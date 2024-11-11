Pour mettre en place et gérer cette tâche cron dans votre application Django, vous pouvez suivre les étapes ci-dessous. Nous allons voir comment exécuter votre tâche toutes les 5 minutes, la supprimer et la programmer pour une exécution quotidienne.

### 1. Utilisation du `django-crontab` pour gérer les tâches cron

Pour faciliter l'ajout, la gestion, et la suppression de tâches cron, nous utiliserons le package `django-crontab`.

#### Installation

Dans votre environnement virtuel, installez `django-crontab` :
```bash
pip install django-crontab
```

#### Configuration de `django-crontab`

1. Ajoutez `django_crontab` à la liste des applications installées dans `settings.py` de votre projet :
   ```python
   INSTALLED_APPS = [
       # Autres applications
       'django_crontab',
   ]
   ```

2. Définissez les commandes cron dans `settings.py`. Pour exécuter votre tâche toutes les 5 minutes, puis la changer pour une fois par jour après vérification :

   - Créez d’abord une fonction qui exécutera le code de votre tâche :
     ```python
     # Dans my_api/cron.py
     from my_api.Views.Document.alert_config import getDocumentNotValidate
     from my_api.Utils.test_email import send_simple_email

     def my_cron_job():
         print("Cron job started.")
         getDocumentNotValidate()
         send_simple_email()
     ```

   - Ajoutez une tâche cron dans `settings.py` pour appeler cette fonction toutes les 5 minutes :
     ```python
     CRONJOBS = [
         ('*/5 * * * *', 'my_api.cron.my_cron_job'),  # Toutes les 5 minutes
     ]
     ```

3. Enregistrez la tâche cron avec la commande suivante :
   ```bash
   python manage.py crontab add
   ```

4. **Vérifiez le bon fonctionnement de la tâche cron** en consultant les logs ou en vérifiant les actions dans votre application.

5. Une fois la vérification terminée, modifiez la planification pour une exécution quotidienne en mettant à jour `CRONJOBS` dans `settings.py` :
   ```python
   CRONJOBS = [
       ('0 0 * * *', 'my_api.cron.my_cron_job'),  # Tous les jours à minuit
   ]
   ```

6. Après avoir modifié la configuration, appliquez le changement en retirant et en ré-ajoutant les tâches cron :
   ```bash
   python manage.py crontab remove
   python manage.py crontab add
   ```

### 2. Vérification et gestion des tâches cron

Pour consulter les tâches cron ajoutées, utilisez :
```bash
python manage.py crontab show
```

Pour supprimer toutes les tâches cron :
```bash
python manage.py crontab remove
```

### 3. Alternatives et options

**Utilisation de `APScheduler` directement dans Django** : Si vous préférez continuer avec `APScheduler`, veillez à utiliser une approche sans blocage et à lancer `BlockingScheduler` en arrière-plan lorsque le serveur est déployé. Toutefois, `django-crontab` est recommandé pour des tâches simples et intégrées au système cron de Linux. 

### Notes

- **Débogage** : Utilisez les fichiers de log pour capturer les messages de débogage et surveiller les exécutions des tâches cron.
- **Permissions** : Assurez-vous que votre utilisateur a les permissions nécessaires pour ajouter et exécuter les tâches cron sur le système.


### Détails

Les tâches cron utilisent une syntaxe bien définie pour planifier des moments d'exécution. Voici les différents formats que vous pouvez utiliser :

### Syntaxe Cron : 

La syntaxe cron suit ce format :
```
* * * * * commande
| | | | |
| | | | └── Jour de la semaine (0 à 7) (0 ou 7 = Dimanche)
| | | └──── Mois (1 - 12)
| | └────── Jour du mois (1 - 31)
| └──────── Heure (0 - 23)
└────────── Minute (0 - 59)
```

### Exemples de Programmations Courantes

| Expression Cron       | Explication                                      |
|-----------------------|--------------------------------------------------|
| `*/5 * * * *`         | Toutes les 5 minutes                             |
| `0 * * * *`           | À chaque heure, à la minute 0                    |
| `0 0 * * *`           | Une fois par jour, à minuit                      |
| `0 12 * * *`          | Une fois par jour, à midi                        |
| `0 0 * * 0`           | Une fois par semaine, le dimanche à minuit       |
| `0 0 1 * *`           | Une fois par mois, le premier jour à minuit      |
| `*/10 0 * * *`        | Toutes les 10 minutes entre minuit et 1h du matin|
| `15 14 * * *`         | Tous les jours à 14h15                           |
| `0 18 * * 1-5`        | En semaine (lundi à vendredi) à 18h              |
| `0 8,16 * * *`        | Deux fois par jour, à 8h et 16h                  |
| `*/15 9-17 * * *`     | Toutes les 15 minutes entre 9h et 17h chaque jour|
| `0 0-5 14 * *`        | Chaque heure entre minuit et 5h le 14e jour du mois|

### Exemples Avancés

- **Toutes les 15 minutes, en semaine uniquement** :
  ```cron
  */15 * * * 1-5
  ```
- **Tous les dimanches à 23h30** :
  ```cron
  30 23 * * 0
  ```
- **Chaque premier du mois à 1h du matin** :
  ```cron
  0 1 1 * *
  ```
- **Tous les jours ouvrables à 8h30** :
  ```cron
  30 8 * * 1-5
  ```
  
En utilisant ces formats, vous pouvez ajuster votre tâche cron en fonction des besoins de votre application.
