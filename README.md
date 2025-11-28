# Projet Données massives et Cloud
Dans le cadre du projet Données massives et cloud, nous avons déployés un prototype de réseau social au lien suivant:  
https://tiny-insta-474215.ew.r.appspot.com/  
On aimerait alors évaluer sa performance lorsque la quantité de données qui lui sont associé croît.  

# Mise en oeuvre
On dispose de la commande
```
python massive-gcp/seed.py --users x1 --posts x2 --follows-min x3 --follows-max x3
```
Cette commande va générer x1 utilisateurs, x2 posts, et chaque user va suivre x3 autres user. Il est important de préciser que on ne peut pas associer un nombre exact de post à chaque utilisateur.
Si on veut 1000 users avec chacun 50 posts, on génèrera 50000 posts mais on ne peut pas garantir que chaque user aura exactement 50 posts.

Pour évaluer la performance, on utilise locust. Voici le script utilisé:
```
from locust import HttpUser, task, events
from gevent.event import Event
from itertools import count

start_event = Event()
user_counter = count(1)
finished_users = 0

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    global finished_users
    finished_users = 0
    print("Test started")

@events.spawning_complete.add_listener
def on_spawning_complete(user_count, **kwargs):
    print(f"All {user_count} users spawned")
    start_event.set() 

class TimelineUser(HttpUser):
    wait_time = lambda self: 999999  # empêche répétition

    def on_start(self):
        self.user_id = next(user_counter)
        self.username = f"user{self.user_id}"
        self.has_fired = False
        start_event.wait()

    @task
    def fire_once(self):
        global finished_users

        if not self.has_fired:
            self.has_fired = True

            url = f"/api/timeline?user={self.username}"
            self.client.get(url, name="/api/timeline")

            finished_users += 1

            total_users = self.environment.runner.user_count

            if finished_users >= total_users:
                self.environment.runner.quit()

```
Ce script se déroule en 2 parties. Dans un premier temps, on fait apparaître tous nos utilisateurs, puis lorsqu'il sont tous prêts à exécuter leur appel, ils l'effectuent en même temps. On recueille les résultats via le lien obtenus en éxecutant la commande

```
locust -f locust_test.py --host https://tiny-insta-474215.appspot.com
```

# Résultats
A chaque fois que nécessaire, la commande ```seed.py``` était utilisée pour rajouter des posts ou des follows.  
L'entièreté des posts et des users étaient supprimés manuellement lorsque le nombre de users ou de posts devaient être réduits, avant de réutiliser la commande ```seed.py``` pour garantir que la répartition du nombre de posts par user était correcte.

<p align="center">
<img width="1000" height="600" alt="conc" src="https://github.com/user-attachments/assets/95a367a7-de52-4d76-875b-820f36387b78" />
</p>

Temps moyen par requête selon la concurrence

<p align="center">
<img width="1000" height="600" alt="post" src="https://github.com/user-attachments/assets/d407bd5a-837a-4bcc-b784-c0cfc3047a54" />
</p>

Temps moyen par requête selon le nombre de posts par utilisateur

<p align="center">
<img width="1000" height="600" alt="fanout" src="https://github.com/user-attachments/assets/972c7896-8e99-4fea-be71-1bf623063b03" />
</p>

Temps moyen par requête selon le nombre de followee
# Discussion
La concurrence entraîne de gros ralentissement. Le serveur sature, ne pouvant pas absorber 1000 requêtes simultanées, et une file d'attente va se mettre en place.  

On n'observe pas de ralentissements significatifs lorsque le nombre de posts augmente. Cela s'explique par le fait que la requête de timeline se limite à 20 posts. Comme le Datastore google cloud utilise un index, on n'a pas à parcourir l'entièreté de la base de données ce qui rend la requête efficace. On récupère alors au plus 20 posts pour chacun des utilisateurs que l'on suit et on les merge pour récupérer les 20 derniers globaux. La quantité de posts que l'on récupère est donc quasiment constante.

On observe un léger ralentissement lorsque le nombre de followee augmente. Comme expliqué précedemment, On récupère au plus 20 posts pour chacun des utilisateurs que l'on suit. Ainsi, plus on augmente le nombre de personne que l'on suit, plus le nombre de posts récupéré augmente et plus le temps que prend la requête augmente. Les mesures effectuées restent de l'ordre du dixième de seconde, il faudrait encore plus accentuer les paramètres afin de mieux observer ce ralentissement.
