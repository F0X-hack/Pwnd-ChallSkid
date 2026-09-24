# Documentation complète — Pwnd-Challenges-Skid

## Table des matières

1. [Présentation](#1-présentation)
2. [Principe de fonctionnement](#2-principe-de-fonctionnement)
3. [Architecture du projet](#3-architecture-du-projet)
4. [Le manifest (Manifest V3)](#4-le-manifest-manifest-v3)
5. [Le popup — interface utilisateur](#5-le-popup--interface-utilisateur)
6. [Le script — popup.js](#6-le-script--popupjs)
7. [Les modes](#7-les-modes)
8. [Personnalisation](#8-personnalisation)
9. [Installation & débogage](#9-installation--débogage)
10. [Dépannage](#10-dépannage)
11. [FAQ](#11-faq)
12. [Sécurité — Avertissement légal](#12-sécurité--avertissement-légal)
13. [Crédits](#13-crédits)

---

## 1. Présentation

**Pwnd-Challenges-Skid** est une extension Google Chrome (Manifest V3) qui injecte des cookies de déblocage sur le site **challenges-kids.fr**. Une fois les cookies posés, tous les challenges sont accessibles sans avoir à les résoudre.

Elle propose **3 modes** accessibles depuis l'icône de l'extension :

| Mode | Effet |
|------|-------|
| `UNLOCK ALL` | Débloque tous les challenges des 5 catégories |
| `NUCLEAR OPTIONS` | Débloque un nombre configurable de challenges `network` (par défaut 301) |
| `CREDIT` | Affiche le crédit du script |

> **À usage éducatif uniquement.**

---

## 2. Principe de fonctionnement

Le site `challenges-kids.fr` est une plateforme de challenges de **cybersécurité** (network, web, crypto, stegano, culture générale). L'accès à chaque challenge est contrôlé côté serveur par la présence d'un cookie (une session PHP).

### Comment l'accès est contrôlé

Chaque challenge correspond vraisemblablement à un cookie dont le **nom** suit le schéma :

```
/<categorie>/chall<numero>
```

Exemple pour le mode `UNLOCK ALL` :

```
/network/chall1      → network, challenge #1
/network/chall2      → network, challenge #2
...
/web/chall1          → web, challenge #1
...
```

La **valeur** du cookie est libre : le serveur ne vérifie probablement que l'**existence** du cookie. L'extension fixe la valeur `Hacked by FoXhack` comme marqueur de déblocage.

### Les cookies de session

Deux cookies de base sont également injectés :

- `disclaimer = accepted` → accepte les avertissements / popups du site.
- `PHPSESSID = 44728fag636e2c03tbdulf8prf` → identifiant de session PHP (capturé à la création du site et réutilisé tel quel).

> **Important** : si le site régénère la session et invalide ce `PHPSESSID`, l'injection pourra échouer. Cette valeur est à ajuster (voir [Personnalisation](#8-personnalisation)).

---

## 3. Architecture du projet

```
skid-extension/
│
├── manifest.json          ← Configuration de l'extension (Manifest V3)
├── popup.html             ← Interface de la fenêtre popup
├── popup.css              ← Styles de l'interface (thème "console/hacker")
├── popup.js               ← Logique : construction et injection des cookies
├── banner.html            ← Fichier de travail / export du cadre SVG décoratif
├── README.md              ← Présentation courte pour GitHub
├── DOCUMENTATION.md       ← Ce document
└── images/
    └── icon.png           ← Icône de l'extension (16, 48, 128 px)
```

| Fichier | Rôle |
|---------|------|
| `manifest.json` | Déclare les permissions, l'action, l'icône |
| `popup.html` | Structure HTML du menu affiché au clic sur l'icône |
| `popup.css` | Mise en forme : thème sombre, typographie monospace, cadre néon |
| `popup.js` | La **logique métier** : génération des cookies et injection |
| `banner.html` | Démo isolée du cadre SVG (non chargé par l'extension) |

---

## 4. Le manifest (Manifest V3)

```json
{
  "manifest_version": 3,
  "name": "Pwnd-Challenges-Skid",
  "version": "1.0",
  "description": "Injecte les cookies de déblocage sur challenges-kids.fr (modes UNLOCK ALL / NUCLEAR).",
  "permissions": ["cookies"],
  "host_permissions": ["https://www.challenges-kids.fr/*", "https://challenges-kids.fr/*"],
  "action": {
    "default_popup": "popup.html",
    "default_title": "SKID Unlocker",
    "default_icon": "images/icon.png"
  },
  "icons": {
    "16": "images/icon.png",
    "48": "images/icon.png",
    "128": "images/icon.png"
  }
}
```

Détail des champs :

| Champ | Valeur | Explication |
|-------|--------|-------------|
| `manifest_version` | `3` | Format actuel de Chrome (le V2 est déprécié) |
| `permissions` | `["cookies"]` | Autorise la lecture/écriture des cookies via l'API `chrome.cookies` |
| `host_permissions` | les deux domaines | Autorise l'accès aux cookies des deux variantes du domaine (`www` et racine) |
| `action.default_popup` | `popup.html` | Fichier affiché quand on clique sur l'icône |
| `action.default_title` | `SKID Unlocker` | Infobulle au survol de l'icône |
| `action.default_icon` | `images/icon.png` | Icône de la barre d'outils |
| `icons` | 16 / 48 / 128 | Icônes du gestionnaire d'extensions, Page Chrome Web Store... |

> **Note** : aucune permission `tabs` n'est déclarée, mais `chrome.tabs.create()` reste utilisable — ouvrir un onglet ne nécessite pas la permission `tabs` en MV3.

---

## 5. Le popup — interface utilisateur

`popup.html` est le menu affiché au clic sur l'icône. Il est autonome (pas de `background.js` ni de `content script`).

### Structure

1. **Cadre décoratif SVG** (`.banner-frame`) — des rectangles pointillés violets et des coins accentués, le tout en arrière-plan.
2. **Bannière ASCII** — le titre `Pwnd-Challenges-Skid` rendu en ASCII art, coloré en 3 parties : rouge / blanc / violet.
3. **Logo** — `images/icon.png`, affiché à droite de la bannière.
4. **3 boutons de mode** (`#unlock`, `#nuclear`, `#credit`).
5. **Zone NUCLEAR** (`#nuclear-box`) — masquée par défaut, contient un champ numérique `#max-chall`.
6. **Zone CREDIT** (`#credit-box`) — masquée par défaut, texte ASCII de remerciements.
7. **Zone de statut** (`#status`) — message de retour visuel (succès / erreur).
8. **Pied de page** — liens vers `guns.lol/foxhack` et `github.com/F0X-hack`.

### Classes CSS importantes

| Classe | Effet |
|--------|-------|
| `.btn` | Style des boutons (fond sombre, bordure violette, hover inversé) |
| `.hidden` | `display: none` — masque les zones optionnelles |
| `.status.ok` | Message vert (`#00ff66`) |
| `.status.err` | Message rouge (`#ff4444`) |
| `.credit` | Texte de crédit en orange (`#ffaa00`) |

### Éléments exposés au JavaScript

| ID | Rôle |
|----|------|
| `#status` | Zone de message de résultat |
| `#nuclear-box` | Bloc options NUCLEAR |
| `#credit-box` | Bloc crédit |
| `#unlock` | Bouton mode UNLOCK ALL |
| `#nuclear` | Bouton mode NUCLEAR |
| `#credit` | Bouton mode CREDIT |
| `#max-chall` | Champ nombre de challenges network |

---

## 6. Le script — popup.js

C'est le cœur de l'extension. Il est chargé en fin de `<body>` et s'exécute directement dans le popup (pas de `content script`).

### 6.1 Constantes globales

```js
const DOMAIN = "www.challenges-kids.fr";
const ORIGIN = "https://www.challenges-kids.fr/";
```

Ces deux constantes servent à écrire les cookies : le **nom de domaine** et l'**URL d'origine** exigés par l'API `chrome.cookies.set()`.

```js
const baseCookies = [
  { name: "disclaimer", value: "accepted" },
  { name: "PHPSESSID", value: "44728fag636e2c03tbdulf8prf" }
];
```

Cookies injectés **dans tous les modes** : validation du disclaimer et session PHP.

```js
const categories = {
  network: 5,
  web: 8,
  crypto: 7,
  stegano: 5,
  culture: 5
};
```

Nombre de challenges par catégorie pour le mode `UNLOCK ALL`

> **Total** : 5 + 8 + 7 + 5 + 5 = **30 cookies de challenges** + 2 cookies de base = **32 cookies** injectés.

### 6.2 Références au DOM

```js
const statusEl = document.getElementById("status");
const nuclearBox = document.getElementById("nuclear-box");
const creditBox = document.getElementById("credit-box");
```

### 6.3 `setStatus(msg, ok = true)`

Affiche un message dans la zone de statut et change sa couleur.

```js
setStatus("Injection Successful !");        // vert
setStatus("Erreur : ...", false);            // rouge
```

### 6.4 `buildCookies(list)`

Transforme une liste de cookies « simplifiés » (au format `{ name, value }`) en objets complets compatibles avec l'API Chrome.

Pour chaque cookie, la fonction ajoute :

```js
{
  url: ORIGIN,        // URL cible
  name, value,        // nom et valeur du cookie
  domain: DOMAIN,     // domaine hôte
  path: "/",          // valable sur tout le site
  secure: true,       // cookie HTTPS uniquement
  httpOnly: false     // accessible au JavaScript du site
}
```

> La propriété `url` est **obligatoire** dans `chrome.cookies.set()` : c'est elle qui détermine le domaine et le path si le champ correspondant est absent.

### 6.5 `inject(cookies)` (asynchrone)

C'est la fonction d'exécution :

1. Boucle sur chaque cookie et appelle `await chrome.cookies.set(c)` — l'extension attend que chaque cookie soit posé avant de continuer.
2. Ouvre un nouvel onglet vers `ORIGIN` avec `chrome.tabs.create({ url: ORIGIN })`.
3. Affiche `Injection Successful !`.
4. En cas d'erreur (promise rejetée), intercepte l'exception et affiche `Erreur : <message>` en rouge.

```
┌────────────┐     ┌──────────────────────┐     ┌─────────────┐
│ buildCookies │ ─► │ for cookie: set()    │ ─► │ tabs.create  │
└────────────┘     │       (await)         │     └──────┬──────┘
                   └──────────────────────┘            ▼
                                           "Injection Successful !"
```

### 6.6 Écouteurs d'événements

#### UNLOCK ALL (`#unlock`)

1. Copie `baseCookies`.
2. Parcourt `categories` → pour chaque catégorie et chaque numéro de challenge, ajoute :

```js
{ name: `/network/chall1`, value: "Hacked by FoXhack" }
```

3. Injecte le tout.

#### NUCLEAR (`#nuclear`)

1. Bascule l'affichage de `nuclearBox` et masque `creditBox`.
2. **Premier clic** → le bloc s'ouvre (l'utilisateur choisit un nombre) et la fonction s'arrête (`return`). Aucune injection n'est faite.
3. **Second clic** (le bloc était visible) → le bloc se masque et l'injection est lancée :
   - Lit `#max-chall`.
   - Valide la saisie : doit être un nombre fini ≥ 1 (sinon « Nombre invalide. » en rouge).
   - Génère les cookies `/network/chall1` → `/network/challN` avec `value = "Hacked by FoXhack"`.
   - Injecte le tout.

**Résumé du toggle** :

| État avant clic | Résultat |
|-----------------|----------|
| Bloc masqué | Le bloc s'affiche — on attend le chiffre |
| Bloc visible | Injection lancée avec la valeur saisie |

#### CREDIT (`#credit`)

Bascule l'affichage du bloc crédit et masque le bloc nuclear. Pas d'injection.

---

## 7. Les modes

### 7.1 UNLOCK ALL

Débloque **30 challenges** répartis ainsi :

| Catégorie | Nombre | Cookies créés |
|-----------|--------|---------------|
| network | 5 | `/network/chall1` → `/network/chall5` |
| web | 8 | `/web/chall1` → `/web/chall8` |
| crypto | 7 | `/crypto/chall1` → `/crypto/chall7` |
| stegano | 5 | `/stegano/chall1` → `/stegano/chall5` |
| culture | 5 | `/culture/chall1` → `/culture/chall5` |

### 7.2 NUCLEAR OPTIONS

Débloque de **1 à N** challenges `network` (valeur par défaut : `301`).

- Le champ accepte n'importe quel entier ≥ 1.
- À chaque changement de valeur, il faut re-cliquer sur « NUCLEAR OPTIONS » pour l'injecter (le clic sert à la fois d'ouvrant et de déclencheur).

**Cas particuliers**

| Saisie | Comportement |
|--------|--------------|
| `150` | 150 cookies `/network/chall1` → `/network/chall150` |
| `0` ou vide ou négatif | Message « Nombre invalide. » en rouge, aucune injection |
| valeur décimale (`12.5`) | `parseInt` la tronque → `12` |

### 7.3 CREDIT

Affiche simplement le crédit ASCII. Actions possibles ensuite : fermer le popup ou re-cliquer sur un autre mode.

---

## 8. Personnalisation

Toutes les modifications se font dans `popup.js` (pas besoin de recompiler, juste **recharger l'extension**).

### 8.1 Ajouter / modifier une catégorie

```js
const categories = {
  network: 5,
  web: 8,
  crypto: 7,
  stegano: 5,
  culture: 5,
  forensics: 6   // ← nouvelle catégorie : /forensics/chall1 → /forensics/chall6
};
```

### 8.2 Changer le nombre par défaut de NUCLEAR

```html
<input type="number" id="max-chall" min="1" value="301">
```

Modifie `value="301"` dans `popup.html`.

### 8.3 Changer la valeur des cookies de déblocage

```js
value: "Hacked by FoXhack"
```

Remplace cette valeur dans les deux boucles et affiche ce que tu veux.

### 8.4 Mettre à jour la session PHP

Si l'injection du `PHPSESSID` ne suffit plus (session expirée côté serveur) :

1. Connecte-toi normalement sur `challenges-kids.fr`.
2. Ouvre les DevTools → onglet **Application** → **Cookies**.
3. Copie la valeur actuelle de `PHPSESSID`.
4. Remplace la valeur dans `baseCookies` :

```js
{ name: "PHPSESSID", value: "<valeur récupérée>" }
```

### 8.5 Modifier les liens du footer

Dans `popup.html`, sections `.footer a` :

```html
<a href="https://guns.lol/foxhack" ...>guns.lol/foxhack</a>
<a href="https://github.com/F0X-hack" ...>github.com/F0X-hack</a>
```

---

## 9. Installation & débogage

### Installation (mode développeur)

1. Télécharge ou clone le dépôt.
2. Ouvre `chrome://extensions`.
3. Active le **mode développeur** (coin haut-droit).
4. Clique sur **Charger l'extension non empaquetée**.
5. Sélectionne le dossier `skid-extension/`.
6. Épingle l'extension dans la barre d'outils si besoin.

### Recharger après une modification

Après chaque modification d'un fichier, rends-toi sur `chrome://extensions` et clique sur l'**icône de rechargement** (⟳) de l'extension.

### Déboguer le popup

1. Clique droit sur l'icône de l'extension → **Inspecter la fenêtre contextuelle**.
2. La console affiche les éventuelles erreurs JS (ex. : cookie rejeté, permission manquante).

### Vérifier les cookies posés

1. Avec le popup ouvert, lance un mode.
2. Sur `challenges-kids.fr`, ouvre **DevTools** → **Application** → **Cookies** → `https://www.challenges-kids.fr`.
3. Tu dois y voir `disclaimer`, `PHPSESSID` et les cookies `/network/chall1` etc.

---

## 10. Dépannage

| Problème | Cause probable | Solution |
|----------|----------------|----------|
| « Erreur : ... » dans le popup | Permission cookies non active ou domaine bloqué | Vérifier que les `host_permissions` couvrent le domaine exact (avec `www`) |
| Les challenges restent verrouillés | `PHPSESSID` expiré côté serveur | Récupérer une session fraîche (voir [8.4](#84-mettre-à-jour-la-session-php)) |
| Le site redirige vers la racine sans `www` | Cookies posés sur `www.*` uniquement | Vérifier que la redirection du site garde le domaine `www` ; sinon ajouter le domaine racine à `host_permissions` |
| Rien ne se passe au clic | Extension non rechargée après modification | Recharger l'extension sur `chrome://extensions` |
| Popup blanc | Erreur JS dans `popup.js` | Inspecter le popup, corriger l'erreur |
| Le champ NUCLEAR refuse une valeur | Saisie 0, vide, négative ou non numérique | Saisir un entier ≥ 1 |

---

## 11. FAQ

**Q : L'extension crée un onglet à chaque clic ?**
R : Oui. `inject()` ouvre toujours un nouvel onglet vers le site après l'injection, quel que soit le mode.

**Q : Peut-on débloquer plus de challenges qu'il n'en existe ?**
R : Oui techniquement. L'extension pose les cookies qu'on lui demande ; le nombre n'est pas limité par le site. Les cookies excédentaires seront simplement ignorés.

**Q : Faut-il une connexion Internet pour le popup ?**
R : Le popup s'affiche hors ligne. L'injection nécessite uniquement un accès au domaine pour appliquer ses cookies (et ouvrir l'onglet).

**Q : Pourquoi de l'ASCII art ?**
R : Le thème « console / hacker » est un parti pris esthétique du projet.

**Q : L'extension fonctionne-t-elle sur les autres navigateurs (Firefox, Edge) ?**
R : L'API `chrome.cookies` et le Manifest V3 existent sur Edge (compatible Chrome). Firefox utilise des API proches mais le manifest peut nécessiter des ajustements (`browser_specific_settings`).

**Q : Pourquoi il n'y a pas de `background.js` ?**
R : Toute la logique tient dans le popup. Aucune tâche en arrière-plan n'est nécessaire : on clique, on injecte, c'est tout.

---

## 12. Sécurité — Avertissement légal

- **Ce projet est fourni à des fins strictement éducatives**, pour comprendre comment fonctionnent les cookies et l'API d'une extension Chrome.
- L'auteur **n'est pas responsable** de l'utilisation faite de cet outil.
- Débloquer des challenges peut violer les **règles du site** ou de la plateforme. Utilise-le uniquement sur des environnements autorisés.
- Le `PHPSESSID` contenu dans le code est une session capturée lors de la création de ce projet. Ne pas la partager / la réutiliser publiquement pour un usage réel.

## 13. Crédits

- **Script By** FoXhack
- **Research By** FoXhack & J202
- Links : [guns.lol/foxhack](https://guns.lol/foxhack) · [github.com/F0X-hack](https://github.com/F0X-hack)