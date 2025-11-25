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

Pour évaluer la performance, on utilise apache-bench. La commande à utiliser est
```
ab -n 1 -c x1 https://tiny-insta-474215.appspot.com/api/timeline?user=user1
```
Cette commande va effectuer x1 requêtes simultanées mais uniquement sur la timeline de user1. Apache-bench ne possède pas de méthode permettant de modifier l'url que l'on test, garantissant ainsi que l'on interroge x1 users différents.  
Voici la solution que j'ai mise en place afin de garantir que la commande appelle bien des user différent:
```
shuf -i 1-x1 -n x2 | sed "s|^|https://tiny-insta-474215.appspot.com/api/timeline?user=user|" > urlsx2.txt
time cat urlsx2.txt | parallel -j x2 "ab -n 1 -c 1 {} 2>&1 | grep -E 'Failed requests|Non-2xx' >> errors.txt"
```
On commence par mélanger les nombres entre 1 et x1 et on en sélectionne x2. ```sed``` va ensuite rajouter ce chiffre à la fin, ce qui va créer des url avec des user différents, puis on sauvegarde ces urls dans le fichier urlsx2.txt  
Pour lancer ```ab``` avec ces urls, on utilise la commande ```parallel``` pour lancer x2 commandes ```ab``` simultanées, puis enfin on récupère les messages d'erreur et les requêtes échoués dans errors.txt  
Cependant, la commande parallèle est limitée et ne permet pas d'effectuer 1000 appels simultanés. Pour ce cas particulier, j'ai utilisé la commande
```
time cat urls1000.txt | xargs -n 1 -P 1000 -I {} sh -c "ab -n 1 -c 1 {} 2>&1 | grep -E 'Failed requests|Non-2xx' >> errors.txt"
```
# Remarques
Bien que ces commandes fonctionne, on ne peut plus récupérer les données via ```ab```. On est obligé d'utiliser la commande ```time```. Le temps obtenu avec risque d'être plus long car on mesure aussi le temps que prennent ces commandes, et pas uniquement le temps des requêtes.  
De plus, on ne sait pas exactement comment fonctionne ```parallel``` et ```xargs```. ```xargs``` doit ouvrir des nouveaux shell, ce qui risque de créer une latence et d'encore plus fausser nos résultats.

# Résultats
A chaque fois que nécessaire, la commande ```seed.py``` était utilisée pour rajouter des posts ou des follows.  
L'entièreté des posts et des users étaient supprimés manuellement lorsque le nombre de users ou de posts devaient être réduits, avant de réutiliser la commande ```seed.py``` pour garantir que la répartition du nombre de posts par user était correcte.
Chaque commande ```time``` était lancée une première fois pour vérifier que l'absence d'erreurs et la cohérences des résultats, puis les 3 runs mesurés étaient ensuite effectués.

<p align="center">
<img width="942" height="530" alt="conc" src="https://github.com/user-attachments/assets/e331cb1b-1e29-4a6e-be5a-02a122189661" />  
</p>

Temps moyen par requête selon la concurrence

<p align="center">
<img width="942" height="530" alt="post" src="https://github.com/user-attachments/assets/eb36e9d2-aed9-44d7-9405-6240de87365e" />  
</p>

Temps moyen par requête selon le nombre de posts par utilisateur

<p align="center">
<img width="942" height="530" alt="fanout" src="https://github.com/user-attachments/assets/ae1064bb-c071-429b-83f0-deef8acdab08" />  
</p>

Temps moyen par requête selon le nombre de followee
# Discussion
La concurrence entraîne de gros ralentissement. Le serveur ne peut pas absorber 1000 requêtes simultanées, et une file d'attente va se mettre en place.  
Il est également possible, comme annoncé plus haut, que la commande ```xargs``` créée elle même artificiellement cette file d'attente en saturant la machien, ne sachant comment paralléliser 1000 commande ```ab``` pour les effectuer en simultané.

On n'observe pas de ralentissements significatifs lorsque le nombre de posts augmente. Cela s'explique par le fait que la requête de timeline se limite à 20 posts. Comme le Datastore google cloud utilise un index, on n'a pas à parcourir l'entièreté de la base de données ce qui rend la requête efficace. On récupère alors au plus 20 posts pour chacun des utilisateurs que l'on suit et on les merge pour récupérer les 20 derniers globaux. La quantité de posts que l'on récupère est donc quasiment constante.

A contrario, on s'attendrait à observer un ralentissement lorsque le nombre d'utilisateurs suivis augmentent. En effet, le nombre de posts récupérés est un multiple du nombre d'utilisateurs suivis. Ici, on ne voit pas ce ralentissement. Dans le processus d'expérimentation, on lance la commande une première fois. On peut imaginer que le résultat de cette requête est enregistré en mémoire, ce qui rend le résultat constant pour les requêtes suivantes. Il est surtout probable que le choix d'utiliser ```parallel``` créé du bruit empêchant de mesurer les variations proprement. En effet, l'ordre de grandeur du nombre de posts à rechercher reste faible quelque soit le nombre d'utilisateurs que l'on follow. La recherche de posts prend beaucoup moins de temps que les 0.8 secondes que l'on observe, et le coût que l'on souhaite mesurer est donc caché.
