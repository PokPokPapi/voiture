# Climatisation à distance — e-Golf 2019

But : démarrer/arrêter la climatisation de l'e-Golf depuis le cellulaire, sans Car-Net.

## Pourquoi l'app ne marche plus

Le problème ne vient pas de l'app : c'est le modem de l'auto. L'OCU (module télématique derrière le
combiné d'instruments) est un modem 2G/3G. Les réseaux 2G/3G ont été fermés, donc le module ne
peut plus parler à personne, avec ou sans app. VW n'offre aucun module 4G de remplacement pour
l'e-Golf. Selon des propriétaires, Car-Net n'aurait même jamais été offert sur les e-Golf canadiens.

## La solution : remplacer le modem mort par un ESP32

L'OCU envoyait simplement des messages sur le **bus CAN confort** pour réveiller l'auto et lancer la
climatisation. Un ESP32 branché sur ce même bus peut envoyer les mêmes messages.

```
Cell ──(MQTT / HTTPS)──> ESP32 + LTE ──(CAN confort 500 kbps)──> Gestion batterie ──> Clim
```

### Base technique existante

- [thomasakarlsen/e-golf-comfort-can](https://github.com/thomasakarlsen/e-golf-comfort-can) :
  e-Golf MY2018 (même génération que 2019). « Controlling climatisation (start, stop and settings) »
  est indiqué **DONE**, avec des enregistrements de trames `start_climate.csv` / `stop_climate.csv`.
- Bus confort : paire torsadée orange/vert (CAN-H) et orange/brun (CAN-L), 500 kbps, identifiants
  29 bits, protocole VW BAP.
- Le port OBD ne convient pas : la passerelle ne transmet rien quand le contact est coupé.
- L'OCU se trouve derrière le combiné d'instruments : deux vis en bas, puis tirer (clips rigides).
- OVMS (Open Vehicle Monitoring System) a un module e-Golf, mais la prise en charge du bus confort
  et de la clim y est encore en cours de développement. À surveiller, pas encore prêt à brancher.

### Matériel (environ 120 à 170 CAD)

| Pièce | Rôle | Prix approx. |
|---|---|---|
| LilyGO T-SIM7600 (ESP32 + LTE-M/4G) | cerveau + connexion partout | 70–90 CAD |
| Transceiver CAN SN65HVD230 | interface avec le bus | 5–10 CAD |
| Convertisseur 12 V → 5 V (faible courant de repos) | alimentation | 10 CAD |
| Raccord en T ou fils sur le connecteur OCU | connexion au bus sans couper | 15–25 CAD |
| Carte SIM données IoT | connectivité | 5–10 CAD/mois |

Si l'auto dort toujours à portée du WiFi de la maison, un ESP32 seul avec un transceiver CAN
suffit (environ 30 CAD) et il n'y a pas de SIM à payer.

## Plan par étapes

1. **Dès maintenant, gratuit** : utiliser les heures de départ de l'e-Manager dans l'auto (menu CAR).
   La clim est prête à l'heure choisie. Activer la climatisation sur batterie si l'auto n'est pas
   branchée.
2. **Écoute seulement** : brancher l'ESP32 en mode *listen-only* (en lecture seule), enregistrer le
   trafic avec SavvyCAN (format GVRET) et vérifier qu'on voit les mêmes ID que le dépôt
   (`17330301`, `17332501`, `1B000067`…).
3. **Capture de référence** : enregistrer le trafic pendant un démarrage de clim fait depuis
   l'e-Manager, puis comparer avec `start_climate.csv`.
4. **Rejeu** : renvoyer la séquence de réveil et de démarrage, contact coupé, auto verrouillée.
   Faire de même pour l'arrêt.
5. **Commande depuis le cell** : un broker MQTT (HiveMQ Cloud gratuit ou Home Assistant), avec un
   bouton dans un raccourci iOS ou un widget. Afficher en retour l'état de charge et l'état de la clim.
6. **Gestion de l'énergie** : ESP32 en veille profonde, réveil du modem en LTE-M/PSM et contrôle
   périodique. Viser un courant de repos de quelques mA, sinon la batterie 12 V se vide.

## Risques

- Les trames du dépôt sont le travail d'une seule personne et ne sont pas documentées. Valider en
  écoute avant de transmettre quoi que ce soit.
- Une mauvaise trame sur le bus confort peut enregistrer des codes d'erreur ou empêcher l'auto de
  dormir (vidange de la 12 V). Dans ce cas, débrancher l'ESP32, puis effacer les codes avec OBDeleven
  ou VCDS.
- Ne rien couper dans le faisceau : utiliser un raccord en T.
