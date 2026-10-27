# MyWeatherStation

Projet de station météo personnelle basée sur un réseau **Modbus RTU** sur bus **RS485**, relié au réseau IP via une passerelle.

## Architecture

```
[Capteurs RS485 (Modbus RTU)] ---> [Passerelle NB114 RS485 -> Ethernet] ---> [Réseau IP]
                                                                     \--> [Caméra IP RLC-840A]
```

| Composant | Modèle | Protocole | Adresse Modbus | Baud |
|---|---|---|---|---|
| Passerelle | NB114 | RS485 → Ethernet | - | - |
| Température / Humidité / Pression | BME280 | RS485 (Modbus) | 10 (0x0A) | 9600 |
| Vitesse du vent | SN-3000-FSJT-N01 | RS485 (Modbus) | 11 (0x0B) | 9600 |
| Direction du vent | SN-3000-FXJT-N01 | RS485 (Modbus) | 12 (0x0C) | 9600 |
| Pluviométrie | SEM404 | RS485 (Modbus) | 13 (0x0D) | 9600 |
| UV / Éclairement | SEN0644 | RS485 (Modbus) | 14 (0x0E) | 9600 |
| Caméra | RLC-840A | IP | - | - |

## Documentation

- [Documentation complète du réseau Modbus](docs/documentation.md) — registres, adresses et trames de configuration (changement d'adresse, baud rate) de chaque capteur.
- Fiches techniques des équipements dans le dossier [`docs/docs/`](docs/docs/) :
  - [NB114 (passerelle)](docs/docs/NB114_UserManual_FR_V1.6.pdf)
  - [SN-3000-FSJT-N01 (vitesse du vent)](docs/docs/SN-3000-FSJT-N01.pdf)
  - [SN-3000-FXJT-N01 (direction du vent)](docs/docs/SN-3000-FXJT-N01.pdf)
  - [SEM404 (pluviométrie)](docs/docs/SEM404.pdf)
  - [SEN0644 (UV)](docs/docs/SEN0644.pdf)
  - [RLC-840A (caméra)](docs/docs/RLC-840A.pdf)

## Structure du dépôt

```
.
├── README.md
└── docs/
    ├── documentation.md      # Schéma de principe, registres & trames Modbus
    └── docs/                 # Fiches techniques PDF des équipements
```
