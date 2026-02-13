---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Welcome to Slidev
info: |
  ## Slidev Starter Template
  Presentation slides for developers.
download: true
  Learn more at [Sli.dev](https://sli.dev)
# apply UnoCSS classes to the current slide
class: text-center

# https://sli.dev/features/drawing

drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# duration of the presentation
duration: 35min
---

# Projet Timothé
## Architecture de Monitoring Solaire Thermique

Une solution **Open Source** de bout en bout :
du capteur physique au tableau de bord.

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->


---

# 🎯 Objectifs du Projet

Le projet vise à fournir une stack complète pour surveiller et optimiser les installations solaires thermiques.

*   **Acquisition :** Lire les données des régulateurs solaires (souvent propriétaires).
*   **Transmission :** Standardiser les données via MQTT.
*   **Stockage :** Historisation performante (Time Series).
*   **Visualisation :** Tableaux de bord pour l'analyse énergétique.
*   **Analyse automatisée :** Alerte à l'aide de technologie de l'IA.

---
layout: two-cols
---
# 🏗️ Architecture Globale

Le flux de données suit ce chemin :

1.  **Capteur (Edge) :** ESP32 connecté au régulateur solaire (DL-Bus/VBus).
2.  **Transport :** Broker MQTT.
3.  **Pont (Bridge) :** Service Rust (`rmqttconnector`) pour l'ingestion.
4.  **Base de données :** TimescaleDB (PostgreSQL).
5.  **IHM :** Grafana.

::right::


![](/archi.svg){width=500}

---

![](/archi.svg)


---

# 🔌 1. La Couche Matérielle (Edge)

Basé sur le dépôt : `esphomesolarthermalpanel`

*   **Rôle :** Interface entre le régulateur solaire et le réseau.
*   **Hardware :**
    *   Microcontrôleur : **ESP32** (ou compatible).
    *   Isolation : Optocoupleur (ex: 817) pour protéger le bus de communication.
    *   Connexion : WiFi ou Ethernet (WT32-ETH01).
*   **Protocoles supportés :**
    *   **DL-Bus** (Technische Alternative, etc.) via composant custom.
    *   **VBus** (Resol) via composant natif ESPHome.

---
layout: two-cols
---
# 🛡️ Choix Matériel : WT32-ETH01

Pour garantir la robustesse de l'acquisition, le choix s'est porté sur la carte **WT32-ETH01**.


![](/w32-eth0.webp){width=300}


::right::


*   **Pourquoi l'Ethernet ?**
    *   Les chaufferies sont souvent en sous-sol (mauvaise réception WiFi).
    *   Les ballons d'eau chaude agissent comme des cages de Faraday.
    *   La connexion filaire garantit une transmission des données sans latence ni perte.
*   **Les atouts de la carte :**
    *   **Tout-en-un :** ESP32 + Port Ethernet (LAN8720) intégrés sur le PCB.
    *   **Coût réduit :** Solution très économique (< 5 €).
    *   **Compacité :** Facile à intégrer dans un boîtier sur rail DIN.


---

# ⚡ Sécurité & Isolation : L'Optocoupleur

Le raccordement direct entre un système de chauffage et un microcontrôleur comporte des risques. L'utilisation d'un **optocoupleur** (ex: PC817) est indispensable.

*   **Le Rôle :**
    Transmettre le signal binaire via la lumière, sans contact électrique direct.
*   **Pourquoi est-ce critique ?**
    1.  **Adaptation de tension :** Le bus de données (VBus, DL-Bus) fonctionne souvent à des tensions supérieures (12V/24V) aux 3.3V tolérés par l'ESP32.
    2.  **Protection du matériel :** En cas de surtension côté chaudière, l'optocoupleur "grille" à la place de l'ESP32.
    3.  **Isolation Galvanique :** Empêche les "boucles de masse" (ground loops) qui perturbent les mesures analogiques.

---

# ⚙️ 2. Configuration ESPHome

L'intelligence du capteur repose sur **ESPHome**.

### 🔌 Qu’est-ce que ESPHome ?

**ESPHome** est un framework open-source permettant de programmer facilement des microcontrôleurs :

- ESP8266
- ESP32
- BK72xx 
- RP2040
- RTL87xx

via de simples fichiers **YAML**.

👉 Il transforme un microcontrôleur en objet connecté sans écrire de code complexe.

---

# ⚙️ Principe de fonctionnement

1. Définition de la configuration en YAML
2. Compilation automatique
3. Flash sur l’ESP
4. Communication réseau (WiFi, ethernet, lora)

---

# 🧠 Philosophie du projet

- Open-source
- Simplicité
- Abstraction du hardware
- Extensibilité
- Développement rapide
- Forte communauté


---

## 📡 Pourquoi ESPHome pour le monitoring solaire ?

ESPHome permet :

- Lecture de données du contrôleur:
  - puissance
  - températures
  - ...
- Acquisition temps réel et prétraitement local
- Envoi des données en réseau

---

# ⚙️ 2. Configuration ESPHome

L'intelligence du capteur repose sur **ESPHome**.

*   Utilisation d'un composant personnalisé `sensordlbus` ou vbus.
*   Définition des capteurs dans le YAML (Températures, Statut pompe).
*   Publication automatique vers MQTT.

```yaml
# Extrait de configuration (YAML)
sensor:
  - platform: sensordlbus
    dl_pin: GPIO5
    model: "UVR61-3"
    temperature_1:
      name: "Température Capteur"
    on_value:
      then:
        mqtt.publish_json: ...
```

---

# Un serveur mqtt écrit aussi en rust 

- https://github.com/rmqtt/rmqtt

---

# 🌉 3. Le Pont MQTT vers Base de Données

Basé sur le dépôt : `rmqttconnector`

Un service haute performance écrit en **Rust**.

*   **Pourquoi Rust ?** Sécurité mémoire, performance, faible empreinte ressource.
*   **Fonctionnement :**
    *   Souscription aux topics MQTT (wildcards supportés).
    *   Authentification MQTT & Reconnexion automatique.
    *   Utilisation de **Tokio** (Async I/O) et **SQLX**.

---

# 🗺️ Mapping Dynamique (Rust)

Le connecteur Rust ne nécessite pas de recompilation pour ajouter des capteurs. Il utilise un fichier `mappings.json` pour lier les topics MQTT aux utilisateurs/équipements.

**Exemple de mapping :**
```json
[
  { 
    "topic": "devices/solar/1", 
    "user_id": 1 
  },
  { 
    "topic": "devices/heater/#", 
    "user_id": 2 
  }
]
```
*Permet une architecture multi-tenant.*

---

# 🗄️ 4. Stockage : TimescaleDB

Les données sont stockées dans **PostgreSQL** avec l'extension **TimescaleDB** pour les séries temporelles.

**Schéma de données (JSONB) :**
Plutôt que de créer une colonne par capteur, on utilise le format JSON binaire pour la flexibilité.

```sql
CREATE TABLE metrics (
 time TIMESTAMPTZ,
 user_id INT,
 device_id VARCHAR(256),
 data JSONB -- Stocke {"temp1": 45.2, "pump": 1, ...}
);

SELECT create_hypertable('metrics', by_range('time'));
```

---

# 📊 5. Visualisation & Monitoring

Les données stockées permettent de générer des tableaux de bord via **Grafana**.

**Métriques clés visualisées :**
*   🌡️ **Températures :** Capteur (Toit), Ballon (Haut/Bas), Retour.
*   ⚡ **Énergie :** Gain solaire journalier/hebdomadaire.
*   ⚙️ **Actionneurs :** État des pompes (On/Off ou PWM).

---

# 📊 5. Visualisation & Monitoring

![](/grafana.webp)

---

# 🚀 Résumé des avantages

1.  **Ouverture :** Pas de cloud propriétaire, tout est hébergeable localement (Docker) sur architecture arm64 ou amd64.
2.  **Modularité :**
    *   Le capteur ESP32 peut être remplacé/modifié facilement.
    *   La passerelle mqtt est prête pour la scalabité.
    *   Le pont Rust est générique pour tout projet IoT MQTT -> SQL.
3.  **Performance :**
    *   Rust assure une ingestion rapide et une portabilité pour différents types d'architecture.
    *   TimescaleDB assure des requêtes rapides sur l'historique.
4.  **Flexibilité :** Le schéma JSONB permet d'ajouter des métriques sans changer la structure de la BDD.

---

# 🛠️ Installation sur Site : Procédure en 3 Étapes

Le déploiement d'un nœud de monitoring suit un processus standardisé pour garantir la connectivité sans configuration complexe sur place.

### 1️⃣ Préparation (En amont)
*   **État initial :** Le nœud est livré en mode "Hotspot WiFi" (SSID: `timothe`).
*   **Pré-requis réseau :** Collecte des IP/Gateway et validation de l'ouverture du **Port 1883** (Sortant vers MQTT).
*   **Firmware :** Le centre IT génère un firmware "spécifique site" envoyé par email au technicien.

### 2️⃣ Installation Physique (Technicien)
1.  **Raccordement :** Alimentation 220V + Bus Données (DL/VBus).
2.  **Connexion :** Via smartphone sur le WiFi `timothe` (mdp: `timothe`).
3.  **Flashage local :** Upload du firmware reçu via l'interface `http://timothe.local`.

### 3️⃣ Finalisation (À distance)
*   Le nœud redémarre et se connecte au réseau du site (Ethernet/WiFi).
*   **Provisioning final :** Le centre IT pousse la configuration complète et les mises à jour via MQTT.

---

# 🔄 Architecture de Mise à Jour (OTA)

Une fois le nœud connecté au réseau du client, toute la maintenance se fait à distance via le protocole MQTT.

**Topologie de communication :**

```text
[ Nœud Site ] ⇄ [ Routeur Client ] ⇄ [ Internet ] ⇄ [ Serveur MQTT ]
                                     (Port 1883)
```

**Flux de mise à jour finale :**

1.  **Check :** Le technicien valide que le nœud est visible sur le serveur.
2.  **Push :** Le Centre IT publie un ordre de mise à jour sur un topic MQTT dédié.
3.  **Pull :** Le nœud télécharge et installe automatiquement la version finale du firmware.

> **Avantage :** Aucune intervention physique nécessaire pour les ajustements de réglages capteurs.

---

<!-- _class: lead -->
# Merci
### Liens vers les projets
- [github.com/barais/esphomesolarthermalpanel](https://github.com/barais/esphomesolarthermalpanel)
- [github.com/barais/rmqttconnector](https://github.com/barais/rmqttconnector)

```