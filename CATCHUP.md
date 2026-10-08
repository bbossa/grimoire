# Rattrapage upstream — fork `bbossa/grimoire`

Ce dépôt est un fork de [`goniszewski/grimoire`](https://github.com/goniszewski/grimoire).
Il sert à suivre les versions upstream et à corriger les CVE détectées par Trivy avant de
redéployer sur le NAS. Objectif à chaque rattrapage : **Trivy vide** (HIGH/CRITICAL,
`--ignore-unfixed`) sur le filesystem **et** sur l'image.

Le workflow `upstream-sync.yml` ouvre chaque lundi une PR `sync/upstream-main` quand
upstream/main a de nouveaux commits. Cette PR est **une simple notification, à ne pas
fusionner**. Le rattrapage se fait à la main, sur un **tag de version** upstream, en suivant
la procédure ci-dessous.

Historique : v1.2.0 (PR #1), v1.3.0 (PR #2).

---

## 0. Prérequis (poste de travail)

- `bun`, `trivy`, `gh` installés via mise : `mise use -g bun@latest aqua:aquasecurity/trivy@latest`.
- **Node 22**, comme en CI. Sous Node ≥ 25, le `localStorage` natif entre en conflit avec
  jsdom et une centaine de tests Vitest échouent. Créer à la racine un fichier
  `mise.local.toml`, à ne pas commiter :
  ```toml
  [tools]
  node = "22"
  ```
- Le hook husky (pre-commit) lance lint, types et tests : il a besoin de `bun` dans le PATH.
  Commiter avec `mise exec -- git commit …`.
- Chromium de Playwright pour les tests e2e : `mise exec -- npx playwright install chromium`.

## 1. Remote upstream et version cible

```bash
git remote add upstream https://github.com/goniszewski/grimoire.git   # une seule fois
git fetch upstream --tags
git tag --sort=-v:refname | head -3          # ex. v1.4.0
git log --oneline main..vX.Y.0 | wc -l       # retard
git diff main vX.Y.0 --stat -- Dockerfile package.json daemon/package.json
```

## 2. Fusion du tag upstream

```bash
git checkout -b sync/upstream-vX.Y main
git merge vX.Y.0 --no-edit
```

- Conflit sur `bun.lock` : reprendre la version upstream, puis réinstaller pour réappliquer
  les overrides du fork :
  ```bash
  git checkout --theirs bun.lock && bun install
  ```
- Vérifier que les **ajouts du fork** ont survécu :
  - `Dockerfile`, stage runtime : `apt-get update && apt-get upgrade -y && apt-get install -y` ;
  - `package.json` → `overrides` ;
  - `daemon/package.json` → `overrides`.
- Supprimer un override dès que upstream installe une version égale ou supérieure, ou que
  le paquet a disparu de l'arbre de dépendances.
- Vérifier les lockfiles :
  ```bash
  bun install --frozen-lockfile && (cd daemon && bun install --frozen-lockfile)
  ```

## 3. Scan Trivy des dépendances et corrections

```bash
trivy fs --severity HIGH,CRITICAL --ignore-unfixed .
```

Pour chaque vulnérabilité trouvée :
- **dépendance directe** : monter sa version dans le `package.json` concerné ;
- **dépendance transitive** : ajouter un override, version exacte dans `daemon/package.json`
  ou plage `^` à la racine, comme les overrides déjà présents.

Puis régénérer les lockfiles :

```bash
(cd daemon && bun install) && bun install
# package-lock.json : utiliser npm ≥ 11 (Node 26). Le npm 10 de Node 22 supprime
# les champs "libc" et pollue le diff.
PATH=~/.local/share/mise/installs/node/26.8.2/bin:$PATH npm install --package-lock-only --ignore-scripts
git diff package-lock.json                   # ne doit montrer que les paquets corrigés
trivy fs --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 .   # → 0
```

Faire un commit dédié `fix: remediate dependency CVEs …` qui liste chaque paquet et sa
CVE : ça facilite le rattrapage suivant.

## 4. Tests

```bash
mise exec -- npm run check        # lint + types + tests frontend (aussi lancés par le hook)
mise exec -- npm run test:daemon
mise exec -- npm run test:e2e
```

## 5. Scan Trivy de l'image

Ne pas utiliser le groupe `docker` : `sudo docker` suffit, puis on scanne une archive
exportée de l'image.

```bash
sudo docker build -t grimoire:candidate .
sudo docker save grimoire:candidate -o /tmp/grimoire.tar && sudo chown "$USER" /tmp/grimoire.tar
trivy image --input /tmp/grimoire.tar --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1
rm /tmp/grimoire.tar
```

## 6. PR et fusion

```bash
git push -u origin sync/upstream-vX.Y
gh pr create --repo bbossa/grimoire --base main --head sync/upstream-vX.Y
gh pr checks --repo bbossa/grimoire --watch
gh pr merge --repo bbossa/grimoire --merge     # merge commit, PAS de squash
```

- `main` est protégée (ruleset `protect-main`) : PR obligatoire, et les contrôles « Trivy
  filesystem scan », « Build image and Trivy scan » et « Lint, types, unit tests, docs,
  build » doivent être verts.
- Il faut un **merge commit** pour garder l'historique upstream dans `main`. Sinon, le
  rattrapage suivant ré-affronte les mêmes conflits.
- Noter le SHA du merge commit : c'est lui qu'on déploie.

## 7. Déploiement sur le NAS

Il n'y a **pas de clone git sur le NAS**. `/volume2/docker/grimoire/compose.yaml` construit
l'image directement depuis GitHub, à un commit pinné :

```yaml
build: https://github.com/bbossa/grimoire.git#<sha>
```

Dans `/volume2/docker/grimoire` :

```bash
cp compose.yaml compose.yaml.bak-$(date +%F)
sed -i 's/<ancien-sha>/<nouveau-sha>/' compose.yaml && grep 'build:' compose.yaml

sudo docker compose build --no-cache grimoire     # l'ancien conteneur tourne encore

sudo docker compose stop grimoire                 # SQLite : copier à l'arrêt
sudo cp -a data data.bak-$(date +%F)
sudo docker compose up -d grimoire

sleep 10; sudo docker compose ps grimoire
sudo docker exec grimoire curl -f http://localhost:3210/health
sudo docker compose logs --tail 30 grimoire
```

Les migrations SQLite s'appliquent au premier démarrage. Pour revenir en arrière : remettre
le `compose.yaml.bak-*` et le `data.bak-*`, puis `docker compose up -d --build grimoire`.

Une fois la nouvelle version validée, après quelques jours : supprimer `data.bak-*` et
`compose.yaml.bak-*`, puis lancer `sudo docker image prune`.
