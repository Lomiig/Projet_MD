# Projet_MD
Projet MD
```
python massive-gcp/seed.py --users 1000 --posts 50000 --follows-min 20 --follows-max 20
shuf -i 1-1000 -n 1000 | sed "s|^|https://tiny-insta-474215.appspot.com/api/timeline?user=user|; s|$|&limit=20|" > urls1000.txt
time cat urls20.txt | parallel -j 50 "ab -n 1 -c 1 {} 2>&1 | grep -E 'Failed requests|Non-2xx' >> errors.txt"
time cat urls1000.txt | xargs -n 1 -P 1000 -I {} sh -c "ab -n 1 -c 1 {} 2>&1 | grep -E 'Failed requests|Non-2xx' >> errors.txt"
```
