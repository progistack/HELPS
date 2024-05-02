## Postgresql: ERROR: invalid memory alloc request size

Lors de certain traitement sur une grande quantité de donnée, il arrive que l'ORM d'Odoo transmette une requete dont la limite excede celle definie par defaut pour les requettes par postgres.

```Conf
; /etc/postgresql/[VERSION]/main/postgresql.conf

work_mem=512MB
maintenance_work_mem=768MB
max_wal_size=1GB

; Les configurations suivantes sont facultatives
max_connections=100
shared_buffers=4GB
effective_cache_size=12GB
checkpoint_completion_target=0.9
wal_buffers=16MB
default_statistics_target=100
random_page_cost=1.1
effective_io_concurrency=200
min_wal_size=1GB
max_worker_processes=4
```

Pour les personnes utilisant un conteneur Docker, ils pourront proceder comme suite en ettendant la commande de demarrage:

```Yaml
version: "3"
services:
  db:
    image: postgres:${POSTGRES_VERSION}
    # ...
    command: >
      -c work_mem=512MB
      -c maintenance_work_mem=768MB
      -c max_wal_size=1GB
    # Les configurations suivantes restent facultatifs
      -c max_connections=100
      -c shared_buffers=4GB
      -c effective_cache_size=12GB
      -c checkpoint_completion_target=0.9
      -c wal_buffers=16MB
      -c default_statistics_target=100
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
      -c min_wal_size=1GB
      -c max_worker_processes=4
    # ...
```


## 413 REQUEST ENTITY TOO LARGE (sans NGINX)

Le problème est dû à un maximum codé en dur qui ne semble pas pouvoir être réglé par autre chose qu'un paramètre système. Cela signifie qu'il ne peut pas être défini tant que vous n'avez pas créé votre instance - donc toute personne essayant de restaurer une base de données supérieure à 128 Mo se retrouvera bloqué.

Afin de resoudre ce probleme, vous avez la possibilité de modifier le fichier des requetes `(http.py)` afin d'augmenter cette limite et aussi de definir la variable `web.max_file_upload_size` dans le fichier de configuration `odoo.conf`.

> Dans le fichier odoo `http.py` du dossier `odoo`

Modifier la valeur de la variable `DEFAULT_MAX_CONTENT_LENGTH`

```Python
# odoo/odoo/http.py

# Remplacer cette ligne
DEFAULT_MAX_CONTENT_LENGTH = 128 * 1024 * 1024  # 128MiB

# Par celle-là
DEFAULT_MAX_CONTENT_LENGTH = 2048 * 1024 * 1024  # 2GiB
```

Toute personne coincée là-dessus et utilisant le conteneur Docker peut utiliser le remplacement de l'utilisateur `(user)` et du point d'entrée `(entrypoint)` comme indiqué ci-après.

> Dans le fichier `compose.yml`

```Yaml
version: "3"
services:
  db:
    image: postgres:${POSTGRES_VERSION}
    # ...
  odoo:
    user: root
    image: odoo:${ODOO_VERSION}
    container_name: ${ODOO_CONTAINER_NAME}
    entrypoint: >
      /bin/bash -c 
      "if [ -f /usr/lib/python3/dist-packages/odoo/http.py ]; then
        cp /usr/lib/python3/dist-packages/odoo/http.py /tmp
        sed -i -e 's/^DEFAULT_MAX_CONTENT_LENGTH.*/DEFAULT_MAX_CONTENT_LENGTH = 2048 * 1024 * 1024/g' /tmp/http.py;
        cat /tmp/http.py >/usr/lib/python3/dist-packages/odoo/http.py
      fi;
      exec odoo"
      # ...
```

> Dans le fichier de configuration odoo `odoo.conf`

```Conf
; odoo/odoo.conf

[options]
web.max_file_upload_size = 1073741824
```
