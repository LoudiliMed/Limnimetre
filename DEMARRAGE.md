# 1. Mise en route — partie logicielle, sans aucun capteur (l'ajouter après)

Durée : environ une heure la première fois.
Il te faut Docker Desktop, Python 3, un ESP32 et un câble USB qui transmet
les données (certains câbles ne font que charger).


## 2. Home Assistant

Une commande. Remplace `CHEMIN` par le chemin de ton dossier.

    docker run -d --name homeassistant --restart=unless-stopped \
      -v CHEMIN/limnimetre/ha-config:/config \
      -p 8123:8123 \
      ghcr.io/home-assistant/home-assistant:stable

Ouvre http://localhost:8123 et crée ton compte.

Sous Linux tu peux remplacer `-p 8123:8123` par `--network=host`, ce qui
active la découverte automatique. Sous Mac et Windows, Docker ne le permet
pas : on ajoutera l'appareil à la main à l'étape 6, ça marche aussi bien.

---

## 3. ESPHome, en local et pas dans Docker

C'est le point important. Docker Desktop ne donne pas accès au port USB sous
Mac ni sous Windows. Installe donc ESPHome directement :

    python3 -m venv ~/esphome-venv
    source ~/esphome-venv/bin/activate
    pip install esphome

Sous Windows, la troisième ligne devient :

    .\esphome-venv\Scripts\activate

À chaque nouvelle session de terminal, refais la ligne `activate`.

---

## 4. Remplir secrets.yaml

Ouvre `secrets.yaml` et mets le nom et le mot de passe de ton réseau Wi-Fi.

Pour la clé API, génère-la :

    python3 -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())"

Colle le résultat à la place de `REMPLACER_PAR_UNE_CLE...`.

Remarque : le Wi-Fi du campus avec authentification ne marchera pas. Utilise
le partage de connexion de ton téléphone, et mets ce SSID dans le fichier.

---

## 5. Flasher l'ESP32

Branche la carte en USB. Puis, dans le dossier :

    esphome run limnimetre.yaml

La première compilation prend cinq à dix minutes. ESPHome te demande le port
série : choisis celui qui ressemble à `/dev/cu.usbserial...` sous Mac, ou
`COM3` sous Windows.

À la fin, il affiche les journaux de la carte et son adresse IP. Note-la.

Les fois suivantes, choisis `OTA` au lieu du port USB : la mise à jour passe
par le Wi-Fi, sans débrancher quoi que ce soit.

---

## 6. Ajouter l'appareil dans Home Assistant

Paramètres, Appareils et services, Ajouter une intégration, chercher
**ESPHome**. Saisis l'adresse IP notée à l'étape 5, port 6053.

Il demande la clé de chiffrement : c'est la valeur de `api_key` de ton
`secrets.yaml`.

Sous Linux avec `--network=host`, l'appareil apparaît tout seul sans rien
saisir. C'est la découverte automatique, celle qui répond à REQ003 et REQ010.

---

## 7. Le tableau de bord

Paramètres, Tableaux de bord, en créer un nouveau. Dans le menu en haut à
droite, Modifier, puis de nouveau le menu, Éditeur YAML brut.

Efface tout et colle le contenu de `dashboard-limnimetre.yaml`.

Si une carte reste vide, le nom d'entité diffère. Va dans Outils de
développement, États, tape `limnimetre` et recopie le nom exact.

---

## 8. Vérifier, et garder les preuves

- Bouge le curseur « Niveau simulé » : le volume et le pourcentage suivent.
- Mets-le à 0 : l'alerte « Cuve basse » se déclenche.
- Débranche l'ESP32 : l'appareil passe en indisponible, puis revient seul au
  rebranchement. **Filme-le, c'est REQ003 démontrée.**
- Laisse tourner trente minutes, puis regarde la courbe : pas de trou.
  **Capture d'écran, c'est REQ002 démontrée.**
- Regarde la LED sur GPIO25 : fixe en service, lente pendant la connexion.
  **C'est REQ007.**

Range ces preuves tout de suite dans le dossier de documentation. En semaine 4
tu n'auras plus le temps de les refaire.

---

## 9. Le jour où les capteurs arrivent

Dans `limnimetre.yaml` :

1. Supprime le bloc `number:` entier.
2. Supprime les deux premiers capteurs template, « Distance brute » et
   « Temperature ».
3. Décommente toute la section CAPTEURS REELS en bas du fichier.
4. Règle `hauteur_capteur` à la vraie distance mesurée au mètre ruban.

Puis `esphome run limnimetre.yaml` en OTA. Rien d'autre ne bouge : la
correction en température, le filtrage, le calcul de volume et les alertes
continuent de fonctionner à l'identique.

---

## À vérifier dans la doc ESPHome avant de brancher les vrais capteurs

- Le nom exact de la plateforme pour le SEN0311 : `a02yyuw` ou `jsn_sr04t`
  selon la version d'ESPHome.
- Le DS18B20 : les versions récentes utilisent `one_wire:` plus
  `platform: dallas_temp`, les anciennes utilisaient `dallas:`.
- L'unité renvoyée par le module : millimètres ou centimètres. Ajuste le
  filtre `multiply:` en conséquence.

---

## Si ça coince

| Symptôme | Cause la plus fréquente |
| --- | --- |
| Aucun port série proposé | Câble USB qui ne fait que charger, ou pilote CP210x/CH340 manquant |
| La compilation échoue | Mauvaise indentation du YAML. L'erreur donne la ligne |
| La carte ne se connecte pas au Wi-Fi | SSID en 5 GHz : l'ESP32 ne fait que du 2,4 GHz |
| Home Assistant refuse la clé | Espace en trop collé avec la clé dans secrets.yaml |