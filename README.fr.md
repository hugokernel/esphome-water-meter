# ESPHome Water Meter

Read this in other languages: [English](README.md)

![Photo du boitier contenant l'électronique](images/boitier.jpg)

## Présentation

Si vous possédez un compteur d'eau (Sensu R-315 pour ma part), ce dernier dispose peut être d'un indicateur vous permettant de connaitre assez précisément la consommation courante.
Son fonctionnement est assez simple, un disque (voir photo) visible dans le cadran du compteur tourne à une vitesse proportionnelle au débit d'eau circulant.
Il est conçu pour que chaque révolution complète corresponde à un 1L d'eau.

Le disque est constitué d'une partie réfléchissante et d'une partie mat (voir photo ci-dessous), ainsi, en utilisant un capteur adapté, il est possible de détecter la rotation du disque et donc d'en déduire la consommation courante de votre compteur.

![Photo du compteur](images/compteur.jpg)

Afin de détecter la rotation le disque, j'ai utilisé un capteur [TCRT5000](docs/tcrt5000.pdf) disposant d'une LED émettrice IR et d'un photo-transistor dans le même boitier.
Il est possible de trouver ces capteurs sur des cartes autonomes avec un amplificateur opérationnel et une résistance ajustable pour un prix très intéressant.

J'ai collé au double face le capteur avec une méthode très approximative mais cela fonctionne très bien, j'ai cependant prévu de faire une pièce imprimable en 3D afin de faciliter l'installation.

![Photo du compteur avec le capteur posé dessus](images/installation.jpg)

## Caractéristiques

### Mesures de consommation

* Consommation journalière
* Consommation hebdomadaire
* Consommation mensuelle
* Consommation annuelle
* Consommation primaire / secondaire: Des compteurs qu'il est possible de remettre à 0 à tout moment afin d'effectuer des mesures en tout genre
* Consommation courante: Indique la consommation instantanée
* Dernière consommation: Indique la dernière consommation (la mesure continue tant que le compteur tourne sans interruption sans arrêt de plus de 5 minute)

### Statut

L'état de fonctionnement du compteur est visible via une LED WS2812 qui clignote en vert à intervalle régulier.

### Mesures annexes

Le compteur étant situé dans la cave, j'ai profité de cette installation pour mesurer la température et l'humidité via un capteur AM3220 (que je ne recommande pas d'ailleurs).

Un capteur de lumière (LDR) permet également de savoir si la lumière est allumé dans la cave indiquant alors un probable oubli d'extinction de cette dernière.
La mesure de luminosité se fait au moment ou la lumière est éteinte afin d'éviter que la LED de statut ne perturbe la mesure.

## Fonctionnement

Le fonctionnement est assez simple et la partie YAML la plus importante est reportée ci-dessous:

```yaml
binary_sensor:
  # TCRT5000 pulse counter
  # IO18 / GPIO18
  - platform: gpio
    id: water_pulse
    pin: GPIO18
    internal: true
    filters:
       - delayed_on_off: 50ms
       - lambda: |-
          id(main_counter_pulses) += x;
          id(secondary_counter_pulses) += x;
          id(daily_counter_pulses) += x;
          id(weekly_counter_pulses) += x;
          id(monthly_counter_pulses) += x;
          id(yearly_counter_pulses) += x;
          id(event_quantity) += x;
          return x;
    on_state:
       - script.execute: publish_states
```

Un capteur numérique est déclaré et utilise l'entrée / sortie numérique 18 sur laquelle est branchée le TCRT5000.

Un filtre permet de supprimer les impulsions parasites en provenance du capteur si son état n'est pas défini durant plus de 50 milli-secondes.

Une fois le filtre passé, on atterrit dans la lambda ou le code C est exécuté et qui consiste simplement à incrémenter des globales définies plus haut.

Une autre partie déclare un capteur de type compteur d'impulsion qui permet de connaître la consommation instantanée.

```yaml
sensor:
  # TCRT5000 pulse counter
  # IO18 / GPIO18
  - platform: pulse_counter
    id: water_pulse_counter
    name: "${friendly_name} water consumption"
    pin: GPIO18
    update_interval: 2sec
    internal_filter: 10us
    unit_of_measurement: "L/min"
    accuracy_decimals: 0
    icon: "mdi:water"
    filters:
      # Divide by 60
      - multiply: 0.0167
      - lambda: return abs(x);
```

## Fichiers

* watermeter.yaml: Le fichier de configuration ESPHome
* network.yaml: Les informations de votre réseau
* secrets.yaml: Les informations secrètes relatives à votre réseau
