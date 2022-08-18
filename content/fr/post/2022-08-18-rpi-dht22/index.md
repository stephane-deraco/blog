---
title: "Graphe de température et humidité avec un Raspberry Pi et un DHT22"
date: 2022-08-18
translationKey: "rpi-dht22"
toc: true
tags: [Raspberry Pi, DHT22, Humidité, Température, Grafana, Prometheus]
featured_image: images/2022-08-18-dht22.jpg
---

Le DHT22 est un petit capteur de température et d'humidité, facile d'utilisation.
On peut utiliser un Raspberry Pi pour interagir avec, et faire des graphes de l'évolution de la température et de l'humidité.

## Matériel
Nous allons utiliser :
- un Raspberry Pi (ici, la version 3B+)
- une carte SD de bonne qualité (classe 10)
- une alimentation fournissant du 5 V et du 2.5 à 3 A
- un capteur d'humidité et de température DHT22
- une résistance de 10 kΩ
- une _breadboard_ (bloc dans lequel on enfiche les composants électroniques)
- un autre ordinateur pour se connecter au RPi en SSH
- un câble RJ45

## Installation du RPi
### Préparation de la carte SD
Il est possible d'installer sur un autre ordinateur le logiciel _[Raspberry Pi Imager](https://www.raspberrypi.com/software/)_ qui permet de choisir la bonne version de l'OS, la télécharger et l'installer sur la carte SD.
C'est la solution la plus simple.

Si on ne souhaite pas installer ce logiciel, alors on peut récupérer l'image correspondante à la bonne version du Raspberry Pi ici : https://www.raspberrypi.com/software/operating-systems/.
Dans ce cas, il faut ensuite extraire l'image, puis la flasher sur la carte SD avec un outil graphique tel que [Etcher](https://www.balena.io/etcher/) ou en ligne de commande pour les plus expérimentés.

Une fois l'OS installé sur la carte SD, nous allons créer deux fichiers spéciaux dans la partition `boot` :
- un fichier vide `ssh`, pour activer le serveur SSH
- un fichier `userconf` contenant le mot de passe de l'utilisateur `pi` (en effet, [il n'y a plus de mot de passe par défaut](https://www.raspberrypi.com/news/raspberry-pi-bullseye-update-april-2022/))

Le contenu du fichier `userconf` doit contenir le nouveau mot de passe hashé de l'utilisateur `pi`.
Le fichier peut être créé de cette façon :

```shell
# Se placer dans la partition boot, à adapter en fonction de votre OS
cd /Volumes/boot

# Créer le fichier
(echo -n 'pi:'; echo 'changeme' | openssl passwd -6 -stdin) > userconf
```

On peut alors insérer la carte SD dans le RPi, brancher le câble RJ45 entre le RPi et la box internet (ou le routeur) et le démarrer en le branchant au secteur.

### Premier démarrage du RPi
Le RPi démarre, et comme il est connecté à la box internet (ou au routeur) par le câble RJ45, il acquiert une adresse IP automatiquement.

> **Note** : Le fait que le RPi récupère automatiquement une adresse IP est grâce au protocole DHCP qui normalement est en place sur la plupart des box internet.

On ne connait pas son adresse IP pour le moment, mais on peut la récupérer d'au moins deux façons.

- **_Via_ l'interface de configuration de la box internet**  
Par exemple, avec une Freebox, aller sur http://mafreebox.freebox.fr/, puis _Périphériques réseau_, clic-droit sur "raspberry pi" > _Propriétés_ puis onglet _Connectivité_.
On voit alors la dernière adresse IP. 
On peut également obtenir cette information en allant dans les paramètres DHCP de la Freebox et visualiser les baux actifs.

- **Avec `nmap`**  
Il faut d'abord identifier sur quel réseau se trouve l'ordinateur à partir duquel on va scanner les appareils.
Pour cela, récupérer l'adresse IP de l'ordinateur :
  ```shell
  # Windows : Panneau de contrôle > Centre réseau et partage, ou ouvrir une console et taper :
  ipconfig
  
  # Linux
  hostname -I
  
  # Mac : Préférences Système > Réseau
  ```
  Si l'adresse est de type 192.168.0.78, alors taper la commande suivante (voir [cette page](https://nmap.org/download) pour installer nmap) :
  ```shell
  sudo nmap -sP 192.168.0.0/24
  ```

  Le résultat est similaire à :

  ```
  Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-29 18:02 CEST
  Nmap scan report for 192.168.0.20
  Host is up (0.19s latency).
  MAC Address: E8:A0:CD:B8:14:8D (Nintendo)
  [...]
  Nmap scan report for 192.168.0.67
  Host is up (0.19s latency).
  MAC Address: B8:27:EB:C8:C4:EE (Raspberry Pi Foundation)
  Nmap scan report for 192.168.0.254
  Host is up (0.074s latency).
  MAC Address: 14:0C:76:A5:00:8E (Freebox SAS)
  Nmap scan report for 192.168.0.55
  Host is up.
  Nmap done: 256 IP addresses (8 hosts up) scanned in 7.22 seconds
  ```

  On trouve alors l'adresse IP du RPi, ici 192.168.0.67.

### Connexion en SSH au RPi
Maintenant que l'on connait son adresse IP, on peut s'y connecter en SSH (répondre `yes` à la question _"Are you sure you want to continue connecting"_ qui ne sera posée qu'à la première connexion) :

```shell
❯ ssh pi@192.168.0.67
The authenticity of host '192.168.0.67 (192.168.0.67)' can't be established.
ED25519 key fingerprint is SHA256:wL5gzndcDXNkA/Zh8RexfyvFWlyhFYQw2PMvVHhyq20.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.0.67' (ED25519) to the list of known hosts.
pi@192.168.0.67's password: 
Linux raspberrypi 5.15.32-v8+ #1538 SMP PREEMPT Thu Mar 31 19:40:39 BST 2022 aarch64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Jul 29 17:07:46 2022 from 192.168.0.67
pi@raspberrypi:~ $ 
```

On est alors connecté sur le RPi.

### Mise à jour et configuration Wi-Fi
Une fois connecté, lancer la mise à jour :

```shell
sudo apt update
sudo apt upgrade
```

La configuration du Wi-Fi est optionnelle, mais permettra de se passer du câble RJ45, et donc de mettre le RPi où l'on veut.
On utilise l'outil `raspi-config` :

```shell
sudo raspi-config
```

Puis aller dans :
- `5 Localisation Options`
- `L4 WLAN Country`
- Sélectionner le pays, par exemple `FR France`
- `Ok`
- `1 System Options`
- `S1 Wireless LAN`
- Renseigner le SSID (le _nom_) du réseau Wi-Fi
- Renseigner le mot de passe (attendre un peu après avoir validé)
- _Tabulation_ > `Finish`
- À la question `Would you like to reboot now?`, sélectionner `No`

Pour vérifier que le réseau Wi-Fi est bien configuré, taper :

```shell
ip a
```

Rechercher alors le bloc correspondant à `wlan0` :

```
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether b8:27:eb:9d:91:bb brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.54/24 brd 192.168.0.255 scope global dynamic noprefixroute wlan0
       valid_lft 43103sec preferred_lft 37703sec
    inet6 2a01:e0a:b15:4890:29a:d0ce:e2aa:4179/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 86303sec preferred_lft 86303sec
    inet6 fe80::ab3e:f91d:ea52:4912/64 scope link 
       valid_lft forever preferred_lft forever
```

On y retrouve alors l'adresse IP de la carte Wi-Fi du RPi, ici 192.168.0.54.

On peut alors éteindre le RPi (`sudo halt`), débrancher le câble RJ45, puis le redémarrer en enlevant et remettant l'alimentation.

Si tout se passe bien, la connexion SSH sur cette nouvelle adresse IP devrait fonctionner.
Comme c'est la première connexion sur cette adresse, on aura la même question _"Are you sure you want to continue connecting"_.

````shell
ssh pi@192.168.0.54
````

## Communiquer avec le capteur
Le capteur sera relié au RPi à travers ses broches GPIO.
Pour récupérer ses valeurs, nous allons écrire un programme en Python.
Nous allons également utiliser une bibliothèque qui nous facilitera la communication avec le capteur.

### Ajout de logiciels
Python est déjà installé par défaut sur le RPi, nous allons simplement installer la bibliothèque [Adafruit CircuitPython DHT](https://github.com/adafruit/Adafruit_CircuitPython_DHT) qui communiquera avec le capteur.
Pour cela, se connecter en SSH au RPi, et taper les commandes suivantes :

```shell
# Installation des dépendances
sudo apt -y install python3-pip python3-venv libgpiod2

# On va utiliser un répertoire dédié
mkdir dht22
cd dht22

# Création d'un environnement virtuel Python
python -m venv .venv
source .venv/bin/activate

# Installation de la bibliothèque "Adafruit CircuitPython DHT" pour communiquer avec le capteur
pip install adafruit-circuitpython-dht
```

### Branchements

Le capteur DHT22 a 4 broches, qui correspondent à :

| PIN | Description     |
| --- | --------------- |
| 1   | VCC (3.3V à 6V) |
| 2   | Données         |
| 3   | Non utilisé     |
| 4   | Terre           |

De plus, il faut une résistance de 10 ohms entre les broches 1 (VCC) et 2 (Données).

On peut retrouver ces informations [sur le site Adafruit](https://learn.adafruit.com/dht/connecting-to-a-dhtxx-sensor) ou encore [dans les specifications techniques du capteur](https://www.sparkfun.com/datasheets/Sensors/Temperature/DHT22.pdf).


Sur le RPi 3B+, la disposition des broches GPIO est la suivante ([source](https://datasheets.raspberrypi.com/rpi3/raspberry-pi-3-b-plus-reduced-schematics.pdf)) :

![RPi GPIO](images/rpi-gpio.png)

On va donc utiliser :
- broche 1 (3V3) pour le courant
- broche 7 (GPIO4) pour recevoir les données
- broche 9 (GND) pour la terre


La connexion du capteur sur le Raspberry Pi se fait de cette façon :

![Platine d'essai](images/circuit-breadboard.png)

Montage réel :

![Montage réel](images/real-circuit.jpeg)

### Premier test
On va vérifier que le montage est correct en utilisant le shell interactif de python.
Dans le répertoire que l'on a créé (`dht22`), s'assurer que l'on a bien activé l'environnement dédié (`source .venv/bin/activate`) puis lancer Python et taper les commandes suivantes :

```python
import adafruit_dht
import board
dht = adafruit_dht.DHT22(board.D4)
dht.humidity
dht.temperature
```

On a alors une sortie similaire à :

```
(.venv) pi@raspberrypi:~/dht22 $ python
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import adafruit_dht
>>> import board
>>> dht = adafruit_dht.DHT22(board.D4)
>>> dht.humidity
55.1
>>> dht.temperature
28.0
>>> 
```

Le circuit fonctionne comme attendu.

## Récupération et stockage des valeurs
Pour stocker les valeurs et permettre leur récupération pour faire des graphes, on va utiliser une base de données temporelles.
Il en existe plusieurs, ici on va utiliser [Prometheus](https://prometheus.io).

Prometheus fonctionne en mode _pull_, c'est lui qui récupère les métriques.
Il faut que l'on fasse un script Python qui retourne les données du capteur en tant que réponse à une requête HTTP issue du collecteur de Prometheus.

### Script de collecte
On installe le [client Prometheus pour Python](https://github.com/prometheus/client_python) :

```shell
pip install prometheus-client
```

Le script suivant démarre un serveur web qui sera interrogé par Prometheus et exposera les métriques récoltées :

```python
#!/home/pi/dht22/.venv/bin/python

import adafruit_dht
import board
import time
from prometheus_client import start_http_server, Gauge

DATA_PIN = board.D4
DELAY = 60

dht = adafruit_dht.DHT22(DATA_PIN)

temperature_gauge = Gauge('dht22_temperature', 'Température ambiante en degrés Celsius')
humidity_gauge = Gauge('dht22_humidity', 'Humidité ambiante en %')

start_http_server(8000)

while True:
    try:
        temperature = dht.temperature
        humidity = dht.humidity

        if humidity is not None and temperature is not None:
            temperature_gauge.set(temperature)
            humidity_gauge.set(humidity)
    except RuntimeError:
        # If error, do nothing, just wait for the next
        pass

    time.sleep(DELAY)
```

Pour rendre ce script exécutable :

```shell
chmod +x sensors.py
```

On peut le tester en le lançant avec `./sensors.py`, et en allant sur la page [http://192.168.0.54:8000](http://192.168.0.54:8000) (adapter l'adresse IP avec celle du RPi).

Pour gérer ce script avec _systemd_ et qu'il soit automatiquement lancé au démarrage, créer le fichier suivant :

```shell
sudo vim /etc/systemd/system/dht22.service
```

```systemd
# /etc/systemd/system/dht22.service

[Unit]
Description=Temperature and humidity sensor
After=network.target

[Service]
Type=simple
Restart=always
ExecStart=/home/pi/dht22/.venv/bin/python /home/pi/dht22/sensors.py

[Install]
WantedBy=multi-user.target
```

Puis lancer les commandes suivantes :

```shell
sudo systemctl enable dht22.service
sudo systemctl start dht22.service
```

### Prometheus
L'installation se fait simplement avec :

```shell
sudo apt install prometheus
```

On peut vérifier qu'il est bien démarré avec :

```shell
systemctl status prometheus
```

et aussi en allant sur la page [http://192.168.0.54:9090](http://192.168.0.54:9090).

On va ajouter notre configuration de collecte des métriques à la fin du fichier de configuration :

```shell
sudo vim /etc/prometheus/prometheus.yml
```

```yaml
  - job_name: dht22
    scrape_interval: 1m
    static_configs:
      - targets: ['localhost:8000']
```

> **Note** : Attention à bien respecter l'indentation, ce bloc doit être sous le bloc `scrape_configs`.

On redémarre Prometheus pour prendre en compte ces modifications :

```shell
sudo systemctl restart prometheus
```

En allant sur la page de Prometheus [http://192.168.0.54:9090](http://192.168.0.54:9090), on devrait normalement trouver nos deux métriques :

![Mesures dans Prometheus GUI](images/prometheus_gui_measures.png)

## Affichage des valeurs
On va utiliser [Grafana](https://grafana.com/grafana/) comme interface pour suivre l'évolution des mesures.

### Mise en place de Grafana
Grafana n'est pas dans les paquets de RaspberryPi OS, on va donc suivre l'installation préconisée :

```shell
sudo apt -y install adduser libfontconfig1
wget https://dl.grafana.com/oss/release/grafana_9.0.5_arm64.deb
sudo dpkg -i grafana_9.0.5_arm64.deb
rm grafana_9.0.5_arm64.deb
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Tester que Grafana est disponible en allant sur [http://192.168.0.54:3000](http://192.168.0.54:3000).

Utiliser `admin` en nom d'utilisateur et en mot de passe pour la première connexion et suivre les instructions pour modifier le mot de passe par défaut.

### Paramétrage de Grafana
Cliquer sur le bouton _Add your first data source_ (ou aller dans _Configuration_ > _Data sources_).

Sélectionner _Prometheus_, mettre `http://localhost:9090` dans le champ URL, puis cliquer sur _Save & Test_.
Normalement, le test devrait être concluant.

Enfin, retourner, à l'accueil et cliquer sur _Dashboard_ > _Import_, et utiliser [ce fichier](data/dashboard.json) pour un exemple de dashboard.

![Dashboard Grafana](images/grafana.png)

On peut voir que le capteur d'humidité n'est pas parfait :)
Il a des pics réguliers, et on n'a pas eu 90% d'humidité le 8 mai 😱.