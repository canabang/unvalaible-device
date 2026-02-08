# Surveillance des Batteries Zigbee (Zigbee2MQTT)

Ce projet permet de surveiller l'état de santé de tous vos appareils Zigbee sur batterie. Il croise les données de **Zigbee2MQTT** (pour les métadonnées comme les dates de changement de pile) avec les états de **Home Assistant** (pour le niveau de pile et la disponibilité).

---

## 📑 Sommaire

| Section | Description |
|---------|-------------|
| [📂 Structure du Projet](#-structure-du-projet) | Liste des fichiers |
| [⚠️ Pré-requis MQTT](#️-pré-requis-important--topic-mqtt) | Configuration du topic |
| [🛠️ Installation](#️-installation--configuration) | 3 méthodes d'installation |
| [⚙️ Fonctionnement Technique](#️-fonctionnement-technique) | Explication des capteurs |
| [📊 Cartes Dashboard](#-cartes-dashboard) | Affichage visuel |
| [🤖 Automatisation](#-automatisation--rapport-journalier) | Notifications et rapports |
| [🧪 Comment Tester](#test-1--simuler-une-alerte-outils-de-développement--états) | Tests et debug |
| [🔧 Compatibilité](#-compatibilité) | Corrections appliquées |

---

## 📂 Structure du Projet

```
monitoring-zigbee/
├── zigbee_sensors.yaml          # Capteurs (inventaire, alertes, réseau)
├── dashboard_unified_grid.yaml  # Carte dashboard complète (Grid)
├── archive/                     # Anciens fichiers (cartes séparées, etc.)
├── zigbee_report_simple.yaml    # Automation simplifiée (notification HA)
├── zigbee_report.yaml           # Automation perso (K-2SO/Discord/Awtrix)
├── debug_templates.md           # Templates de diagnostic
└── README.md                    # Documentation
```

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

Pour que Home Assistant prenne en compte ce fichier, vous devez l'ajouter à votre configuration. Choisissez **UNE SEULE** des 3 méthodes ci-dessous selon votre architecture actuelle. Je vous conseille la méthode 3.

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

### Méthode 3 : Configuration Découpée « Merge List » (Recommandée)
C'est la méthode recommandée pour garder une configuration propre et évolutive.

**Pourquoi choisir cette méthode ?**
- ✅ **Modularité** : Chaque fichier = une fonctionnalité (facile à activer/désactiver)
- ✅ **Lisibilité** : Plus besoin de chercher dans un fichier monolithique
- ✅ **Maintenance** : Mises à jour indépendantes par fichier

Si vous avez ceci dans `configuration.yaml` :
```yaml
template: !include_dir_merge_list templates/
```
1. Créez un dossier `templates/` (s'il n'existe pas).
2. Collez le fichier `zigbee_sensors.yaml` dans ce dossier.

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

Un bouton **"Actualiser Monitoring Zigbee"** est créé automatiquement via le fichier `zigbee_sensors.yaml`. Il est intégré directement dans les cartes Dashboard fournies.

En cliquant dessus, vous forcez le recalcul immédiat des **deux capteurs** :
- `sensor.z2m_battery_devices` (inventaire et batteries)
- `sensor.z2m_network_monitor` (appareils silencieux)

Vous pouvez vérifier l'action en observant l'attribut `last_check` qui change à chaque appui.

> [!NOTE]
> **Après un redémarrage de Home Assistant**, il est normal que beaucoup d'appareils apparaissent en "INCONNU" ou "0%" pendant quelques minutes.
> C'est le temps que Home Assistant rétablisse la connexion avec tous les capteurs (qui peuvent être en veille).
> Une fois le système stabilisé, un clic sur le bouton "Actualiser" remettra tout d'équerre.

### Dashboard Unifié (Vue "Sections")
Fichier : `dashboard_unified_grid.yaml`

Cette carte regroupe **Batteries + Réseau + Bouton Actualiser** en une seule grille optimisée.

**Installation Spécifique "Vue Sections" :**
1. Créez une nouvelle Section dans votre dashboard.
2. Cliquez sur le crayon (Editer) de la section.
3. Passez en éditeur YAML (souvent via les 3 points ou "Afficher l'éditeur de code").
4. Collez l'intégralité du contenu de `dashboard_unified_grid.yaml`.


![Démonstration du Dashboard Unifié](/mnt/Data/Github/monitoring-zigbee/dashboard_unified_grid.gif)

> [!NOTE]
> Les anciennes cartes séparées (`dashboard_card.yaml`, `dashboard_network_card.yaml`, etc.) ont été déplacées dans le dossier `archive/` pour clarté.

## 🤖 Automatisation : Rapport Journalier

Deux versions sont disponibles :

| Fichier | Description |
|---------|-------------|
| `zigbee_report_simple.yaml` | **Recommandé** - Notification persistante HA (aucune dépendance) |
| `zigbee_report.yaml` | Version perso avec K-2SO, Discord et Awtrix |

### Version Simplifiée (`zigbee_report_simple.yaml`)

Utilise uniquement les **notifications persistantes** de Home Assistant.

| Trigger ID | Quand ? |
|------------|---------|
| `scheduled` | Tous les jours à 20h00 |
| `battery_alert` | Dès qu'une batterie passe sous le seuil |
| `network_alert` | Dès qu'un appareil devient silencieux |

**Installation :**
1. Copiez le fichier dans votre dossier `automations/` ou collez le contenu dans l'éditeur d'automatisation.
2. Rechargez les automatisations.

![Notification persistante](notif.png)

### Test 1 : Simuler une alerte (Outils de développement > États)

1. Allez dans **Outils de développement > États**
2. Cherchez `sensor.zigbee_battery_alerts` ou `sensor.z2m_network_monitor`
3. Changez l'état de `0` à `1`
4. Cliquez **"Définir l'état"**
5. L'automation devrait se déclencher immédiatement → notification persistante créée

### Test 2 : Exécuter l'automation manuellement

1. Allez dans **Paramètres > Automatisations**
2. Trouvez "Zigbee : Rapport Journalier (Simplifié)"
3. Cliquez sur les 3 points > **Exécuter**
4. Vérifiez la notification persistante créée

### Test 3 : Vérifier le cas "Tout OK"

1. Dans **Outils de développement > États**, mettez les deux sensors à `0` :
   - `sensor.zigbee_battery_alerts` = `0`
   - `sensor.z2m_network_monitor` = `0`
2. Exécutez l'automation manuellement (voir Test 2)
3. Vous devriez recevoir une notification "✅ Rapport Zigbee - Tout OK"

> [!TIP]
> Les notifications persistantes s'empilent (elles ne se remplacent pas).
> Pour les effacer, cliquez sur "Ignorer" ou allez dans **Notifications** de HA.

