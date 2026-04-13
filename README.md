# 📦 Invest Tracker — Pokémon TCG & Collectibles

Tracker d'investissement personnel pour produits scellés Pokémon TCG (coffrets, displays, pokéboxes, mini tins…) et autres collectibles (Topps, etc.). Application mobile-first installable en PWA sur iPhone, avec synchronisation GitHub et estimation automatique des prix eBay via script Python PC.

---

## 🎯 Objectif

Suivre en temps réel la valeur de son stock de produits scellés Pokémon :
- Prix d'achat vs estimation eBay actuelle
- Plus-value latente et bénéfice réalisé
- Historique des cotes dans le temps
- Signaux de vente automatiques

---

## 🌐 Liens clés

| Ressource | URL |
|-----------|-----|
| **App live (PWA)** | https://alkaatrazz218-cpu.github.io/Pokemon-tracker/pokemon_tracker_mobile.html |
| **Repo GitHub** | https://github.com/alkaatrazz218-cpu/Pokemon-tracker |
| **Données (data.json)** | https://github.com/alkaatrazz218-cpu/Pokemon-tracker/blob/main/data.json |
| **eBay Proxy Worker 1** | https://ebay-proxy.alkaatrazz218.workers.dev |
| **eBay Proxy Worker 2** | https://ebay-proxy-2.alkaatrazz218.workers.dev |
| **eBay Proxy Worker 3** | https://ebay-proxy-3.alkaatrazz218.workers.dev |

---

## 🛠 Stack technique

| Composant | Technologie |
|-----------|-------------|
| **Frontend** | HTML/CSS/JS vanilla — fichier unique `pokemon_tracker_mobile.html` |
| **Hébergement** | GitHub Pages (branche `main`) |
| **Persistance données** | GitHub API (fichier `data.json` dans le repo) |
| **Estimation eBay (iPhone)** | 3 Cloudflare Workers en rotation (Browse API eBay) |
| **Estimation eBay (PC)** | Python 3 + Playwright (scraping navigateur headless) |
| **Librairie Excel** | SheetJS / XLSX (CDN) |
| **Fonts** | Space Mono + Syne (Google Fonts) |
| **PWA** | meta `apple-mobile-web-app-capable` — installable iOS |

---

## 📲 Installation (iPhone)

1. Ouvrir Safari sur : `https://alkaatrazz218-cpu.github.io/Pokemon-tracker/pokemon_tracker_mobile.html`
2. Appuyer sur **Partager** → **Sur l'écran d'accueil**
3. Au premier lancement, entrer le token GitHub si demandé
4. Importer son fichier Excel via le bouton 📂

---

## 🖥 Installation script PC (Windows)

### Prérequis
- Python 3.9+ : https://www.python.org/downloads/
- Fichiers dans `C:\invest-tracker\` : `fetch_cotes.py`, `requirements.txt`

### Installation (une seule fois)
```bash
cd C:\invest-tracker
pip install -r requirements.txt
playwright install chromium
```

### Utilisation hebdomadaire
```bash
python fetch_cotes.py
# Répondre : eBay FR ? o | Cardmarket ? n
# Durée estimée : ~7 minutes pour 145 articles
# Taux de réussite observé : 97% (140/145)
```

---

## ✨ Fonctionnalités

### Dashboard
- Plus-value latente totale (stock)
- Investi / Estimation / Vendus / À vendre
- Articles sans estimation
- Date dernière MAJ eBay
- Signaux de vente automatiques (>+50%, >+100%)
- Top performers
- Filtre global par type (Pokémon / Topps / Autre)

### Saisie
- Autocomplétion catalogue produits (~130 références)
- Ajout multi-quantité
- Stockage / site d'achat / date
- Export Excel complet (toutes cotes + ebayQ)

### Stock
- Recherche texte fulltext
- Filtres : statut (Tous/Stock/Vendus/À vendre/Sans cote), Stockage, Site achat
- Tri : Date / Coût / Estimation
- Fiche article avec onglets Infos + Historique cotes
- Champ `ebayQ` personnalisable par article

### Cotes eBay (onglet Cotes)
- Estimation via Cloudflare Workers (rotation 3 proxies)
- Pause / Reprise / Sauvegarde / Ignorer / Réinitialiser
- Affichage raison d'échec par article
- Sauvegarde automatique toutes les 10 articles

---

## 📁 Structure du repo

```
Pokemon-tracker/
├── pokemon_tracker_mobile.html   # App complète (fichier unique)
├── data.json                     # Données (articles, cotes, cat)
├── README.md
├── CLAUDE.md
└── CONTEXT.md
```

---

## 📊 Format données

Voir `CLAUDE.md` pour la structure complète de `data.json`.

---

## 🔑 Token GitHub

Le token GitHub (`ghp_...`) est nécessaire pour lire/écrire `data.json`.  
Il est stocké dans le `localStorage` de l'iPhone et dans `fetch_cotes.py` sur le PC.  
Ne jamais commiter un token valide dans un fichier public — GitHub le révoque automatiquement.
