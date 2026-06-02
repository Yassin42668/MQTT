# Partie Backend MQTT
https://stackforge.thrnecore.fr/

tree

---

# Étape 1 — Mettre à jour le système

```bash
sudo apt update && sudo apt upgrade -y
```

---

# Étape 2 — Installer tout ce dont tu as besoin

```bash
sudo apt install mosquitto mosquitto-clients -y
sudo apt install libmosquitto-dev libmysqlclient-dev g++ make -y
```

| Paquet | Rôle |
|---|---|
| `mosquitto` | Le broker MQTT |
| `mosquitto-clients` | Outils de test (`mosquitto_pub` / `mosquitto_sub`) |
| `libmosquitto-dev` | Bibliothèque C++ pour MQTT |
| `libmysqlclient-dev` | Bibliothèque C++ pour MySQL |
| `g++` | Compilateur C++ |
| `make` | Outil de compilation automatique |

---

# Étape 3 — Créer le fichier de mots de passe

```bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd yassin
```

> ⚠️ **À adapter :** remplace `yassin` par le nom demandé dans ton sujet

Pour ajouter un deuxième utilisateur **sans écraser** le fichier (pas de `-c`) :

```bash
sudo mosquitto_passwd /etc/mosquitto/passwd autreuser
```

---

# Étape 4 — Configurer Mosquitto

```bash
sudo nano /etc/mosquitto/conf.d/default.conf
```

Contenu **exact** à écrire :

```
listener 1883 0.0.0.0
allow_anonymous false
password_file /etc/mosquitto/passwd
log_type all
```

Sauvegarder : **Ctrl+X** → **Y** → **Entrée**

| Ligne | Rôle |
|---|---|
| `listener 1883 0.0.0.0` | Écoute sur le port 1883, toutes les interfaces |
| `allow_anonymous false` | Interdit les connexions sans mot de passe |
| `password_file ...` | Pointe vers le fichier de mots de passe |
| `log_type all` | Active tous les logs |

> ⚠️ **Piège :** ne PAS mettre `persistence`, `persistence_location` ou `log_dest file` dedans — déjà dans `/etc/mosquitto/mosquitto.conf`. Un doublon = erreur au démarrage.

> ⚠️ **Piège :** pas d'espaces avant un `#` en début de ligne — ça provoque une erreur.

---

# Étape 5 — Redémarrer et vérifier Mosquitto

```bash
sudo systemctl restart mosquitto
sudo systemctl status mosquitto
```

Tu dois voir **active (running)**.

```bash
sudo systemctl is-enabled mosquitto
```

Doit afficher **enabled**. Si c'est pas le cas :

```bash
sudo systemctl enable mosquitto
```

### En cas d'erreur

Lancer à la main pour voir le problème exact :

```bash
sudo mosquitto -c /etc/mosquitto/mosquitto.conf
```

S'il se bloque sans erreur → c'est bon, fais **Ctrl+C** et relance le service.

Voir les logs :

```bash
sudo journalctl -xeu mosquitto.service | tail -30
```

ou

```bash
sudo tail -f /var/log/mosquitto/mosquitto.log
```

---

# Étape 6 — Configurer le firewall UFW

```bash
sudo ufw allow 22
sudo ufw allow 1883
sudo ufw enable
sudo ufw status
```

Tape **y** pour confirmer l'activation.

Tu dois voir :

```
22     ALLOW     Anywhere
1883   ALLOW     Anywhere
```

| Port | Pourquoi |
|---|---|
| `22` | SSH — administrer le serveur à distance |
| `1883` | MQTT — les M5Stack communiquent avec le broker |

---

# Étape 7 — Tester MQTT

## 7.1 — Test basique (tous synoptiques)

**Terminal 1 — subscriber (écoute tout) :**

```bash
mosquitto_sub -h localhost -t "salle_serveur/#" -u yassin -P tonmotdepasse
```

Avec le nom du topic affiché en plus :

```bash
mosquitto_sub -h localhost -t "salle_serveur/#" -u yassin -P tonmotdepasse -v
```

**Terminal 2 — publisher (envoie un message) :**

```bash
mosquitto_pub -h localhost -t "salle_serveur/capteur1/temperature" -m "22.5" -u yassin -P tonmotdepasse
```

Tu dois voir **22.5** dans le terminal 1.

---

## 7.2 — Test par synoptique

### Synoptique 1 — RFID

```bash
mosquitto_pub -h localhost -t "salle_serveur/capteur1/rfid" -m "A1B2C3D4" -u yassin -P tonmotdepasse
```

### Synoptique 2 — ENV4 + SCD040 (température, humidité, pression, CO₂)

```bash
mosquitto_pub -h localhost -t "salle_serveur/capteur1/temperature" -m "22.5" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/humidite" -m "60" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/pression" -m "1013" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/co2" -m "650" -u yassin -P tonmotdepasse
```

### Synoptique 3 — ENV4 + Radar LD2410 (température, humidité, pression, présence)

```bash
mosquitto_pub -h localhost -t "salle_serveur/capteur1/temperature" -m "22.5" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/humidite" -m "60" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/pression" -m "1013" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/presence" -m "1" -u yassin -P tonmotdepasse
```

---

## 7.3 — Test avec JSON (comme le vrai M5Stack)

```bash
mosquitto_pub -h localhost -t "salle_serveur/capteur1/data" -m '{"capteur":"capteur1","temperature":22.5,"humidite":60}' -u yassin -P tonmotdepasse
```

---

## 7.4 — Test de sécurité (à montrer au jury)

```bash
mosquitto_sub -h localhost -t "salle_serveur/#" -u yassin -P mauvais_mdp
```

Doit afficher : **Connection Refused: not authorised**

→ C'est la preuve que ta sécurité fonctionne !

---

# Étape 8 — Modifier le fichier config.h

Le prof te donnera un dossier avec les fichiers du backend C++. Dedans il y aura un `config.h` à modifier :

```bash
cd /le/dossier/du/prof
nano config.h
```

Le fichier ressemblera à ça :

```cpp
// ===== Configuration MQTT =====
#define MQTT_HOST       "localhost"
#define MQTT_PORT       1883
#define MQTT_USER       "yassin"
#define MQTT_PASSWORD   "admin"
#define MQTT_TOPIC      "salle_serveur/#"

// ===== Configuration MySQL =====
#define DB_HOST         "localhost"
#define DB_USER         "mqtt_user"
#define DB_PASSWORD     "mqtt_password"
#define DB_NAME         "salle_serveur"
```

### Ce que tu adaptes :

| Paramètre | Ce que tu mets |
|---|---|
| `MQTT_HOST` | `localhost` si broker sur le même serveur, sinon l'IP |
| `MQTT_PORT` | `1883` |
| `MQTT_USER` | Le nom créé avec `mosquitto_passwd` (étape 3) |
| `MQTT_PASSWORD` | Le mot de passe défini à l'étape 3 |
| `MQTT_TOPIC` | `"salle_serveur/#"` (fonctionne pour tous les synoptiques) |
| `DB_HOST` | `localhost` |
| `DB_USER` | **Demander à ton coéquipier BDD** |
| `DB_PASSWORD` | **Demander à ton coéquipier BDD** |
| `DB_NAME` | **Demander à ton coéquipier BDD** |

Sauvegarder : **Ctrl+X** → **Y** → **Entrée**

---

# Étape 9 — Compiler le backend C++

Si un **Makefile** est fourni dans le dossier :

```bash
make
```

Si **pas de Makefile** :

```bash
g++ -o backend main.cpp -lmosquitto -lmysqlclient
```

### En cas d'erreur de compilation

| Erreur | Solution |
|---|---|
| `cannot find -lmosquitto` | `sudo apt install libmosquitto-dev -y` |
| `cannot find -lmysqlclient` | `sudo apt install libmysqlclient-dev -y` |
| `command not found: g++` | `sudo apt install g++ -y` |
| `command not found: make` | `sudo apt install make -y` |

---

# Étape 10 — Lancer le backend

```bash
./backend
```

Pour le lancer en arrière-plan :

```bash
./backend &
```

Pour voir s'il tourne toujours :

```bash
ps aux | grep backend
```

Pour l'arrêter :

```bash
kill $(pidof backend)
```

---

# Étape 11 — Test final de bout en bout

C'est le test le plus important — il prouve que TOUT fonctionne :

> `mosquitto_pub` → Broker → Backend C++ → MySQL

**Terminal 1 — le backend tourne :**

```bash
./backend
```

**Terminal 2 — tu simules un capteur :**

```bash
mosquitto_pub -h localhost -t "salle_serveur/capteur1/temperature" -m "22.5" -u yassin -P tonmotdepasse
```

**Terminal 3 — tu vérifies en base de données :**

```bash
mysql -u mqtt_user -p salle_serveur
```

```sql
SELECT * FROM mesures;
```

Tu dois voir :

```
+----+----------+-------------------------------------+--------+---------------------+
| id | capteur  | topic                               | valeur | date_heure          |
+----+----------+-------------------------------------+--------+---------------------+
|  1 | capteur1 | salle_serveur/capteur1/temperature  | 22.5   | 2026-05-28 14:30:00 |
+----+----------+-------------------------------------+--------+---------------------+
```

```sql
EXIT;
```

> ⚠️ **Important :** sans le `./backend` qui tourne, les données **ne vont PAS** en base. `mosquitto_pub` seul = juste un test du broker. `mosquitto_pub` + `./backend` = les données arrivent en MySQL.

---

# Résumé — Ce qui change et ce qui ne change pas

## ✅ Ne change jamais

| Commande |
|---|
| `apt install mosquitto mosquitto-clients` |
| `apt install libmosquitto-dev libmysqlclient-dev g++ make` |
| Contenu de `default.conf` |
| `systemctl restart / status / enable mosquitto` |
| `ufw allow 22 / 1883 / enable` |
| `make` ou `g++ -o backend main.cpp -lmosquitto -lmysqlclient` |
| `./backend` |

## ⚠️ À adapter

| Élément | Quoi adapter |
|---|---|
| `mosquitto_passwd` | Nom d'utilisateur selon le sujet |
| `mosquitto_pub` / `mosquitto_sub` | Utilisateur + mot de passe |
| `mosquitto_pub` | Topics selon le synoptique |
| `config.h` — partie MQTT | Utilisateur + mot de passe |
| `config.h` — partie MySQL | Demander au coéquipier BDD |

---

# Checklist du jour J

- [ ] Système à jour
- [ ] Mosquitto + clients installés
- [ ] Bibliothèques C++ installées
- [ ] Fichier de mots de passe créé
- [ ] `default.conf` configuré (sans doublons !)
- [ ] Mosquitto `active (running)` + `enabled` au boot
- [ ] UFW : ports 22 et 1883 ouverts
- [ ] Test pub/sub réussi
- [ ] Test sécurité (mauvais mot de passe → refusé)
- [ ] `config.h` modifié (MQTT + MySQL)
- [ ] Backend compilé (`make` ou `g++`)
- [ ] Backend lancé (`./backend`)
- [ ] Test de bout en bout : pub → backend → MySQL ✅
- [ ] Demandé les identifiants MySQL à mon coéquipier BDD

- [ ] 
ls- l /etc/passwd
sudo chown mosquitto:mosquitto /etc/mosquitto/passwd

