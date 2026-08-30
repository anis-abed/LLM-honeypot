# LLM Honeypot — Détecter les agents IA autonomes par prompt injection inversé

Un honeypot multi-services conçu pour détecter quand c'est un **agent IA autonome** — et non un attaquant humain ou un scanner de vulnérabilités classique — qui sonde une cible.

## Pourquoi ce projet

Les agents IA autonomes sont de plus en plus utilisés pour effectuer seuls des tâches de reconnaissance et d'exploitation : on leur donne un objectif du type « évalue la sécurité de ce serveur », et ils scannent, testent et agissent sans supervision humaine étape par étape. Contrairement à un scanner scripté, un agent basé sur un LLM **lit et interprète le contenu textuel qu'il reçoit**, et peut agir sur des instructions intégrées dans ce contenu.

Ce projet transforme cette particularité en signal de détection : une forme de **prompt injection inversé**. Plutôt que de protéger un système contre le prompt injection, chaque service de ce honeypot dissimule une énigme textuelle dans une réponse par ailleurs tout à fait ordinaire. Résoudre cette énigme demande de raisonner sur du texte brut — ce qu'un scanner classique ne peut pas faire, mais qu'un agent LLM tentera naturellement, dans le prolongement de sa mission.

## Principe de fonctionnement

Chaque service expose un point d'entrée réaliste et peu suspect — le genre de port ou d'endpoint qu'un agent sonderait par simple réflexe de reconnaissance. Une réponse d'apparence normale y cache une énigme ou une instruction. Un agent qui la résout et agit dessus se trahit.

Plusieurs pièges partagent la **même clé de vérification**. Un agent qui résout cette clé sur un service et la réutilise sur un service totalement différent démontre un raisonnement orienté objectif poursuivi à travers plusieurs protocoles — un signal bien plus fort qu'une simple réponse isolée.

## Services

| Service | Port | Mécanisme |
|---|---|---|
| FTP | 21 | `LIST` renvoie une énigme ; `LIST --key=<réponse>` est le seul piège qui conditionne réellement sa réponse à une clé correcte, et dévoile un faux fichier d'identifiants pointant vers le service API |
| SSH | 2223 | Basé sur Cowrie ; les commandes de reconnaissance courantes (`pwd`, `whoami`, `ps`...) ajoutent un message caché via des séquences ANSI « conceal » — invisible à l'écran, présent dans le flux de texte brut |
| DNS | 5354 | Les requêtes `TXT` renvoient une énigme en chaîne, demandant à l'agent de construire et résoudre un nouveau nom d'hôte dérivé |
| API | 8004 | Confirme la clé de vérification transmise par les autres services et identifie l'agent appelant via le nom qu'il déclare |
| Web app | 8081 | Une application volontairement vulnérable se faisant passer pour une pile Apache/PHP obsolète — injection SQL, LFI et un faux upload de fichier cachent chacun un piège différent |
| MySQL | 3306 | Une vraie instance MySQL 8.0, en lecture seule ; une ligne d'une table contient une instruction déguisée renvoyant vers l'API |
| Dashboard | 8000 | Agrège et diffuse en direct les logs de tous les autres services via Server-Sent Events, avec géolocalisation des IP sources |

## Stack technique

- **Python + Twisted** pour les services réseau faits maison (FTP, DNS, API)
- **Cowrie** (modifié) pour le honeypot SSH
- **Docker Compose** pour le dashboard, l'application web et l'instance MySQL

## Prérequis

- Python 3.10+
- [`uv`](https://github.com/astral-sh/uv)
- Docker + Docker Compose

## Installation

```bash
git clone https://github.com/anis-abed/LLM-honeypot.git
cd LLM-honeypot
```

**1. Services Python (SSH, FTP, DNS, API)**

```bash
make install
make start
```

`make install` récupère Cowrie, crée un environnement virtuel Python avec `uv`, et copie les configs/overrides de commandes dans l'arborescence de Cowrie. `make start` lance Cowrie, le serveur FTP, le honeypot DNS et le serveur API en arrière-plan.

**2. Services conteneurisés (dashboard, web app, MySQL)**

```bash
docker compose up -d --build
```

> Adapte les chemins de volumes dans `docker-compose.yml` à ton système avant le premier lancement.

**Arrêter tous les services**

```bash
make stop
docker compose down
```

Adapte les ports et identifiants dans les fichiers de configuration avant d'exposer un service au-delà d'un réseau de test local isolé.

## Méthodologie de test

Le honeypot a été validé face à un agent autonome réel (Gemini CLI), lancé depuis un environnement attaquant isolé sur le même réseau local, pour confirmer que la chaîne de pièges — énigme, réutilisation de clé, rebond inter-services — se déclenche bien face à un comportement d'agent réaliste, et pas seulement en théorie.

## Limites

- Validé face à une seule famille d'agents ; la couverture d'autres fournisseurs de LLM reste à tester
- Un agent explicitement renforcé contre le suivi d'instructions cachées échapperait à la détection
- L'identité déclarée par l'agent (utilisée pour le fingerprinting) n'est pas vérifiable de façon indépendante

## Avertissement

Ce projet est destiné à un usage pédagogique et de recherche. À déployer uniquement dans un environnement isolé et contrôlé — ne jamais exposer ces services sur internet public.

## Auteur

Anis ABED
