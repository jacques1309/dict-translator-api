#  API de Traduction par Dictionnaire

API REST de traduction de mots basée sur des dictionnaires de correspondances caractère par caractère. Le projet est conteneurisé avec Docker et déployé derrière un load balancer Nginx répartissant la charge sur plusieurs instances de l'application.

---

## 1. Introduction

Le but de ce projet est de fournir un **traducteur de mots configurable**. Le principe est simple : on définit d'abord un **dictionnaire** qui associe chaque caractère à sa traduction, puis on envoie un mot à traduire via l'API.

Le fonctionnement se fait en deux temps :

1. **Créer un dictionnaire** - on fournit un nom et une table de correspondances `key` -> `valeur`.
2. **Traduire un mot** - l'API parcourt chaque caractère du mot et le remplace par sa valeur dans le dictionnaire choisi.

L'exemple de référence est le **code Morse**. Voici le dictionnaire :

```json
{
  "dictionnary": "morse",
  "table": [
    { "key": "A", "valeur": ".-" },
    { "key": "B", "valeur": "-..." },
    { "key": "C", "valeur": "-.-." },
    { "key": "D", "valeur": "-.." },
    { "key": "E", "valeur": "." },
    { "key": "F", "valeur": "..-." },
    { "key": "G", "valeur": "--." },
    { "key": "H", "valeur": "...." },
    { "key": "I", "valeur": ".." },
    { "key": "J", "valeur": ".---" },
    { "key": "K", "valeur": "-.-" },
    { "key": "L", "valeur": ".-.." },
    { "key": "M", "valeur": "--" },
    { "key": "N", "valeur": "-." },
    { "key": "O", "valeur": "---" },
    { "key": "P", "valeur": ".--." },
    { "key": "Q", "valeur": "--.-" },
    { "key": "R", "valeur": ".-." },
    { "key": "S", "valeur": "..." },
    { "key": "T", "valeur": "-" },
    { "key": "U", "valeur": "..-" },
    { "key": "V", "valeur": "...-" },
    { "key": "W", "valeur": ".--" },
    { "key": "X", "valeur": "-..-" },
    { "key": "Y", "valeur": "-.--" },
    { "key": "Z", "valeur": "--.." }
  ]
}
```

> Le dictionnaire Morse est par ailleurs chargé automatiquement au démarrage de l'application (seed), donc il est disponible dès le premier lancement.

---

## 2. Contexte

Ce projet a été réalisé dans un **cadre scolaire**, dans le cadre d'un exercice DevOps portant sur la conteneurisation et le load balancing d'une application web.

---

## 3. Stack technique

| Composant | Technologie |
|-----------|-------------|
| API | FastAPI (Python 3.13) |
| Serveur ASGI | Uvicorn |
| Base de données | MySQL |
| ORM | SQLAlchemy |
| Load balancer | Nginx (stratégie `least_conn`) |
| Admin BDD | phpMyAdmin |
| Orchestration | Docker Compose |

**Architecture :** Nginx reçoit les requêtes sur le port `5000` et les répartit entre **5 instances** du service FastAPI, qui partagent une même base MySQL.


---

## 4. Installation de A à Z

### Prérequis

- [Docker](https://docs.docker.com/get-docker/)

### Étapes

**1. Cloner le dépôt**

```bash
git clone https://github.com/jacques1309/dict-translator-api.git
```

**Puis rentrer dans le dossier du projet**

```bash
cd dict-translator-api.git
```

**2. Lancer l'application**

Tout est automatisé : un seul script lance l'application, la base de données et le load balancer.

```bash
docker compose up -d --build
```

Cette commande va :
- construire l'image de l'API FastAPI ;
- démarrer la base MySQL et créer la base `dictionnaire` ;
- créer automatiquement les tables et charger le dictionnaire Morse ;
- lancer **5 réplicas** de l'API derrière Nginx.

**3. Vérifier que tout tourne**

| Service | URL |
|---------|-----|
| API (via load balancer) | http://localhost:5000 |
| Documentation Swagger | http://localhost:5000/docs |
| phpMyAdmin | http://localhost:8090 |

Un appel à la racine doit renvoyer :

```json
{ "msg": "Devops - projet de Jacques Lin" }
```

**4. Arrêter l'application**

```bash
docker compose stop          # arrête les conteneurs
docker compose down     # arrête et detruit tous les containers
docker compose down -v  # arrête, detruit tous les containers et supprime les volumes !
```

---

## 5. Utilisation : créer un dictionnaire et traduire

Toute la documentation interactive est disponible sur **http://localhost:5000/docs** (Swagger UI), qui permet de tester chaque endpoint directement depuis le navigateur.

### Étape 1 - Le dictionnaire Morse

Le dictionnaire `morse` est déjà chargé au démarrage. On peut donc passer directement à la traduction. Pour créer un autre dictionnaire, envoie une table de correspondances `key` -> `valeur` au format vu en introduction.

### Étape 2 - Traduire un mot

**Endpoint :** `POST /trad/`

**Corps de la requête :**

```json
{
  "word": "SOS",
  "dictionnary": "morse"
}
```

> Le champ `dictionnary` vaut `morse` par défaut : il peut être omis si l'on traduit en Morse.

**Exemple avec `curl` :**

```bash
curl -X POST http://localhost:5000/trad/ \
  -H "Content-Type: application/json" \
  -d '{"word": "SOS", "dictionnary": "morse"}'
```

**Réponse :**

```json
{
  "word": "SOS",
  "dictionnary": "morse",
  "trad": "...---..."
}
```

### Autres exemples

| Mot | Traduction Morse |
|-----|------------------|
| `SOS` | `...---...` |
| `HELLO` | `....`·`.`·`.-..`·`.-..`·`---` |
| `OK` | `---` `-.-` |

> ⚠️ La traduction se fait caractère par caractère, sans séparateur entre les lettres : `OK` devient `----.-`. Les caractères absents du dictionnaire (chiffres, espaces, accents…) sont simplement ignorés.

---

## Structure du projet

```
.
├── app/
│   ├── main.py              # Point d'entrée FastAPI + init BDD + seed Morse
│   ├── databases/           # Connexion MySQL (SQLAlchemy)
│   ├── models/              # Modèles ORM (Dict, DicLine, Trad) + schémas Pydantic
│   ├── routers/             # Routes HTTP (/trad)
│   ├── handlers/            # Couche d'aiguillage
│   ├── services/            # Logique de traduction
│   ├── repositories/        # Accès aux données
│   ├── middlewares/         # Dépendance get_db
│   └── utils/               # Seed du dictionnaire Morse
├── nginx/nginx.conf         # Configuration du load balancer
├── docker-compose.yml       # Orchestration (5 réplicas API + LB + MySQL)
├── dockerfile               # Image de l'API
└── requirements.txt
```
