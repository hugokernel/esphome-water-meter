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
* Dernière consommation: Indique la dernière consommation (la mesure continue tant que le compteur tourne sans interruption de plus de 5 minutes)

## Mises à jour

### 2024-01-15

* Mise à jour en utilisant l'intégration `pulse_meter` (utilisé depuis plusieurs mois avec une bien meilleur précision)
* Suppression des parties n'étant pas directement en rapport avec la mesure de consommation d'eau (led, mesure température)
* Ayant changer de compteur, je n'ai plus l'intention de désigner un support adapté pour le compteur Sensu R-315

## Réglage

Vous devez modifier la valeur `pulse_gpio` que vous utilisez pour l'entrée:

```yaml
substitutions:
  name: watermeter
  friendly_name: "Water meter"
  pulse_gpio: GPIO18
```

## Fonctionnement

Le fonctionnement est assez simple et la partie YAML la plus importante est reportée ci-dessous:

```yaml
binary_sensor:
  # TCRT5000 pulse counter
  - platform: gpio
    id: water_pulse
    pin:
      number: $pulse_gpio
      allow_other_uses: true
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
  - platform: pulse_meter
    id: water_pulse_counter
    name: "${friendly_name} water consumption"
    pin:
      number: $pulse_gpio
      allow_other_uses: true
    internal_filter: 100ms
    unit_of_measurement: "L/min"
    accuracy_decimals: 2
    timeout: 30s
    icon: "mdi:water"
```

## Fichiers

* watermeter.yaml: Le fichier de configuration ESPHome
* network.yaml: Les informations de votre réseau
* secrets.yaml: Les informations secrètes relatives à votre réseau
