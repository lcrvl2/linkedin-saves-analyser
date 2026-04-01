# LinkedIn Saves Analyzer

Extrait tous tes posts sauvegardés sur LinkedIn et génère un rapport d'insights actionnable — catégories, fraîcheur, actionnabilité, top auteurs et priorités d'implémentation.

Fonctionne avec ton abonnement Claude existant. Aucune clé API requise.

> [English version](README.en.md)

## Ce que ça fait

1. **Scrape** ta page "Posts enregistrés" LinkedIn via Chrome en remote debugging
2. **Découvre les catégories** depuis ton contenu réel (pas de taxonomie fixe — s'adapte à chaque utilisateur)
3. **Catégorise** chaque post (catégorie, fraîcheur, actionnabilité, insight clé) via des agents Claude en parallèle
4. **Génère** un rapport Markdown avec breakdown par catégorie, top auteurs, matrice d'implémentation et signal de positionnement

## Exemple de rapport

```
# Analyse LinkedIn Saves — 2026-03-25

| Métrique             | Valeur |
|----------------------|--------|
| Posts analysés       | 487    |
| Auteurs uniques      | 203    |
| Posts à forte valeur | 89     |
| Posts obsolètes      | 34     |
```

Le rapport complet inclut : insights par catégorie, posts obsolètes à archiver, classement des auteurs, priorités d'implémentation (maintenant / 30 jours / preuves), et un signal de positionnement.

---

## Deux façons de l'utiliser

### Option A : Claude Code (CLI / IDE)

Pour les devs à l'aise avec le terminal ou les extensions IDE.

### Option B : Claude Cowork (interface web)

Pour les utilisateurs non-techniques qui préfèrent une interface visuelle. Mêmes capacités, pas de terminal.

---

## Installation

### 1. Chrome avec Remote Debugging

Lance Chrome avec le port de debug distant ouvert, puis **connecte-toi à LinkedIn** dans cette fenêtre.

**Windows :**
```bash
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222
```

**macOS :**
```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
```

**Linux :**
```bash
google-chrome --remote-debugging-port=9222
```

### 2a. Installation pour Claude Code

Installe [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI ou extension IDE).

Ajoute le serveur MCP Chrome DevTools :

```bash
claude mcp add chrome-devtools -- npx @anthropic-ai/chrome-devtools-mcp@latest
```

Copie le fichier skill dans ton répertoire de skills Claude Code :

```bash
mkdir -p ~/.claude/skills/linkedin-saves-analyzer
cp SKILL.md ~/.claude/skills/linkedin-saves-analyzer/SKILL.md
```

**Node.js 18+** est aussi requis (pour l'étape d'agrégation).

Ensuite, lance :

```
/linkedin-saves-analyzer
```

Ou dis : *"analyse mes saves LinkedIn"*

### 2b. Installation pour Claude Cowork

1. Ouvre [Claude Cowork](https://cowork.claude.ai)
2. Crée un nouveau projet (ou ouvre un projet existant)
3. Va dans **Integrations** et ajoute le serveur MCP **Chrome DevTools**
4. Upload `SKILL.md` dans les fichiers du projet (glisser-déposer ou bouton d'upload)
5. Assure-toi que Chrome tourne avec `--remote-debugging-port=9222` et que tu es connecté à LinkedIn

Ensuite tape :

```
Analyse mes posts LinkedIn sauvegardés en utilisant le skill linkedin-saves-analyzer
```

Claude suit le workflow du SKILL.md automatiquement : scrape tes saves, propose des catégories basées sur ton contenu, demande confirmation, puis lance l'analyse complète et génère le rapport.

---

## Comment ça fonctionne

```
Page LinkedIn Saves (Chrome)
    |  Chrome DevTools MCP — scroll + extraction JS
    v
posts.json (brut : auteur, contenu, post_url)
    |  Claude lit un échantillon de ~50 posts
    v
Catégories dynamiques (8-12, proposées + confirmées par l'utilisateur)
    |  Découpage en batches de 50
    v
batches/batch_01.json ... batch_NN.json
    |  Agents Claude en parallèle catégorisent chaque batch
    v
analyzed/batch_XX_YY.json
    |  Agrégation Node.js
    v
all_analyzed.json (enrichi : catégorie, fraîcheur, actionnabilité, insight_clé)
    |  Génération du rapport
    v
rapport-linkedin-saves-{YYYY-MM-DD}.md
```

### Catégories dynamiques

Les catégories ne sont **pas en dur**. Le skill :
1. Échantillonne ~50 de tes posts sauvegardés
2. Identifie les thèmes et sujets récurrents
3. Propose 8-12 catégories spécifiques (ex. "outbound-automation", "product-led-growth", "ai-for-content")
4. Te demande de confirmer ou ajuster avant de lancer l'analyse

Ça veut dire que l'outil fonctionne pour n'importe qui — marketeur, dev, recruteur — sans configuration.

### Dimensions de scoring

- **Fraîcheur** : `evergreen` / `current` / `outdated`
- **Actionnabilité** : `high` / `medium` / `low`

### Paramètres

| Paramètre   | Défaut  | Description                              |
|-------------|---------|------------------------------------------|
| `language`  | `fr`    | Langue du rapport (`fr` ou `en`)         |
| `max_posts` | tout    | Nombre max de posts à extraire           |

---

## Structure des fichiers

```
linkedin-saves-analyser/
  SKILL.md                        # Définition du skill Claude (le cerveau)
  README.md                       # Ce fichier (FR)
  README.en.md                    # Version anglaise
  .gitignore                      # Exclut les données générées

  Généré à chaque run (gitignored) :
  posts.json                      # Posts extraits bruts
  batches/                        # Batches découpés (50 posts chacun)
  analyzed/                       # Résultats catégorisés par batch
  all_analyzed.json               # Analyse agrégée
  rapport-linkedin-saves-*.md     # Rapport final
```

---

## Dépannage

| Problème | Solution |
|----------|----------|
| Chrome non joignable | Lance Chrome avec `--remote-debugging-port=9222` |
| Pas connecté à LinkedIn | Connecte-toi manuellement dans la fenêtre Chrome, puis relance |
| 0 posts extraits | LinkedIn a changé ses classes CSS — Claude prend un screenshot et adapte les sélecteurs automatiquement |
| Scroll infini bloqué | Claude réessaie avec des stratégies de scroll alternatives |

---

## Confidentialité

- Les posts sont traités localement dans ta session Claude
- Aucune donnée n'est envoyée à des services externes au-delà de l'API Claude
- Les fichiers générés restent sur ta machine

## Licence

MIT
