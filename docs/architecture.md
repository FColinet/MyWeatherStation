# Document d'architecture — MyWeatherStation

## 1. Objectif du projet

Construire une station météo personnelle reposant sur un réseau de capteurs **Modbus RTU**
sur bus **RS485**, relié au réseau IP via une passerelle série/ethernet, et intégrant une
caméra IP pour l'observation visuelle.

Ce document décrit l'architecture globale du système : composants, topologie, protocoles,
adressage, flux de données, **architecture logicielle** (Go, plugins-sondes, Kubernetes,
InfluxDB, Home Assistant, API/site public, exports vers sites météo tiers), choix
techniques, contraintes et évolutions possibles.

> Documentation détaillée des registres et trames Modbus : voir [documentation.md](./documentation.md).

## 2. Vue d'ensemble

```
        ┌────────────────────────── CHAMP / EXTÉRIEUR ──────────────────────────┐
        │                                                                       │
        │  [BME280]   [SN-3000-FSJT]  [SN-3000-FXJT]   [SEM404]    [SEN0644]    │
        │  T°/H°/P°     Vent (vit.)     Vent (dir.)     Pluie        UV/Lux     │
        │   addr 10       addr 11         addr 12        addr 13      addr 14   │
        │       └──────────┴───────────────┴──────────────┴───────────┘         │
        │                                 │                                     │
        │                       Bus RS485 (2 fils A/B + GND)                    │
        │                       Modbus RTU — 9600 bauds                         │
        └─────────────────────────────────┬─────────────────────────────────────┘
                                          │
                                   ┌──────┴───────┐
                                   │   NB114      │
                                   │ Passerelle   │
                                   │ RS485→Eth    │
                                   └──────┬───────┘
        ┌─────────────────────────────────┼──────────── RÉSEAU IP ────────────┐
        │                                 │                                    │
        │        ┌────────────────────────┴───────────────────────┐            │
        │        │        Cluster Kubernetes (namespace           │            │
        │        │        weatherstation)                         │            │
        │        │  collector Go (plugins sondes) ──► InfluxDB v2 │◄─────────┐ │
        │        │      │  MQTT ──► Home Assistant                │          │ │
        │        │      ▼                                         │   Caméra │
        │        │  API REST ──► site public / Grafana / exports  │   RLC-840A│
        │        │              (WU, Infoclimat…)                 │  (RTSP)  │ │
        │        └────────────────────────────────────────────────┘──────────┘ │
        └─────────────────────────────────────────────────────────────────────────┘
```

## 3. Composants matériels

| Composant | Modèle | Rôle | Protocole | Adresse Modbus | Bauds |
|---|---|---|---|---|---|
| Passerelle | NB114 | Conversion RS485 ↔ Ethernet (RTU → TCP) | Modbus | — | — |
| Température / Humidité / Pression | BME280 | Mesures atmosphériques | RS485 (Modbus RTU) | 10 (0x0A) | 9600 |
| Vitesse du vent | SN-3000-FSJT-N01 | Anémomètre | RS485 (Modbus RTU) | 11 (0x0B) | 9600 |
| Direction du vent | SN-3000-FXJT-N01 | Girouette | RS485 (Modbus RTU) | 12 (0x0C) | 9600 |
| Pluviométrie | SEM404 | Capteur de pluie | RS485 (Modbus RTU) | 13 (0x0D) | 9600 |
| UV / Éclairement | SEN0644 | Rayonnement UV et lux | RS485 (Modbus RTU) | 14 (0x0E) | 9600 |
| Caméra | RLC-840A | Observation visuelle | IP (ONVIF / RTSP) | — | — |

Chaque capteur est alimenté en 12–24 V DC selon sa fiche technique
(voir les PDF dans [`docs/docs/`](./docs/)).

## 4. Architecture réseau

### 4.1 Couche terrain — Bus RS485 / Modbus RTU

- **Topologie** : bus multipoint en guirlande (daisy-chain), câble torsadé blindé 2 fils
  (A/B) + masse de signal commune à tous les équipements.
- **Protocole** : Modbus RTU maître/esclave. Les capteurs sont des esclaves ; le maître
  est la passerelle NB114 (ou le collecteur côté IP).
- **Paramètres série** : 9600 bauds, 8 data bits, pas de parité, 1 stop bit (8N1) —
  paramètres homogènes sur tout le bus.
- **Adressage** : adresses 10 à 14 (0x0A–0x0E), uniques sur le bus. L'adresse d'origine
  (0x01) et le changement de bauds se font via les trames décrites dans
  [documentation.md](./documentation.md).
- **Terminaison** : résistance de terminaison 120 Ω aux deux extrémités du bus ;
  résistance de polarisation (pull-up/pull-down sur A/B) si nécessaire.
- **Masse** : relier les GND des équipements entre eux pour éviter les tensions
  différentielles qui perturbent la communication.

### 4.2 Couche passerelle — NB114 (RS485 → Ethernet)

- Rôle : pont transparent Modbus RTU ↔ Modbus TCP (port 502).
- Le maître IP adresse chaque esclave via son unit id (= adresse Modbus RTU).
- Configuration IP : adresse statique recommandée hors plage DHCP pour fiabilité.
- Timeout de réponse à dimensionner : le poller ne doit pas enchaîner les requêtes
  avant réception/expiration de la réponse précédente (Modbus half-duplex).

### 4.3 Couche IP — Collecte et supervision

- Un **collecteur** (service `collector` du cluster Kubernetes, maître Modbus TCP) joue le
  rôle de maître applicatif : il interroge périodiquement les registres des capteurs via
  la passerelle (voir §7 Architecture logicielle).
- La caméra RLC-840A est exposée en RTSP/ONVIF sur le même réseau ; elle peut être
  intégrée au tableau de bord (image courante, time-lapse).
- Stockage : historisation des mesures dans **InfluxDB v2** (détail du schéma §7.7).
- Restitution : Grafana (interne), API REST + site public (externe), Home Assistant
  via MQTT (§7.8–7.9).

## 5. Flux de données

```
Capteur (esclave RTU) ──requête/registres──► Bus RS485 ──► NB114 ──Modbus TCP:502──►
Collecteur (polling N s) ──► Normalisation (échelle, signed/float, unités) ──►
Base de temps ──► API / Dashboard ──► Notifications (dépassement de seuils)
Caméra ──► RTSP ──► Snapshot / Enregistrement ──► Dashboard
```

Chaîne de traitement typique d'une mesure :

1. Lecture des registres (fonction 0x03 Read Holding Registers) ;
2. Décodage (16 bits, ordre MSW/LSW, éventuel float IEEE754 sur 2 registres) ;
3. Application du facteur d'échelle (ex. vitesse du vent en mm/s → m/s) ;
4. Contrôle de cohérence / rejet des valeurs aberrantes ;
5. Écriture horodatée en base ;
6. Affichage et alertes.

## 6. Adressage des registres (synthèse)

| Capteur | Grandeur | Registre | Unité |
|---|---|---|---|
| BME280 (10) | Humidité | 0 | %rH |
| BME280 (10) | Température | 1 | °C |
| BME280 (10) | Pression atmosphérique | 2 | hPa |
| SN-3000-FSJT (11) | Vitesse du vent | 0 | m/s |
| SN-3000-FXJT (12) | Direction (rose des vents) | 0 | — |
| SN-3000-FXJT (12) | Direction (degrés) | 1 | ° |
| SEM404 (13) | Pluviométrie | 0 | mm |
| SEN0644 (14) | Éclairement (haute valeur) | 2 | Lux |
| SEN0644 (14) | Éclairement (basse valeur) | 3 | Lux |

Détail complet des trames de configuration (changement d'adresse, baud rate) :
[documentation.md](./documentation.md).

## 7. Architecture logicielle

### 7.1 Principes directeurs

- **Langage unique : Go.** Toutes les briques applicatives maison sont écrites en Go
  (binaire statique, image containerisée légère `FROM distroless`, empreinte mémoire
  faible, concurrence native via goroutines/canaux — adaptée au polling séquentiel
  multi-capteurs).
- **Sondes = plugins.** Chaque périphérique (capteur Modbus, caméra…) est un plugin
  chargé dynamiquement, respectant une interface commune (`Sensor`). L'ajout d'un
  capteur ne nécessite ni recompilation du cœur, ni redémarrage complet du collecteur.
- **Cible Kubernetes.** Les services sont déployés sous forme de déploiements/StatefulSets
  dans un cluster K8s personnel ; l'état persistant (InfluxDB) passe par des volumes
  persistants, la config par ConfigMaps/Secrets.
- **Stockage : InfluxDB** (v2.x, API Line Protocol + Flux), seul moteur de séries
  temporelles retenu — plus d'alternative SQLite.
- **Ouverture vers l'extérieur.** Home Assistant (MQTT discovery), site public, API REST
  et export automatique vers les sites météo tiers (Weather Underground, Infoclimat,
  PWS Weather, OpenWeatherMap…).
- **Découplage acquisition / traitement / distribution** : chaque brique communique par
  contrats bien définis (bus interne, InfluxQL/Flux, MQTT), et peut être remplacée ou
  mise à l'échelle indépendamment.
- **Idempotence & reprise sur erreur** : le bus RS485 est half-duplex et sujet aux pannes
  temporaires ; le logiciel doit tolérer les timeouts, retries et redémarrages, sans
  perte de données (buffer disque/WAL).

### 7.2 Vue d'ensemble des microservices

```
                         ┌──────────────────────────────────────────────────────┐
   Bus RS485 ──► NB114 ──►│              collector (Go, Deployment)             │
                         │  plugin registry → sondes Modbus (plugins .so)       │
                         │  pipeline : décodage → dérivés → bus interne         │
                         └───────┬───────────────┬───────────────┬──────────────┘
                                 │ Line Protocol │ MQTT          │ gRPC/Go lib
                          ┌──────▼──────┐  ┌─────▼──────────┐    │
                          │ InfluxDB v2 │  │ Mosquitto/     │    │
                          │ (StatefulSet│  │ EMQTT + HA     │◄───┼── Home Assistant
                          │  + PVC)     │  │ discovery MQTT │    │
                          └──────┬──────┘  └────────────────┘    │
              ┌──────────────────┼───────────────────┐           │
      ┌───────▼───────┐  ┌───────▼────────┐  ┌───────▼────────┐  │
      │ api (Go)      │  │ exporter (Go)  │  │ site public    │  │
      │ REST /v1/*    │  │ WU, Infoclimat,│  │ (Astro/Hugo →  │  │
      │ (ClusterIP)   │  │ PWS Weather…   │  │ Cloudflare     │  │
      └───────┬───────┘  └────────────────┘  │ Pages/Netlify) │  │
              │                              └────────────────┘  │
      ┌───────▼───────┐                                          │
      │ Grafana       │◄── Flux/InfluxQL ── InfluxDB             │
      │ (dashboards)  │                                          │
      └───────────────┘        ingress-nginx / Traefik ──► exposition web (api,
                               site, Grafana)             docs, snapshots caméra)
```

Cinq déploiements Go + trois briques tierces (InfluxDB, broker MQTT, Grafana), tous dans
le namespace `weatherstation`. Le `collector` est le seul à toucher la passerelle NB114 ;
les autres consommateurs lisent InfluxDB ou le flux MQTT.

### 7.3 Composants logiciels

| Composant | Rôle | Technologie | Format K8s |
|---|---|---|---|
| `collector` | Charge les plugins-sondes, orchestre le polling Modbus TCP, publie les mesures normalisées sur le bus interne | Go 1.22+, `goburrow/modbus`, `hashicorp/go-plugin` | Deployment (1 réplique, `Recreate`) |
| Plugins-sondes | Un par périphérique : BME280, SN-3000-FSJT, SN-3000-FXJT, SEM404, SEN0644, caméra RLC-840A | Modules Go compilés en `.so` (`plugin` stdlib) | Fichiers montés via ConfigMap/PVC ou image initiale |
| `influxdb` | Base de séries temporelles, rétention, downsampling (tasks) | InfluxDB 2.x | StatefulSet + PVC |
| `api` | API REST publique/privée (dernières mesures, historique, statut capteurs) | Go (chi/echo) | Deployment + Service ClusterIP |
| `exporter` | Envoi périodique vers WU, Infoclimat, PWS Weather, OpenWeatherMap… | Go (clients HTTP, jobs cron) | CronJob (chaque site = un plugin d'export) |
| `mqtt-publisher` | Publication Home Assistant MQTT Discovery + états | Go + Eclipse Mosquitto (ou EMQX) | Deployment + Sidecar/Service |
| `site-public` | Site web vitrine (mesures live, graphiques, photos) | Astro/Hugo (statique) consommant l'API | Déployé hors cluster (Cloudflare Pages/Netlify) |
| `grafana` | Dashboards internes (Flux) | Grafana OSS | Deployment + PVC |
| `ingress` | Terminaison TLS (Let's Encrypt/cert-manager), routing api/site/grafana | ingress-nginx ou Traefik | Add-on cluster |

### 7.4 Système de plugins (sondes)

Chaque sonde est un plugin Go respectant une interface commune définie dans le module
partagé `pkg/pluginapi` :

```go
// pkg/pluginapi/plugin.go — contrat commun à toutes les sondes
package pluginapi

type Reading struct {
    Sensor  string      // ex. "bme280"
    Metric  string      // ex. "temperature"
    Value   float64     // grandeur physique décodée (SI)
    Unit    string      // ex. "°C"
    Quality QualityFlag // ok | suspect | timeout | crc_error
    TS      time.Time   // horodatage UTC de la lecture
}

type Descriptor struct {
    Name        string        // id unique, ex. "sensor.bme280"
    Version     string
    APIVersion  string        // version du contrat acceptée par le plugin
    ModbusSlave uint8         // adresse RTU portée par l'unit id TCP
    PollEvery   time.Duration // période souhaitée
    Config      []ConfigField // champs attendus (IP passerelle, registres…)
}

type Sensor interface {
    Descriptor() Descriptor
    Init(ctx context.Context, cfg map[string]string) error
    // Plan décrit les requêtes Modbus à émettre (start/count/fonction)
    Plan() []ModbusRequest
    // Decode transforme la réponse brute en grandeurs physiques
    Decode(req ModbusRequest, resp []uint16) ([]Reading, error)
    HealthCheck(ctx context.Context) error
}
```

**Deux mécanismes de chargement**, selon le besoin :

| Mécanisme | Description | Quand l'utiliser |
|---|---|---|
| `plugin` stdlib (`.so`) | Le collector charge des fichiers `.so` compilés avec `go build -buildmode=plugin`, depuis un répertoire monté (`/plugins`). Même version de toolchain Go exigée entre cœur et plugins. | Sondes « de confiance » développées dans le dépôt ; simple et rapide |
| `hashicorp/go-plugin` (RPC) | Chaque sonde tourne dans un **processus séparé** (eventuellement un sidecar container), communicant en gRPC/RPC avec le collector. Pas de contrainte de toolchain, isolation des pannes (un crash de plugin ne tue pas le collecteur). | Plugins expérimentaux ou tiers, dépendances natives lourdes (ex. caméra RTSP/ffmpeg) |

Règles de conception :

- **Versionnage explicite** : chaque plugin expose `Descriptor.Version` et une version de
  contrat (`APIVersion`) ; le collector refuse un plugin incompatible.
- **Manifeste** : un fichier `station.yaml` (monté en ConfigMap) déclare les instances de
  sondes actives, leurs adresses esclaves et leur configuration — le plugin fournit le
  code, le manifeste fournit les paramètres.
- **Isolation des erreurs** : tout appel plugin est exécuté avec timeout + `recover()` ;
  un plugin qui échoue plus de N fois est désactivé temporairement (circuit breaker) et
  signalé via MQTT et `/metrics`.
- **Testabilité** : un esclave Modbus TCP simulé (binaire Go `cmd/modbus-sim`) permet de
  tester chaque plugin sans matériel, en rejouant des trames réelles capturées.

### 7.5 Organisation du code (dépôt cible)

```
myweatherstation/                      # monorepo Go
├── docs/                              # ✔ existe déjà (architecture, registres, PDF)
├── cmd/
│   ├── collector/main.go              # service d'acquisition + gestion des plugins
│   ├── api/main.go                    # serveur REST
│   ├── exporter/main.go               # CLI/CronJob d'export vers sites tiers
│   ├── mqtt-pub/main.go               # publisher Home Assistant Discovery
│   └── modbus-sim/main.go             # esclave Modbus TCP simulé (tests)
├── pkg/
│   ├── pluginapi/                     # contrat Sensor / Reading / Descriptor (module partagé)
│   ├── modbuscli/                     # client Modbus TCP : pool, sérialisation, retries, timeouts
│   ├── pipeline/                      # bus interne (channels), sanity checks, grandeurs dérivées
│   ├── influx/                        # writer Line Protocol batching + WAL disque
│   ├── ha/                            # schémas MQTT discovery Home Assistant
│   ├── exporters/                     # clients WU, Infoclimat, PWS Weather, OWM… (interface Exporter)
│   └── config/                        # chargement YAML + env, validation
├── plugins/                           # une sonde = un sous-module Go compilable en .so
│   ├── bme280/
│   ├── sn3000fsjt/                    # anémomètre
│   ├── sn3000fxjt/                    # girouette
│   ├── sem404/                        # pluviomètre
│   ├── sen0644/                       # UV/lux
│   └── rlc840a/                       # caméra (snapshots RTSP, plugin RPC)
├── deploy/
│   ├── docker/                        # Dockerfile multi-stage par binaire (distroless)
│   └── k8s/
│       ├── manifests/                 # namespace, deployments, services, configmaps,
│       │                              # secrets, PVC InfluxDB, CronJobs, ingress,
│       │                              # network policies
│       └── helm/                      # chart global (valeurs par environnement)
├── web/                               # site public (Astro/Hugo) consommant l'API
├── station.yaml                       # manifeste des sondes (adresses, périodes, seuils)
├── go.work                            # workspace reliant cœur + plugins
└── Makefile                           # build, test, lint, docker-build, deploy
```

Règles :

- **Mapping des registres dans les plugins uniquement** : le cœur ne connaît aucun
  registre ; [documentation.md](./documentation.md) reste la source de vérité unique,
  chaque plugin l'implémente et ses tests unitaires la valident.
- **Configuration externalisée** : `station.yaml` (ConfigMap) + variables
  d'environnement/Secrets pour les tokens (clé WU, mot de passe InfluxDB…). Rien de codé
  en dur.
- **CI** : `golangci-lint`, tests unitaires (dont rejeu de trames réelles), builds
  reproductibles des images, scan de vulnérabilités (Trivy), déploiement GitOps
  (Argo CD ou Flux) depuis `deploy/k8s/`.

### 7.6 Flux applicatif (détail)

```
[ticker collector] → plan = registry.Sensors().Plan()   # ordonnancement des requêtes
for req in plan:                                        # file stricte : 1 transaction à la fois
    resp = modbuscli.Call(req, timeout=1s, retries=2)   # sérialisé vers la NB114
    if err: pipeline.Emit(Reading{Quality: timeout}); continue
    readings = plugin.Decode(req, resp)
    pipeline.SanityFilter(readings)                     # bornes physiques, sauts, cohérence
    pipeline.Derive(readings)                           # rosée, wind-chill, rafale max…
    → fan-out : influx.WAL.write | mqtt.publish(HA) | alerts.evaluate
```

- **Période de polling** : 5–10 s par défaut (déclarée par chaque plugin, surchargeable
  dans `station.yaml`).
- **Sérialisation stricte** : un seul goroutine émetteur vers la NB114 (bus half-duplex) ;
  les réponses sont dispatchées vers les plugins par `(unit id, fonction, start)`.
- **Écriture groupée** (batch ~30 s ou 1000 points) vers InfluxDB via l'API HTTP v2.
- **WAL local** : si InfluxDB est injoignable, les points sont écrits sur un
  PersistentVolume puis rejoués au retour — aucune perte pendant les maintenances.
- **Horodatage** : UTC partout ; l'horloge des pods suit le cluster (NTP/chrony sur les nœuds).

### 7.7 Modèle de données InfluxDB

Bucket `weather` + buckets agrégés produits par des **tasks Flux** planifiées :

```
weather,source=bme280,host=station,unit=id10 temperature=12.4,humidity=63.1,pressure=1013.2 1730000000000000000
weather,source=sn3000fsjt,host=station,unit=id11 wind_speed_ms=3.7,gust_ms=5.2 1730000000000000000
weather,source=sn3000fxjt,host=station,unit=id12 wind_dir_deg=215 1730000000000000000
weather,source=sem404,host=station,unit=id13 rain_mm=0.2,rain_rate_mmh=1.4 1730000000000000000
weather,source=sen0644,host=station,unit=id14 uv_index=2.1,lux=18400 1730000000000000000
weather,source=derived,host=station dewpoint_c=6.1,windchill_c=11.2,heatindex_c=13.0 1730000000000000000
weather,source=status,host=station sensor_ok=1,modbus_errors=0,wal_depth=0 1730000000000000000
```

Rétention & downsampling :

| Série | Rétention | Regroupement |
|---|---|---|
| Brute (5–10 s) | 90 jours | — |
| `weather_1m` | 2 ans | moyenne/min/max par minute |
| `weather_hourly` | 10 ans | moyenne/min/max + cumuls pluie par heure/jour |

- Tags : `source` (id plugin), `host`, `unit` (adresse Modbus) ; un champ par grandeur,
  toujours en unités SI ; horodatage **UTC** (cf. §9).
- Grandeurs dérivées (point de rosée, wind-chill, evapotranspiration…) calculées par le
  pipeline du collector, série `source=derived`.

### 7.8 Intégration Home Assistant

- **Protocole : MQTT + Home Assistant Discovery** (aucune intégration custom côté HA).
- Un broker léger (Mosquitto) tourne dans le namespace ; le service `mqtt-pub` publie :
  - `homeassistant/sensor/<id>/config` : découverte automatique d'une entité par grandeur
    (nom, unité, device_class, state_class, icône) — ex. `sensor.station_temperature`,
    `sensor.station_wind_speed`, `binary_sensor.station_bme280_online` ;
  - `weatherstation/state/<source>/<metric>` : états JSON (valeur, qualité, horodatage) ;
  - `weatherstation/status/<source>` : LWT `online`/`offline` pour surveiller chaque sonde.
- Chaque plugin déclare ses métadonnées HA dans son `Descriptor` → la configuration de
  découverte est générée automatiquement, sans double maintenance.
- La caméra RLC-840A est intégrée à HA indépendamment via ONVIF/RTSP (hors périmètre Go).

### 7.9 API et site public

**API REST (`api`, Go + chi)** :

| Endpoint | Visibilité | Description |
|---|---|---|
| `GET /v1/latest` | publique (rate-limitée, cache CDN) | Dernier relevé de toutes les sondes |
| `GET /v1/sensors/{id}/history?from=&to=&agg=` | privée (auth) | Séries agrégées (traduites en Flux) |
| `GET /v1/stats?period=` | publique | min/max/moyenne, cumul pluie jour/mois/année |
| `GET /v1/status` | privée | Santé des sondes, erreurs Modbus, profondeur WAL |
| `GET /metrics` | interne | Métriques Prometheus |

- API versionnée (`/v1`), documentée OpenAPI, pagination + ETag.
- **Site public** : générateur statique (Astro/Hugo) dans `web/`, buildé en CI et hébergé
  hors cluster (Cloudflare Pages / Netlify). Il consomme `/v1/latest` et `/v1/stats` en
  client-side (mesures live, graphiques légers, photo caméra via snapshots statiques).
  Le cluster n'est ainsi jamais exposé directement au trafic public ; seul l'endpoint
  public de l'API passe par l'Ingress (+ CDN).

### 7.10 Export vers les sites extérieurs

Le service `exporter` expose une interface `Exporter` — chaque site est un plugin
d'export — et s'exécute en **CronJob Kubernetes** :

```go
type Exporter interface {
    Name() string
    Validate(cfg map[string]string) error
    Export(ctx context.Context, batch WeatherSnapshot) (Report, error) // parsing ACK/throttle
}
```

| Site | Protocole d'envoi | Fréquence | Spécificités |
|---|---|---|---|
| Weather Underground | HTTPS GET/POST `rapidfire` | 1–5 min | Station ID + API key, ACK à parser, throttle minimal |
| Infoclimat (Acast) | POST `live2.php` (format compatible WU) | 1–5 min | Compte + clé Acast, conventions FR |
| PWS Weather | POST `upload/pwsupload.php` | 1–5 min | Cumuls jour/mois/année calculés côté Go |
| OpenWeatherMap | `stations/{id}/measure` (JSON) | 10 min | Station à créer au préalable |
| Windy / Weathercloud (option) | SDK/REST dédiés | 5–10 min | Bibliothèques tierces |

Comportements communs gérés par le framework d'export :

- **File de rejeu** sur PVC en cas d'échec réseau/HTTP ; clés d'idempotence (horodatage
  du lot) pour éviter les doublons après rejeu.
- **Respect des quotas** : backoff exponentiel + délai minimal entre deux envois.
- **Observabilité** : métriques `export_success_total{site}`, `export_latency_seconds` ;
  alerte si un site n'a rien reçu depuis > 30 min.
- **Secrets** : clés API en Secrets K8s, injectées en variables d'environnement.

### 7.11 Observabilité et exploitation

- **Logs structurés** : `log/slog` (JSON lines) avec niveau, sonde concernée, durée de
  transaction Modbus ; collecte par DaemonSet (Vector/Fluent Bit) vers Loki.
- **Métriques Prometheus** (`/metrics` sur chaque service Go) : latence Modbus, taux
  d'erreur par esclave, âge du dernier point par sonde, profondeur WAL, succès d'export
  par site. Alertmanager : capteur muet > 5 min, WAL > 1 h, export KO > 30 min.
- **Supervision** : Grafana (pilotage interne) ; le dashboard affiche l'âge du dernier
  point par sonde. Tracing OpenTelemetry optionnel sur la chaîne API → InfluxDB.
- **Sauvegarde** : `influx backup` en CronJob vers un stockage objet (S3/MinIO) ou restic.

### 7.12 Sécurité

- **NetworkPolicies** : seul le `collector` peut joindre la NB114 (port 502) ; seuls
  `api`/`grafana`/`exporter` accèdent à InfluxDB ; sorties Internet limitées au HTTPS
  des sites d'export ; rien n'entre depuis Internet sauf via l'Ingress.
- **Ingress + cert-manager** : terminaison TLS Let's Encrypt ; endpoints privés (Grafana,
  `/v1/history`, `/v1/status`) derrière authentification (OAuth2 Proxy ou VPN WireGuard/Tailscale).
- **La NB114 et la caméra ne sont jamais exposées** sur Internet (pas de port-forwarding).
- Identifiants par défaut changés (NB114, RLC-840A, Mosquitto, InfluxDB, Grafana) ;
  rotation des tokens d'export.
- **Moindres privilèges** : containers non root (`securityContext`), images distroless
  scannées (Trivy), RBAC limité au namespace `weatherstation`, secrets hors dépôt git
  (SOPS/Sealed Secrets ou Vault).
- Rate limiting et cache CDN sur les endpoints publics de l'API.

### 7.13 Déploiement Kubernetes

- **Namespace** dédié `weatherstation` ; ressources décrites dans un **chart Helm**
  (`deploy/k8s/helm/weatherstation`) avec valeurs par environnement.
- Formes de travail :

| Workload | Composants | Motif |
|---|---|---|
| Deployment `replicas: 1`, stratégie `Recreate` | collector, mqtt-pub | Le polling Modbus doit rester **unique** (bus half-duplex) ; RollingUpdate créerait deux maîtres simultanés |
| StatefulSet + PVC | influxdb, mosquitto, WAL du collector | État persistant, identité stable |
| Deployment (2 répliques, HPA éventuel) | api, grafana | Sans état, montée en charge horizontale possible |
| CronJob | exporter (par site), influx-backup | Échéances découplées du flux temps réel |

- **Probes** : `/healthz` (processus vivant) et `/readyz` (passerelle joignable, InfluxDB
  accessible, plugins chargés) ; le collector n'est prêt que si au moins une sonde répond.
- **Accès à la NB114 depuis les pods** : la passerelle est hors cluster → Service headless
  + EndpointSlice pointant vers son IP LAN statique, ou petit agent `modbus-proxy`
  (DaemonSet) sur le nœud connecté au VLAN de collecte. À figer selon la topologie réseau
  du cluster (cf. §11 questions ouvertes).
- **Stockage** : Longhorn/open-local si pas de stockage réseau ; anti-affinité pour
  séparer collector et InfluxDB sur des nœuds différents.
- **GitOps** : Argo CD (ou Flux) synchronise `deploy/k8s/` — tout changement passe par PR.
- **Images** : builds multi-stage (`CGO_ENABLED=0` sauf plugins `.so`), tags sémantiques,
  registre privé (Harbor/GHCR).

### 7.14 Trajectoire recommandée

1. **MVP** : collector Go + plugins natifs (`.so`) BME280 / anémomètre / girouette /
   SEM404 / SEN0644 → InfluxDB + Grafana sur le cluster ; validation des mappings
   registres sur le matériel réel.
2. **Phase 2** : MQTT + Home Assistant Discovery, API REST `/v1`, WAL disque, métriques
   Prometheus + alertes Alertmanager.
3. **Phase 3** : plugins d'export (WU, Infoclimat, PWS, OWM) en CronJobs avec file de
   rejeu ; site public Astro branché sur l'API ; snapshots caméra publiés sur le site.
4. **Phase 4** : plugins RPC (`hashicorp/go-plugin`) pour les sondes tierces,
   downsampling longue période, sauvegardes automatisées, second collecteur passif
   (chaud/froid).

## 8. Choix d'architecture et justifications

### 8.1 Volet matériel / réseau

- **RS485/Modbus plutôt que capteurs individuels sans fil** : robustesse du bus câblé en
  environnement extérieur (UV, humidité, orages), unicité du protocole, coût réduit,
  distances longues (jusqu'à ~1000 m à 9600 bauds).
- **Passerelle matérielle dédiée (NB114)** : évite de maintenir un convertisseur USB-RS485
  sur le serveur ; le bus reste indépendant de la disponibilité du collecteur.
- **Polling centralisé** : Modbus étant un protocole maître/esclave sans publication
  spontanée, un poller unique simplifie la planification et évite les collisions sur le bus.
- **Adresse IP statique pour la passerelle et la caméra** : fiabilité de la collecte.
- **Caméra sur le réseau IP** : la vidéo transite uniquement en IP, sans charger le bus RS485.

### 8.2 Volet logiciel

- **Go comme langage unique** : binaires statiques légers, idéals pour des containers
  distroless ; concurrence adaptée au polling ; typage fort pour les contrats de plugins ;
  écosystème mature (`goburrow/modbus`, client InfluxDB v2, SDK MQTT).
- **Sondes en plugins** : le parc capteurs évolue dans le temps ; un contrat `Sensor`
  stable permet d'ajouter un périphérique sans toucher au cœur ni redéployer tous les
  services (versionnage d'API + circuit breaker pour contenir les défauts).
- **Kubernetes comme cible** : déclaratif, GitOps, self-healing, stockage via PVC —
  cohérent avec l'infrastructure existante ; contrepartie assumée : le collector reste
  mono-réplique (bus half-duplex oblige).
- **InfluxDB v2** : modèle séries temporelles natif, tasks de downsampling intégrées,
  Flux pour les agrégats servis à l'API et à Grafana.
- **MQTT + Home Assistant Discovery** : standard de facto, zéro intégration custom à
  maintenir côté HA, LWT pour détecter une sonde muette.
- **Site public statique hors cluster + API publique filtrée** : isole le cluster du trafic
  Internet, réduit coûts et surface d'attaque ; les exports vers les sites tiers sont
  découplés du temps réel (CronJobs, file de rejeu, quotas).

## 9. Contraintes et points de vigilance

- **Half-duplex** : une seule transaction à la fois sur le bus RS485 ; sérialiser les
  requêtes et prévoir des timeouts (ex. 500 ms – 1 s) avec retries limitées.
- **Mono-master garanti** : en K8s, veiller à ce qu'il n'existe jamais deux pods collector
  actifs (stratégie `Recreate`, `replicas: 1`, éventuellement verrou via Lease API).
- **Qualité du signal** : blindage, terminaisons 120 Ω, éloignement des sources de
  parasites (onduleurs, variateurs) ; préférer un trajet séparé du 230 V.
- **Alimentation** : alimenter les capteurs au plus près ; section de câble adaptée aux
  chutes de tension sur les longues portées.
- **Environnement** : boîtiers IP65/IP66 pour les raccordements extérieurs, presse-étoupes,
  protection foudre/surtension (parafoudre RS485) sur les longues lignes aériennes.
- **Horodatage** : synchroniser les nœuds du cluster (NTP/chrony) pour des séries
  temporelles fiables.
- **Évolutivité de l'adressage** : les adresses 1–9 et 15+ restent libres pour de futurs
  capteurs ; garder 9600 8N1 sur tout le bus.
- **Plugins `.so`** : contrainte de version Go identique entre cœur et plugins (et appel
  à cgo) ; figer la toolchain en CI ou basculer les plugins sensibles vers le mode RPC.
- **Sites extérieurs** : les protocoles d'upload tiers peuvent changer sans préavis ;
  isoler chaque plugin d'export et le tester contre des mocks HTTP en CI.

## 10. Évolutions possibles

- Ajout de capteurs sur le bus existant (niveau neige, qualité de l'air, batterie…)
  sous forme de nouveaux plugins, sans modification du collector.
- Redondance de la collecte (second collecteur passif avec Leader Election K8s).
- Exposition GraphQL ou WebSocket temps réel en complément de l'API REST.
- Grandeurs dérivées avancées : evapotranspiration, degree-days, nowcasting du vent.
- Nouveaux sites d'export (Weathercloud, Windy, Meteologix…) via plugins dédiés.
- Validation par consensus multi-capteurs / ML légère (détection de givre, outlier).

## 11. Questions ouvertes

À trancher avant l'implémentation :

1. **Topologie réseau du cluster** : les pods ont-ils un accès direct au VLAN de la NB114
   (EndpointSlice externe) ou faut-il un agent proxy (DaemonSet) sur le nœud connecté au
   réseau de collecte ?
2. **Distribution des plugins** : compilés dans l'image du collector, livrés sur un PVC
   monté, ou téléchargés à chaud avec vérification de signature/checksum ?
3. **Broker MQTT** : Mosquitto (léger) vs EMQX (auth fine, persistance) selon la taille
   du parc Home Assistant.
4. **Hébergement du site public** : Cloudflare Pages/Netlify vs auto-hébergé derrière
   l'Ingress.
5. **Périmètre initial des exports** : WU + Infoclimat suffisent-ils pour le lancement,
   ou PWS Weather/OpenWeatherMap dès la phase 3 ?

## 12. Références documentaires

- [Documentation Modbus complète (registres & trames)](./documentation.md)
- Fiches techniques : [`docs/docs/`](./docs/)
  - [NB114 — Manuel utilisateur](./docs/NB114_UserManual_FR_V1.6.pdf)
  - Fiches capteurs (BME280, SN-3000-FSJT/FXJT, SEM404, SEN0644, RLC-840A)
- Spécifications protocoles d'export :
  - Weather Underground — Personal Weather Database (`rapidfire`)
  - Infoclimat — Acast (upload temps réel)
  - PWS Weather — `pwsupload.php`
  - OpenWeatherMap — Stations API
- Standards & bibliothèques :
  - Spec Modbus Application Protocol V1.1b3 ; Home Assistant MQTT Discovery ;
    InfluxDB Line Protocol v2 / Flux
  - Go : `plugin` (stdlib), `hashicorp/go-plugin`, `goburrow/modbus`,
    `influxdb-client-go`, `eclipse/paho.mqtt.golang`, `go-chi/chi`
  - Kubernetes : Helm, Argo CD/Flux, cert-manager, ingress-nginx, Prometheus Operator
