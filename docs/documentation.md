# Réseau modbus

## Introduction



## Schéma de principe




## Passerelle

```
Modèle: NB114
Protocol: RS485 to Eth
```

[Documentation](./docs/NB114_UserManual_FR_V1.6.pdf)

## Station météo

### Capteur de température/humidité/pression atmosphérique

```
Modèle: BME280
Protocol: RS485
Adresse: 10 (0A h)
Baud: 9600
```

#### Adresses

- Humidité: 0
- Température: 1
- Pression atmosphérique: 2

#### Trames

- ***Changement d'adresse***

| Adresse Code | fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0X01 | 0X06 | 0X07 | 0XD0 | 0X00 | 0X0A | 0X09 | 0X40 |

- ***Changement du baud rate***

| Adresse Code | fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0X0A | 0X06 | 0X07 | 0XD1 | 0X00 | 0X02 | 0X59 | 0X8A |

### Capteur de vitesse du vent

```
Modèle: SN-3000-FSJT-N01
Protocol: RS485
Adresse: 11 (0B h)
Baud: 9600
```

[Documentation](./docs/SN-3000-FSJT-N01.pdf)

#### Adresses

- Vitesse du vent: 0

#### Trames

- ***Changement d'adresse***

| Adresse Code | fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0X01 | 0X06 | 0X07 | 0XD0 | 0X00 | 0X0B | 0XC8 | 0X80 |

- ***Changement du baud rate***

| Adresse Code | fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0X0B | 0X06 | 0X07 | 0XD1 | 0X00 | 0X02 | 0X59 | 0X8A |

### Capteur de direction du vent

```
Modèle: SN-3000-FXJT-N01
Protocol: RS485
Adresse: 12 (0C h)
Baud: 9600
```

[Documentation](./docs/SN-3000-FXJT-N01.pdf)

#### Adresses

- Direction du vent (Rose des vents): 0
- Direction du vent (Degré): 1

#### Trames

- ***Changement d'adresse***

| Adresse Code | fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0X01 | 0X06 | 0X07 | 0XD0 | 0X00 | 0X0C | 0X89 | 0X42 |

- ***Changement du baud rate***

| Adresse Code | fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0X0C | 0X06 | 0X07 | 0XD1 | 0X00 | 0X02 | 0X59 | 0X8A |

### Capteur de pluviométrie

```
Modèle: SEM404
Protocol: RS485
Adresse: 13 (0D h)
Baud: 9600
```

[Documentation](./docs/SEM404.pdf)

#### Adresses

- : 0

#### Trames

- ***Changement d'adresse***

| Adresse<br>Code | Fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0X01 | 0X06 | 0X07 | 0XD0 | 0X00 | 0X0D | 0X48 | 0X82 |

- ***Changement du baud rate***

| Adresse<br>Code | Fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0X0D | 0X06 | 0X07 | 0XD1 | 0X00 | 0X02 | 0X59 | 0X8A |

### Capteur UV

```
Modèle: SEN0644
Protocol: RS485
Adresse: 14 (0E h)
Baud: 9600
```

[Documentation](./docs/SEN0644.pdf)

#### Adresses

- Eclairage (Valeur haute): 2
- Eclairage (Valeur basse): 3

#### Trames

- ***Changement d'adresse***

| Adresse<br>Code | Fonction | Registre<br>1er bit | Registre<br>2nd bit | Position<br>1er bit | Position<br>2nd bit | CRC<br>1er bit | CRC<br>2nd bit |
| - | - | - | - | - | - | - | - |
| 0x01 | 0x06 | 0x00 | 0x64 | 0x00 | 0x0E | 0x49 | 0xD1 |

### Caméra

```
Modèle: RLC-840A
Protocol: IP
```

[Documentation](./docs/RLC-840A.pdf)
