# Surveillance des Batteries Zigbee (Zigbee2MQTT)

Ce projet permet de surveiller l'état de santé de tous vos appareils Zigbee sur batterie. Il croise les données de **Zigbee2MQTT** (pour les métadonnées comme les dates de changement de pile) avec les états de **Home Assistant** (pour le niveau de pile et la disponibilité).

## 📂 Structure du Projet

- `zigbee_sensors.yaml` : Contient **tous** les capteurs (Batteries + Disponibilité "Radar").
- `README.md` : Ce fichier de documentation.

## ⚠️ Pré-requis Important : Topic MQTT
Le fichier `zigbee_sensors.yaml` est configuré par défaut avec un topic spécifique : **`zigbee2mqtt02`**.
```yaml
- trigger:
    - platform: mqtt
      topic: zigbee2mqtt02/bridge/devices  <-- VÉRIFIEZ CE TOPIC !
    - platform: mqtt
      topic: zigbee2mqtt02/+              <-- ET CELUI-CI AUSSI !
```
Si votre installation Zigbee2MQTT utilise le topic par défaut (`zigbee2mqtt`), **vous devez modifier ces 2 lignes** avant l'installation pour mettre : `zigbee2mqtt/...`.

## 🛠️ Installation & Configuration

Pour que Home Assistant prenne en compte ce fichier, vous devez l'ajouter à votre configuration. Choisissez **UNE SEULE** des 3 méthodes ci-dessous selon votre architecture actuelle.

### Méthode 1 : Tout dans `configuration.yaml` (Débutant)
Si vous n'utilisez pas de fichiers séparés, copiez le contenu de `zigbee_sensors.yaml` directement dans `configuration.yaml` sous la clé `template:`.
⚠️ **Attention à l'indentation** : Vous devez ajouter 2 espaces au début de chaque ligne collée.
```yaml
template:
  - trigger: ...  <-- Notez le décalage
    platform: mqtt
    ...
```

### Méthode 2 : Via `templates.yaml` (Intermédiaire)
Si votre configuration ressemble à ça :
```yaml
template: !include templates.yaml
```
Copiez simplement tout le contenu de `zigbee_sensors.yaml` et collez-le à la fin de votre fichier `templates.yaml`.  
Aucune indentation supplémentaire n'est nécessaire (respectez juste l'alignement des tirets existants).

### Méthode 3 : Configuration Découpée « Merge List » (Expert)
C'est la méthode recommandée pour garder une configuration propre. Si vous avez ceci :
```yaml
template: !include_dir_merge_list templates/
```
1.  Créez un dossier `templates/` (s'il n'existe pas).
2.  Collez le fichier `zigbee_sensors.yaml` dans ce dossier.

> **Astuce de Migration** :
> Si vous migrez de la Méthode 2 vers la Méthode 3, vous pouvez simplement déplacer votre fichier `templates.yaml` existant vers le dossier `templates/`.
> Vous pourrez ensuite "découper" ce gros fichier par étapes ultérieurement.

Home Assistant fusionnera automatiquement tous les fichiers de ce dossier.

## ⚙️ Fonctionnement Technique

### 1. Le Capteur Maître (`sensor.z2m_battery_devices`)
Ce capteur écoute **deux sources MQTT** :
1.  `zigbee2mqtt02/bridge/devices` : Pour l'inventaire complet des appareils (déclenché rarement).
2.  `zigbee2mqtt02/+` : Pour le trafic temps réel (mise à jour de l'attribut `last_seen_registry`).

- **État** : Nombre total d'appareils sur batterie détectés.
- **Attributs clés** :
    - `last_seen_registry` : Dictionnaire stockant l'heure de dernier passage de chaque appareil qui "parle".
    - `devices` : Liste enrichie des appareils sur batterie (nom, statut, pile, date maintenance).
    - `raw_devices` : Données brutes de l'inventaire Z2M.

### 2. Le Capteur Réseau (`sensor.z2m_network_monitor`)
Ce capteur analyse `last_seen_registry` pour détecter les appareils "silencieux" depuis trop longtemps.

> [!NOTE]
> **Pourquoi un trigger `time_pattern` (toutes les 15 min) ?**
> L'attribut `last_seen_registry` est mis à jour à **chaque message MQTT** (potentiellement des centaines par minute).
> Sans ce timer, le capteur recalculerait inutilement à chaque message reçu, gaspillant des ressources.
> Le délai de 15 minutes est un bon compromis entre réactivité et performance.

### 3. Le Capteur d'Alertes (`sensor.zigbee_battery_alerts`)
Ce capteur filtre la liste du capteur maître pour ne sortir que les appareils nécessitant une intervention humaine.

**Critères d'alerte :**
- Appareil marqué `offline`.
- Niveau de batterie `< 15%`.
- Niveau de batterie inconnu (`?`).

## 📋 Comment tenir à jour les dates ?
Pour que la date de changement de pile s'affiche :
1. Allez dans l'interface **Zigbee2MQTT**.
2. Cliquez sur un appareil > **Settings** (Paramètres).
3. Dans le champ **Description**, écrivez par exemple : `pile 02/02/2026`.
4. Le capteur se mettra à jour automatiquement à la prochaine publication du bridge.

## 🔄 Comment forcer une actualisation ?
Un bouton **"Actualiser Monitoring Zigbee"** est créé automatiquement via le fichier `zigbee_sensors.yaml`.
Il est intégré directement dans la carte Dashboard fournie (voir section suivante).

En cliquant dessus, vous forcez le recalcul immédiat des capteurs. Vous pouvez vérifier l'action en observant l'attribut `last_check` du capteur `sensor.z2m_battery_devices` qui change à chaque appui.

> [!NOTE]
> **Après un redémarrage de Home Assistant**, il est normal que beaucoup d'appareils apparaissent en "INCONNU" ou "0%" pendant quelques minutes.
> C'est le temps que Home Assistant rétablisse la connexion avec tous les capteurs (qui peuvent être en veille).
> Une fois le système stabilisé, un clic sur le bouton "Actualiser" remettra tout d'équerre.

## 📊 Bonus : Carte Dashboard
Pour afficher un joli tableau récapitulatif sur votre Dashboard :
1. Créez une nouvelle carte **"Manuel"**.
2. Copiez le contenu du fichier `dashboard_card.yaml`.
3. Vous aurez un tableau avec statut, batterie colorée et date de maintenance.

![Aperçu du Monitoring Zigbee](dashboard_preview.png)

## 🤖 Automatisation : Rapport Journalier
Le fichier `zigbee_report.yaml` contient une automation clé en main qui :
1.  Se déclenche chaque soir (ex: 20h, configurable dans le fichier).
2.  Vérifie s'il y a des alertes en cours (`sensor.zigbee_battery_alerts > 0`).
3.  Génère un message sarcastique via le script **K-2SO**.
4.  Envoie une notification **Discord** détaillée (avec la liste des appareils) et une alerte visuelle sur **Awtrix**.

ℹ️ *Assurez-vous que ce fichier est bien pris en compte par votre configuration Home Assistant.*

---

## 📡 Carte Réseau (Bonus)

Une carte spécifique pour le moniteur réseau est disponible : `dashboard_network_card.yaml`

Elle affiche :
- Les appareils **hors-ligne** (non vus depuis 25h+)
- L'**activité récente** (les 10 derniers appareils ayant parlé)

Pour l'installer, suivez la même procédure que pour `dashboard_card.yaml`.

---

## 🧪 Templates de Debug

Le fichier `debug_templates.md` contient des templates prêts à copier/coller dans **Outils de développement > Modèle** pour diagnostiquer le système :

| Template | Utilité |
|----------|---------|
| 1. Vérification Globale | Aperçu rapide du système complet |
| 2. Raw Devices | Vérifie l'inventaire allégé |
| 3. Batteries | Vérifie la détection des entités |
| 4. Moniteur Réseau | Vérifie le registre last_seen |
| 5. Debug Appareil | Recherche un appareil spécifique |
| 6. Alertes | Liste les alertes actives |

---

## 🔧 Compatibilité

Ce projet a été testé avec différentes configurations et inclut des corrections pour :

| Correction | Description |
|------------|-------------|
| **Limite 16KB** | L'attribut `raw_devices` est allégé (sans icônes/bindings) |
| **Batteries textuelles** | Les valeurs `low`/`medium`/`high` sont converties en `~10`/`~50`/`~90` |
| **Entités sans device_class** | Recherche élargie des capteurs batterie |
| **Noms avec espaces** | Conversion automatique `espaces → underscores` pour matcher les entity_id |

---

## 📂 Liste des Fichiers

| Fichier | Description |
|---------|-------------|
| `zigbee_sensors.yaml` | Capteurs principaux (inventaire, alertes, réseau) |
| `dashboard_card.yaml` | Carte dashboard pour les batteries |
| `dashboard_network_card.yaml` | Carte dashboard pour le moniteur réseau |
| `zigbee_report.yaml` | Automation de rapport journalier |
| `debug_templates.md` | Templates de diagnostic |
| `README.md` | Cette documentation |
