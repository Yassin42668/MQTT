# ═══════════════════════════════════════════════════════════════
# ÉPREUVE E5 — PARTIE BACKEND MQTT
# Toutes les commandes — Version définitive
# ═══════════════════════════════════════════════════════════════


# ───────────────────────────────────────────
# ÉTAPE 1 — Mettre à jour le système
# ───────────────────────────────────────────

sudo apt update && sudo apt upgrade -y


# ───────────────────────────────────────────
# ÉTAPE 2 — Installer tout ce dont tu as besoin
# ───────────────────────────────────────────

sudo apt install mosquitto mosquitto-clients -y
sudo apt install libmosquitto-dev libmysqlclient-dev g++ make -y

# mosquitto             → le broker MQTT
# mosquitto-clients     → outils de test (mosquitto_pub / mosquitto_sub)
# libmosquitto-dev      → bibliothèque C++ pour MQTT
# libmysqlclient-dev    → bibliothèque C++ pour MySQL
# g++                   → compilateur C++
# make                  → outil de compilation


# ───────────────────────────────────────────
# ÉTAPE 3 — Créer le fichier de mots de passe
# ───────────────────────────────────────────

# ⚠️ À ADAPTER : remplace "yassin" par le nom demandé dans ton sujet
sudo mosquitto_passwd -c /etc/mosquitto/passwd yassin

# Pour ajouter un deuxième utilisateur SANS écraser le fichier (pas de -c) :
# sudo mosquitto_passwd /etc/mosquitto/passwd autreuser


# ───────────────────────────────────────────
# ÉTAPE 4 — Configurer Mosquitto
# ───────────────────────────────────────────

sudo nano /etc/mosquitto/conf.d/default.conf

# Contenu EXACT à écrire dans le fichier :
# ----------------------------------------
# listener 1883 0.0.0.0
# allow_anonymous false
# password_file /etc/mosquitto/passwd
# log_type all
# ----------------------------------------
# Sauvegarder : Ctrl+X → Y → Entrée
#
# ⚠️ PIÈGES À ÉVITER :
# - Ne PAS mettre persistence, persistence_location ou log_dest file
#   (déjà dans /etc/mosquitto/mosquitto.conf, un doublon = erreur)
# - Ne PAS mettre d'espaces avant un # en début de ligne


# ───────────────────────────────────────────
# ÉTAPE 5 — Redémarrer et vérifier Mosquitto
# ───────────────────────────────────────────

sudo systemctl restart mosquitto
sudo systemctl status mosquitto
# → Tu dois voir "active (running)"

sudo systemctl is-enabled mosquitto
# → Doit afficher "enabled"

# Si c'est pas enabled :
# sudo systemctl enable mosquitto

# ─── En cas d'erreur ───
# Lancer à la main pour voir le problème exact :
sudo mosquitto -c /etc/mosquitto/mosquitto.conf
# S'il se bloque sans erreur → c'est bon, fais Ctrl+C et relance le service

# Voir les logs :
sudo journalctl -xeu mosquitto.service | tail -30
# ou
sudo tail -f /var/log/mosquitto/mosquitto.log


# ───────────────────────────────────────────
# ÉTAPE 6 — Configurer le firewall UFW
# ───────────────────────────────────────────

sudo ufw allow 22
sudo ufw allow 1883
sudo ufw enable
# → Tape "y" pour confirmer

sudo ufw status
# → Tu dois voir :
# 22     ALLOW     Anywhere
# 1883   ALLOW     Anywhere


# ═══════════════════════════════════════════════════════════════
# ÉTAPE 7 — TESTER MQTT
# ═══════════════════════════════════════════════════════════════


# ─── 7.1 Test basique (tous synoptiques) ───

# TERMINAL 1 — subscriber (écoute tout) :
mosquitto_sub -h localhost -t "salle_serveur/#" -u yassin -P tonmotdepasse

# Avec le nom du topic affiché en plus :
mosquitto_sub -h localhost -t "salle_serveur/#" -u yassin -P tonmotdepasse -v

# TERMINAL 2 — publisher (envoie un message) :
mosquitto_pub -h localhost -t "salle_serveur/capteur1/temperature" -m "22.5" -u yassin -P tonmotdepasse
# → Tu dois voir "22.5" dans le terminal 1


# ─── 7.2 Test par synoptique ───

# SYNOPTIQUE 1 — RFID :
mosquitto_pub -h localhost -t "salle_serveur/capteur1/rfid" -m "A1B2C3D4" -u yassin -P tonmotdepasse

# SYNOPTIQUE 2 — ENV4 + SCD040 (température, humidité, pression, CO₂) :
mosquitto_pub -h localhost -t "salle_serveur/capteur1/temperature" -m "22.5" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/humidite" -m "60" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/pression" -m "1013" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/co2" -m "650" -u yassin -P tonmotdepasse

# SYNOPTIQUE 3 — ENV4 + Radar LD2410 (température, humidité, pression, présence) :
mosquitto_pub -h localhost -t "salle_serveur/capteur1/temperature" -m "22.5" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/humidite" -m "60" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/pression" -m "1013" -u yassin -P tonmotdepasse
mosquitto_pub -h localhost -t "salle_serveur/capteur1/presence" -m "1" -u yassin -P tonmotdepasse


# ─── 7.3 Test avec JSON (comme le vrai M5Stack) ───

mosquitto_pub -h localhost -t "salle_serveur/capteur1/data" -m '{"capteur":"capteur1","temperature":22.5,"humidite":60}' -u yassin -P tonmotdepasse


# ─── 7.4 Test de sécurité (à montrer au jury) ───

mosquitto_sub -h localhost -t "salle_serveur/#" -u yassin -P mauvais_mdp
# → Doit afficher : "Connection Refused: not authorised"
# → C'est la preuve que ta sécurité fonctionne !


# ═══════════════════════════════════════════════════════════════
# ÉTAPE 8 — Modifier le fichier config.h
# ═══════════════════════════════════════════════════════════════

# Le prof te donnera un dossier avec les fichiers du backend C++
# Dedans il y aura un config.h à modifier

cd /le/dossier/du/prof
nano config.h

# Le fichier ressemblera à ça :
# ─────────────────────────────────────────
# // ===== Configuration MQTT =====
# #define MQTT_HOST       "localhost"
# #define MQTT_PORT       1883
# #define MQTT_USER       "yassin"
# #define MQTT_PASSWORD   "admin"
# #define MQTT_TOPIC      "salle_serveur/#"
#
# // ===== Configuration MySQL =====
# #define DB_HOST         "localhost"
# #define DB_USER         "mqtt_user"
# #define DB_PASSWORD     "mqtt_password"
# #define DB_NAME         "salle_serveur"
# ─────────────────────────────────────────
#
# ⚠️ CE QUE TU ADAPTES :
#
# MQTT_HOST       → "localhost" si broker sur le même serveur, sinon l'IP
# MQTT_PORT       → 1883
# MQTT_USER       → le nom créé avec mosquitto_passwd (étape 3)
# MQTT_PASSWORD   → le mot de passe défini à l'étape 3
# MQTT_TOPIC      → "salle_serveur/#" (fonctionne pour tous les synoptiques)
#
# DB_HOST         → "localhost"
# DB_USER         → DEMANDER À TON COÉQUIPIER BDD
# DB_PASSWORD     → DEMANDER À TON COÉQUIPIER BDD
# DB_NAME         → DEMANDER À TON COÉQUIPIER BDD
#
# Sauvegarder : Ctrl+X → Y → Entrée


# ═══════════════════════════════════════════════════════════════
# ÉTAPE 9 — Compiler le backend C++
# ═══════════════════════════════════════════════════════════════

# Si un Makefile est fourni dans le dossier :
make

# Si PAS de Makefile :
g++ -o backend main.cpp -lmosquitto -lmysqlclient

# ─── En cas d'erreur de compilation ───
# "cannot find -lmosquitto"    → sudo apt install libmosquitto-dev -y
# "cannot find -lmysqlclient"  → sudo apt install libmysqlclient-dev -y
# "command not found: g++"     → sudo apt install g++ -y
# "command not found: make"    → sudo apt install make -y


# ═══════════════════════════════════════════════════════════════
# ÉTAPE 10 — Lancer le backend
# ═══════════════════════════════════════════════════════════════

./backend

# Pour le lancer en arrière-plan :
./backend &

# Pour voir s'il tourne toujours :
ps aux | grep backend

# Pour l'arrêter :
kill $(pidof backend)


# ═══════════════════════════════════════════════════════════════
# ÉTAPE 11 — TEST FINAL DE BOUT EN BOUT
# ═══════════════════════════════════════════════════════════════

# C'est le test le plus important — il prouve que TOUT fonctionne :
# mosquitto_pub → Broker → Backend C++ → MySQL

# TERMINAL 1 — le backend tourne :
./backend

# TERMINAL 2 — tu simules un capteur :
mosquitto_pub -h localhost -t "salle_serveur/capteur1/temperature" -m "22.5" -u yassin -P tonmotdepasse

# TERMINAL 3 — tu vérifies en base de données :
mysql -u mqtt_user -p salle_serveur
# Tape le mot de passe MySQL puis :

# SELECT * FROM mesures;
#
# Tu dois voir :
# +----+----------+-------------------------------------+--------+---------------------+
# | id | capteur  | topic                               | valeur | date_heure          |
# +----+----------+-------------------------------------+--------+---------------------+
# |  1 | capteur1 | salle_serveur/capteur1/temperature  | 22.5   | 2026-05-28 14:30:00 |
# +----+----------+-------------------------------------+--------+---------------------+
#
# EXIT;

# ⚠️ IMPORTANT : sans le ./backend qui tourne, les données NE VONT PAS en base.
# mosquitto_pub seul = juste un test du broker
# mosquitto_pub + ./backend = les données arrivent en MySQL


# ═══════════════════════════════════════════════════════════════
# RÉSUMÉ — CE QUI CHANGE ET CE QUI NE CHANGE PAS
# ═══════════════════════════════════════════════════════════════

# ✅ NE CHANGE JAMAIS :
# - apt install mosquitto mosquitto-clients
# - apt install libmosquitto-dev libmysqlclient-dev g++ make
# - Contenu de default.conf
# - systemctl restart/status/enable mosquitto
# - ufw allow 22 / 1883 / enable
# - make ou g++ -o backend main.cpp -lmosquitto -lmysqlclient
# - ./backend

# ⚠️ À ADAPTER :
# - mosquitto_passwd → nom d'utilisateur selon le sujet
# - mosquitto_pub / mosquitto_sub → utilisateur + mot de passe
# - mosquitto_pub → topics selon le synoptique
# - config.h → paramètres MQTT (user/password) + paramètres MySQL (demander au coéquipier)


# ═══════════════════════════════════════════════════════════════
# CHECKLIST DU JOUR J
# ═══════════════════════════════════════════════════════════════

# [ ] Système à jour
# [ ] Mosquitto + clients installés
# [ ] Bibliothèques C++ installées
# [ ] Fichier de mots de passe créé
# [ ] default.conf configuré (sans doublons !)
# [ ] Mosquitto active (running) + enabled au boot
# [ ] UFW : ports 22 et 1883 ouverts
# [ ] Test pub/sub réussi
# [ ] Test sécurité (mauvais mot de passe → refusé)
# [ ] config.h modifié (MQTT + MySQL)
# [ ] Backend compilé (make ou g++)
# [ ] Backend lancé (./backend)
# [ ] Test de bout en bout : pub → backend → MySQL ✅
# [ ] Demandé les identifiants MySQL à mon coéquipier BDD
