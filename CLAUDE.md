# CLAUDE.md — Référence technique pour LLM

> Ce fichier est destiné à être lu par un LLM (Claude, GPT, etc.) pour reprendre le projet sans contexte additionnel. Il décrit l'architecture, les conventions, les règles métier et les contraintes techniques.

---

## 1. Architecture générale

```
iPhone (PWA Safari)
  └── pokemon_tracker_mobile.html (fichier unique, ~1400 lignes)
        ├── Lit/écrit data.json via GitHub API
        └── Estime les prix via 3 Cloudflare Workers (eBay Browse API)

PC Windows (script hebdomadaire)
  └── fetch_cotes.py (Python + Playwright)
        ├── Scrape ebay.fr avec vrai navigateur Chromium
        └── Push data.json sur GitHub via API

GitHub (stockage central)
  └── data.json (source de vérité unique)
        ├── Lu au démarrage par l'app iPhone
        └── Écrit par l'app iPhone ET le script PC
```

---

## 2. Structure de données

### `data.json` (fichier central)

```json
{
  "articles": [ /* tableau d'articles */ ],
  "cat": [ /* catalogue produits pour autocomplétion */ ],
  "ebayDate": "2026-04-01T23:36:51.988454",
  "at": "2026-04-01T21:06:32.616Z"
}
```

### Structure d'un article (objet normalisé)

```json
{
  "id": "1775077568798",        // String(Date.now() + index)
  "type": "Pokemon",            // "Pokemon" | "Topps" | "Autre"
  "item": "Coffret Dresseur...", // Nom du produit
  "cout": 55.99,                 // Prix d'achat unitaire (€)
  "stk": "Tiroir",              // Lieu de stockage physique
  "site": "Amazon",             // Site d'achat
  "dt": "07/02/2025",           // Date achat (JJ/MM/AA ou JJ/MM/AAAA)
  "le": null,                   // Cote manuelle (€) — rarement utilisé
  "ebay": 95.0,                 // Dernière cote eBay connue (legacy)
  "pv": null,                   // Prix de vente réalisé (si vendu)
  "statut": "stock",            // "stock" | "avendre" | "vendu"
  "cotes": [                    // Historique des estimations
    {
      "date": "22/02/2025",     // JJ/MM/AAAA ou JJ/MM/AA
      "prix": 120.0,
      "source": "import"        // "import" | "ebay" | "manuel" | "cardmarket"
    }
  ],
  "ebayQ": "Coffret Dresseur Elite Évolution Prismatique" // Terme recherche eBay custom
}
```

### Fonction `normalise(a)` (JavaScript)
Toujours passer les objets bruts par `normalise()` avant de les stocker dans `DB`. Elle garantit que tous les champs existent avec les bons types. Se trouve ligne ~600 du HTML.

### Fonction `currentEstim(a)` (JavaScript)
Retourne l'estimation courante d'un article :
1. Dernière entrée de `a.cotes[]` (priorité absolue)
2. Sinon `a.ebay`
3. Sinon `a.le`
4. Sinon `null`

---

## 3. GitHub API — Conventions

### Lecture
```javascript
GET https://api.github.com/repos/{REPO}/contents/{FILE}
Authorization: token {GH_TOKEN}
// → response.content est du base64, décodé avec atob()
// → response.sha doit être conservé pour l'écriture
```

### Écriture
```javascript
PUT https://api.github.com/repos/{REPO}/contents/{FILE}
// body: { message, content (base64), sha (obligatoire si fichier existe) }
```

### Variable SHA
`ghSha` est une variable globale JavaScript mise à jour à chaque lecture/écriture. **Ne jamais écrire sans le SHA courant** — GitHub retourne 422 si le SHA est périmé (conflit de concurrence).

### Token GitHub
- Variable : `GH_TOKEN` (en dur dans le JS ou via `localStorage.getItem("gh_token")`)
- Portée nécessaire : `repo` (classic token) ou Contents read/write (fine-grained)
- **Problème connu** : GitHub révoque automatiquement les tokens classic détectés dans des fichiers publics. Solution : utiliser fine-grained tokens, ou ne jamais commit un token valide.
- Repo : `alkaatrazz218-cpu/Pokemon-tracker` (public)

---

## 4. Import Excel — Format attendu

### Nouveau format (actuel)
```
Type | Stockage | Item | ebayQ | Px achat U | Date achat | Site achat | Statut | Prix vente | [dates cotes...]
```

### Ancien format (legacy, encore supporté)
```
Stockage | Item | Px achat U | Date achat | Site achat | [dates cotes...]
```

### Détection automatique
Le parser détecte le format via `header[0]` :
- `"type"` → nouveau format
- autre → ancien format

### Colonnes d'estimation (dates)
Les colonnes de cotes sont détectées par leur header :
- Format `JJ/MM/AAAA` → direct
- Format `YYYY-MM-DD` (datetime Python/JS) → converti en `JJ/MM/AA`
- Format nombre (serial Excel) → converti via epoch

**Important** : la colonne `ebayQ` est détectée en vérifiant si `header[3].toLowerCase() === 'ebayq'`.

---

## 5. Estimation eBay

### Depuis l'iPhone (Cloudflare Workers)
- 3 Workers en rotation : `ebay-proxy.alkaatrazz218.workers.dev` (1, 2, 3)
- Workers scrape `ebay.fr` HTML avec paramètres `_udlo` (prix min) et `_udhi=1200`
- Extrait les prix via regex : `/([\d]+[,][\d]{2})\s*EUR/g`
- **Limitation majeure** : eBay rate-limit les IPs Cloudflare après ~15 requêtes → résultats très limités
- Délai entre requêtes : 12 secondes

### Depuis le PC (fetch_cotes.py)
- Playwright + Chromium headless
- IP personnelle → pas bloquée par eBay
- Taux de réussite observé : **97% (140/145)**
- Délai entre requêtes : 3 secondes
- Sauvegarde intermédiaire toutes les 10 articles
- Extraction prix : regex `/([\d]+[,][\d]{2})\s*EUR/`
- Filtre statistique IQR sur les prix trouvés
- Prix minimum : `max(5, round(cout * 0.7))` (70% du prix d'achat)

### Terme de recherche
1. Si `ebayQ` renseigné → utilisé tel quel
2. Sinon → `buildQuery(nom)` génère automatiquement (préfixe + série extraite)

### Médiane filtrée (IQR)
```python
s = sorted(prices)
q1, q3 = s[n*0.25], s[n*0.75]
iqr = q3 - q1
filtered = [p for p in s if q1 - 1.5*iqr <= p <= q3 + 1.5*iqr]
return median(filtered)
```

---

## 6. Cloudflare Workers

### Code Workers (identique sur les 3)
- Scrape `ebay.fr` avec headers browser-like
- Retourne `{prices[], total, query, htmlLen}`
- Si `htmlLen < 50000` → rate limit silencieux détecté

### eBay credentials (Browse API — non utilisée activement)
```
clientId:     SimonMal-pokemont-PRD-bbf324dd4-afabc136
clientSecret: PRD-bf324dd49915-f174-447a-b458-8a25
```

---

## 7. Conventions de code (JavaScript)

- **Pas de framework** — vanilla JS pur
- **Variable globale principale** : `DB` (tableau d'articles normalisés)
- **Rendu** : fonctions `renderStock()`, `updateMetrics()`, `renderSignals()`, `renderTops()` — appelées via `updateAll()`
- **Navigation** : `goTab('dash'|'saisie'|'stock'|'ebay', btn)` — change `.page.active`
- **Helper** : `el(id)` = `document.getElementById(id)` ; `f(v)` = formateur €
- **Toast** : `showToast(msg, type='')` — types : `''` (vert) / `'error'` (rouge) / `'info'` (bleu)
- **Dates** : format stocké `JJ/MM/AA` ou `JJ/MM/AAAA` (français) — `parseDt()` pour comparer
- **Fichier unique** : TOUT est dans `pokemon_tracker_mobile.html` — CSS, JS, HTML en un seul fichier

---

## 8. Règles métier

### Statuts produits
| Statut | Description |
|--------|-------------|
| `stock` | En possession, pas encore à vendre |
| `avendre` | Mis en vente (stockage = "A vendre") |
| `vendu` | Vendu — exclu du calcul de la plus-value latente |

### Calculs dashboard
- **Investi** = somme des `cout` des articles `statut !== 'vendu'`
- **Estimation** = somme des `currentEstim()` des articles non vendus (articles sans cote = 0)
- **Plus-value latente** = Estimation - Investi
- **Bénéfice réalisé** = somme des `(pv - cout)` des articles `statut === 'vendu'`

### Signaux de vente
- `>= +100%` → "Idéal pour vendre"
- `>= +50%` → "Bonne plus-value"
- `< -10%` → "À surveiller"

### Types d'investissement
- `Pokemon` — produits scellés Pokémon TCG FR
- `Topps` — cartes Topps (Marvel, Disney, etc.)
- `Autre` — tout autre collectible

---

## 9. Export Excel

Format de sortie :
```
Type | Stockage | Item | ebayQ | Px achat U | Date achat | Site achat | Statut | Prix vente | Dernière cote | [dates cotes chronologiques...]
```

Généré via SheetJS (`XLSX.utils.aoa_to_sheet`). Déclenché par le bouton "Exporter tout le stock" dans l'onglet Saisie.

---

## 10. Ce qu'il ne faut jamais modifier sans validation

1. **La fonction `normalise(a)`** — tout l'écosystème dépend de sa structure de sortie
2. **La structure de `data.json`** — le script Python et l'app iPhone doivent rester compatibles
3. **Le SHA tracking GitHub** — supprimer le tracking SHA causera des erreurs 422 en écriture
4. **La fonction `currentEstim(a)`** — utilisée partout pour calculer les métriques
5. **Le format des dates dans `cotes[]`** — le tri et les comparaisons dépendent de `parseDtFr()`
6. **Les noms de champs de l'objet article** — `id`, `type`, `item`, `cout`, `stk`, `site`, `dt`, `le`, `ebay`, `pv`, `statut`, `cotes`, `ebayQ`

---

## 11. Problèmes connus et limitations

| Problème | Cause | Statut |
|----------|-------|--------|
| Workers eBay retournent peu de résultats | IP Cloudflare rate-limitée par eBay | Contourné via script PC |
| Token GitHub révoqué si dans fichier public | GitHub secret scanning | Token en localStorage / fine-grained |
| Cardmarket non implémenté | Protection Cloudflare avancée | Playwright PC à tester |
| Pas de sync temps réel multi-device | Architecture sans backend | Acceptable pour usage solo |
