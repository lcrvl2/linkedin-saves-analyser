# LinkedIn Saves Analyzer

Analyse tous tes posts enregistrés sur LinkedIn et génère un rapport d'insights détaillé : catégories, freshness, actionnabilité, top auteurs, matrice d'implémentation.

Fonctionne avec ta souscription Claude existante — aucune clé API requise.

---

## Prérequis

### 1. Claude Code
Installe Claude Code si ce n'est pas déjà fait :
```bash
npm install -g @anthropic-ai/claude-code
```

### 2. Chrome DevTools MCP
Ce skill pilote Chrome directement via le protocole DevTools. Pour l'activer :

```bash
claude mcp add chrome-devtools-mcp
```

> Chrome s'ouvre automatiquement quand le skill se lance. Pas besoin de configuration supplémentaire.

### 3. Node.js
Requis pour l'agrégation des résultats (étape de chunking).

Vérifie que tu l'as :
```bash
node --version
```

Si absent : [nodejs.org/fr/download](https://nodejs.org/fr/download)

---

## Utilisation

1. Ouvre Claude Code dans ce dossier :
```bash
cd linkedin-saves-analyzer
claude
```

2. Lance le skill :
```
/linkedin-saves-analyzer
```

3. Connecte-toi à LinkedIn dans la fenêtre Chrome qui s'ouvre (si ce n'est pas déjà fait)

4. Laisse tourner — pour 500+ posts, l'analyse prend 5-10 minutes via agents parallèles

5. Le rapport est généré dans ce dossier : `rapport-linkedin-saves-YYYY-MM-DD.md`

---

## Ce que tu obtiens

- **Catégorisation post par post** 
- **Freshness** : evergreen / current / outdated, tu sais ce qui est encore pertinent en 2026
- **Actionnabilité** : high / medium / low — priorise ce que tu dois implémenter
- **Top insights par catégorie** les pépites extraites de chaque thème
- **Matrice d'implémentation** : quoi faire maintenant vs dans 30 jours
- **Signal de positionnement** : ce que tes saves révèlent sur ton profil

---

## Structure du dossier

```
linkedin-saves-analyzer/
├── README.md              ← ce fichier
├── posts.json             ← posts extraits (généré automatiquement)
├── batches/               ← batches de 50 posts (généré automatiquement)
├── analyzed/              ← résultats par batch (généré automatiquement)
└── rapport-*.md           ← rapport final
```

Les dossiers `batches/` et `analyzed/` sont créés automatiquement par le skill.
