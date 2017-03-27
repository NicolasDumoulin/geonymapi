# GeonymAPI

Implémentation minimaliste d'une API de conversion *geonym* <-> lat/lon disponible sur http://api.geonym.fr/

L'API repose sur le module python Falcon, et un module geonym.py (utilisable hors API par tout script python).


## À propos des Géonym

Communiquer la position d'un lieu se fait en général par une adresse, mais en l'absence d'adresse on ne peut que recourir aux coordonnées GPS peut faciles à mémoriser et trop variables dans leurs formats (dégrés, dégrés+minutes, dégrés+minutes+secondes, etc).

Afin de combler ces deux manques, une **position géographique peut être traduite en une série de caractères, mémorisable et facile à communiquer**.

Contrairement à d'autres solutions, **Géonym est LIBRE**, basé sur un algorithme sous licence LGPL ([openspostcode](http://www.openspostcode.org)) et ne dépendant pas d'une API. Le calcul est simple et peut se faire directement côté client.


## Comment ça marche ?

Une grille fixe est choisie pour une région donnée. Dans notre cas cette grille correspond à la France métropolitaine et englobe la majeure partie des eaux territoriales.

Elle est découpée en 25 zones de (5 x 5), elles mêmes découpées en 25 et ainsi de suite. Voir: http://www.geonym.fr/visu pour visualiser les zones et le fonctionnement (n'hésitez pas à zoomer).

Le territoire couvert est ainsi découpé au final en milliards de carrés de 3m x 3m (plus de 60 milliards sur le territoire métropolitain). Pour les DOM, ceux-ci sont remis dans les zones 0,1,4 de la grille qui ne recouvrent pas de territoire métropolitain.

A chaque zone on fait correspondre un chiffre ou une consonne qui forme ainsi une série de caractères. Plus cette série est longue, plus le *geonym* correspond donc à une petite zone géographique (25 fois plus petite en surface, donc des côtés 5 fois plus petits).

- P = zone de 234km x 234km
- PP = zone de 47km x 47km inclue dans celle de 'P'
- PP7 = zone de 9 x 9km inclue dans celle de 'PP'
- ...
- PP7K-RF4V = zone de 3m x 3m correspondant à l'entrée de la Tour Mirabeau à Paris

Voir: http://www.geonym.fr/visu/#19/48.84687/2.27924

L'algorithme utilisé est identique à OpenPostcode (licence LGPL), avec un alphabet différent limités aux chiffres de 0 à 9 et aux consonnes non ambigües (0/D/O, 1/I, 2/Z, 4/A, 5/S, 6/G, 8/B). Un caractère de contrôle peut s'ajouter en option pour vérifier que le geonym est sans erreur.

![Alphabet geonym et répartition en spirale](https://raw.githubusercontent.com/geonym/visugeonym/master/img/geonym_small.png)


## Installation

`git clone https://github.com/geonym/geonymapi.git`

pour installer les modules pythons nécessaires:

`pip install -r requirements.txt`

L'idéal est de fonctionner dans un virtualenv python3.


## Test / serveur

`gunicorn geonymapi:app -b 0.0.0.0:1405 -w 4`

ou pour fonctionner en daemon avec rechargement automatique lorsque le code est modifié:

`gunicorn geonymapi:app -b 0.0.0.0:1405 -w 4 --reload -D`


## Utilisation de l'API

Conversion geonym > lat/lon:
- http://api.geonym.fr/PP7K-RF4R
- http://api.geonym.fr/?geonym=PP7K-RF4R

Conversion lat/lon > geonym:
- http://api.geonym.fr/?lat=48.8&lon=2.35

Conversion X/Y lambert > geonym:
- http://api.geonym.fr/?x=647095&y=6860995

Conversion adresse > geonym via géocodage intermédiaire:
- http://api.geonym.fr/?adresse=39+quai+andré+citroen+paris

Réponse au format geojson:
- **north/west/south/east** sont les limites du geonym
- **lat/lon** correspond au centre de la bbox du geonym
- **X/Y** correspond au coordonnées Lambert 93 du centre de la bbox du geonym
- **params** contient les paramètres de la grille (alphabet et limites géographiques)
- **geocode** contient la réponse du géocodeur
- **reverse** contient l'adresse la plus proche trouvée par géocodage inversé (pour les géonym d'au moins 6 caractères), sauf si reverse=no passé en paramètre de la requête.

```
{
    "geocode": null,
    "geometry": {
        "coordinates": [
            2.2790806399999997,
            48.846832486400004
        ],
        "type": "Point"
    },
    "params": {
        "alpha": "456783NPR92MXTC1LWVD0KJHF",
        "max_lat": 51.45,
        "max_lon": 9.8,
        "min_lat": 40.91,
        "min_lon": -5.45
    },
    "properties": {
        "east": 2.2791001599999996,
        "geonym": "PP7KRF4R",
        "lat": 48.846832486400004,
        "lon": 2.2790806399999997,
        "north": 48.846845977600005,
        "south": 48.8468189952,
        "west": 2.2790611199999997,
        "x": 647094,
        "y": 6860995
    },
    "reverse": {
        "attribution": "BAN",
        "features": [
            {
                "geometry": {
                    "coordinates": [
                        2.279092,
                        48.846828
                    ],
                    "type": "Point"
                },
                "properties": {
                    "city": "Paris",
                    "citycode": "75115",
                    "context": "75, Île-de-France",
                    "distance": 0,
                    "housenumber": "39",
                    "id": "75115_0318_91e007",
                    "label": "39 Quai André Citroën 75015 Paris",
                    "name": "39 Quai André Citroën",
                    "postcode": "75015",
                    "score": 0.9999999996245265,
                    "street": "Quai André Citroën",
                    "type": "housenumber"
                },
                "type": "Feature"
            }
        ],
        "licence": "ODbL 1.0",
        "limit": 1,
        "source": "http://api-adresse.data.gouv.fr",
        "type": "FeatureCollection",
        "version": "draft"
    },
    "type": "Feature"
}

```
