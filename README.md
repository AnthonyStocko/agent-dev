# Agent développeur (agent-dev)

L'agent qui modifie le code des autres agents pour les améliorer. Il est commandé par l'agent chef
([agent-manager](https://github.com/AnthonyStocko/agent-manager)) : tu décris une amélioration dans
son onglet **Développeur** ou dans sa discussion, et cet agent la réalise, la teste et la met en ligne.

Il ne tourne pas sur un serveur : chaque demande est une exécution du workflow GitHub Actions
`developpeur.yml` de ce dépôt (gratuit, le dépôt est public), en 5 à 20 minutes.

## Ce qu'il fait d'une demande

1. Il clone le dépôt visé et installe ses dépendances.
2. Claude Code (ton abonnement) lit le code et fait la modification, tests compris.
3. Les vérifications du dépôt sont lancées ; si elles échouent, Claude Code corrige (deux essais).
4. Il pousse une branche `dev/<référence>` et ouvre une pull request. Si tout est vert, elle est
   fusionnée dans `main` et le déploiement habituel du dépôt suit, **sans relecture**.
5. Il envoie son compte rendu à l'agent chef, qui l'affiche dans la discussion.

## Dépôts et garde-fous

Réglés dans `depots.json` :

| Dépôt | Vérifications avant fusion | Jamais modifié | Pull request laissée à relire |
|---|---|---|---|
| agent-emploi | compilation, tests s'il y en a, démarrage du serveur | `.github/` | — |
| agent-perso | compilation, tous les tests `test_*.py` | `.github/` | — |
| agent-manager | compilation, tests s'il y en a, démarrage du serveur | `.github/` | — |
| mon-guide-agent-instagram | `verifier-principes.mjs`, puis la CI du dépôt | `.github/`, `CLAUDE.md`, `PRINCIPES.md`, `verifier-principes.mjs` | migrations Supabase |
| Monguide | `npm test`, puis les workflows du dépôt | `.github/` | migrations et rollbacks Supabase |

- Un travail qui touche un chemin « jamais modifié » est rejeté en entier : rien n'est poussé.
- Vérifications en échec : la pull request reste ouverte, préfixée « [tests en échec] », rien n'est fusionné.
- agent-emploi n'a pas encore de tests : « vert » y veut seulement dire que le code compile et que
  le serveur démarre. L'agent a pour consigne d'ajouter des tests à chaque modification.
- Cet agent ne se modifie pas lui-même : agent-dev n'est pas dans la liste.
- Une seule demande à la fois par dépôt ; les suivantes attendent.
- Claude Code ne reçoit ni le jeton GitHub ni `HUB_TOKEN` : il ne peut ni pousser ni fusionner lui-même.

Pour ajouter un dépôt : une entrée dans `depots.json`, la même clé dans la liste `options` de
`.github/workflows/developpeur.yml`, et l'accès du jeton GitHub à ce dépôt.

## Mise en place (une fois)

1. **Jeton GitHub** : GitHub > Settings > Developer settings > Fine-grained tokens > Generate new
   token. Repository access : agent-dev et les cinq dépôts du tableau. Permissions : **Contents**
   (Read and write), **Pull requests** (Read and write), **Actions** (Read and write).
2. **Secrets de ce dépôt** (Settings > Secrets and variables > Actions) :
   - `DEV_GITHUB_TOKEN` : le jeton de l'étape 1 ;
   - `CLAUDE_CODE_OAUTH_TOKEN` : le jeton d'abonnement de tes autres agents (`claude setup-token`) ;
   - `AGENT_URL` : l'adresse de l'agent chef (`https://….onrender.com`) ;
   - `HUB_TOKEN` : la même valeur que dans l'agent chef.
3. **Agent chef** (Render, service agent-manager) : variable `DEV_GITHUB_TOKEN` avec le même jeton.

Facultatif : variable de dépôt `CLAUDE_CLI_MODEL` pour choisir le modèle de Claude Code.

## Lancer une demande sans l'agent chef

Onglet Actions > Agent développeur > Run workflow : choisis le dépôt et écris la demande. Le compte
rendu est dans le résumé de l'exécution (et envoyé à l'agent chef si `AGENT_URL` est défini).

## Fichiers

- `travail.py` : le déroulé d'une demande — `depots.json` : dépôts et garde-fous
- `consigne.md` : la consigne donnée à Claude Code — `demarrage.py` : vérifie qu'un serveur démarre
- `test_travail.py` : tests, avec un faux Claude Code, un faux `gh` et un dépôt git local
  (`python -m unittest discover -p 'test_*.py'`)
