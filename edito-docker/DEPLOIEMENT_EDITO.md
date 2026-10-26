# Déploiement de Carpediem sur EDITO (Process Playground)

Mode opératoire du premier déploiement de test. Les étapes 1 à 4 sont faites par Benoit
(compte Docker Hub d'EGM, compte EDITO du client). Mise à jour : 26 octobre 2026 (version 2.1.0 publiée dans le catalogue officiel « Metrics and indicators »).

Registre provisoire : `docker.io/easyglobalmarket/carpediem` (compte EGM), en attendant le
choix du client (prérequis #5 du plan).

## 1. Construire et pousser l'image

Depuis la racine du dépôt, sur la branche `v2-engineering` (commit `0ca0789` ou suivant) :

```bash
docker login                       # compte Docker Hub EGM
docker buildx build --platform linux/amd64 \
  -f edito-docker/Dockerfile \
  --build-arg CARPEDIEM_VERSION=2.1.0 \
  --build-arg CARPEDIEM_COMMIT=$(git rev-parse --short HEAD) \
  -t easyglobalmarket/carpediem:2.1.0 \
  --push .
```

* Depuis la 2.1.0, l'étiquette est la version du processus seule (`2.1.0`). Le commit du code
  reste inscrit dans le `run.json` de chaque run (arguments `CARPEDIEM_VERSION` et
  `CARPEDIEM_COMMIT`). Une étiquette publiée
  n'est jamais réécrite : toute modification de l'image (code ou `edito-docker/`) donne une
  nouvelle version.
* `linux/amd64` est obligatoire (nœuds EDITO). Sur un Mac Apple Silicon, la construction
  passe par l'émulation et prend plus de temps.
* Vérifier sur Docker Hub que le dépôt est **public** (exigence EDITO).
* Taille attendue : environ 0,75 Go compressé.

## 2. Publier l'étude Mer du Nord (archive publique)

Depuis la version 2.1.0, l'image ne contient plus d'étude, et l'étude par défaut du processus
est l'archive publique de l'étude Mer du Nord, lisible par tous les utilisateurs d'EDITO.

1. Produire l'archive : `migration_v1_v2/convertir_mer_du_nord.sh` écrit
   `Donnees Mer du Nord/V2/study.tar.gz` (environ 7,4 Mo : `config/` et `inputs/`).
2. La déposer dans le bucket du compte EDITO, sous `carpediem/mer_du_nord/study.tar.gz`, par
   l'explorateur de fichiers du Datalab (« Mes fichiers »).
3. La rendre publique dans l'explorateur de fichiers (icône en forme d'œil, visibilité
   publique).
4. Vérifier, sans identifiants, depuis n'importe quel poste :
   ```bash
   curl -sI https://minio.dive.edito.eu/oidc-antoningimard/carpediem/mer_du_nord/study.tar.gz | head -1
   ```
   Réponse attendue : `200` (vérifié le 22 octobre 2026).

L'archive ne contient ni identifiant de base de données ni le fichier de paramétrage
d'origine. La rendre publique revient à rediffuser les données Mer du Nord : décision du
22 octobre 2026, à faire confirmer par écrit par le client (prérequis #26).

L'ancien dossier `carpediem/mer_du_nord/study/` peut être conservé (lecture avec les
identifiants du compte, `CARPEDIEM_STUDY=s3://${AWS_BUCKET_NAME}/carpediem/mer_du_nord/study`).

## 3. Déclarer le processus (add-your-process)

Process Catalog > Contribution > add-your-process, puis remplir :

| Section du formulaire | Champ | Valeur |
|---|---|---|
| 1 | Friendly name | `CumulativeImpactTool-CarpeDiem` |
| 2. Configuration | Container image is already built | `true` |
| | Image tag | `docker.io/easyglobalmarket/carpediem:2.1.0` |
| 3. Metadata | Process name | `carpediem` |
| | Process version | `2.1.0` (toujours supérieure à la version déjà déclarée) |
| | Process description | texte de la fiche client (wiki Redmine `Fiche_Application`) |
| | Icon URL | URL provisoire de la fiche client (à remplacer) |
| | Homepage URL | `https://github.com/FredQuemmerais/carpediem/tree/carpediem_v2` |
| | Keywords | cumulative, impacts, marine, spatial, planning, biodiversity, activities, habitats, pressures, uncertainty |
| 4. Git | | rien (l'image est déjà construite) |
| 5. Copernicus Marine | | non coché |
| 6. Other environment variables | | voir ci-dessous |
| 7. Resources for add-your-process | | valeurs par défaut |
| 8. Resources for your process | CPU | minimum 1000m ; maximum 8000m si le champ existe |
| | Mémoire (si le champ existe) | minimum 2 Gi ; maximum 8 Gi |

Mots-clés (section 3) : un mot-clé par champ (sur la version 2.0.4, seul `cumulative` a été conservé) : cumulative, impacts, marine, spatial, planning, biodiversity, activities, habitats, pressures, uncertainty.

Variables d'environnement (section 6), à saisir en anglais, descriptions comprises (elles étaient vides dans le chart 2.0.4) :

| Name | Description | DefaultValue |
|---|---|---|
| `CARPEDIEM_STUDY` | Study to run: a .tar.gz archive (https:// URL or s3://bucket/key) or a folder (s3://bucket/prefix) containing config/ and inputs/. ${AWS_BUCKET_NAME} is replaced by your bucket. Default: North Sea example | `https://minio.dive.edito.eu/oidc-antoningimard/carpediem/mer_du_nord/study.tar.gz` |
| `CARPEDIEM_CONFIG` | YAML configuration, relative to the study. config/config_analyse_simple.yml: simple analysis (about 1 min); config/config.yml: with Monte Carlo and robustness analyses (about 20 min with 6 CPUs) | `config/config_analyse_simple.yml` |
| `CARPEDIEM_OUTPUT` | Where the runs are stored: each run writes its own folder <run id>/ there (run.json, run.log, study/, outputs/). ${AWS_BUCKET_NAME} is replaced by your bucket | `s3://${AWS_BUCKET_NAME}/carpediem/runs` |
| `CARPEDIEM_RUN_DESCRIPTION` | Short description of this run, shown in the list of runs (e.g. "Mer du Nord, without pelagic fishing") | (vide) |
| `CARPEDIEM_WORKERS` | Number of parallel workers for the Monte Carlo analysis; empty: number of CPUs minus one | (vide) |

Ressources : la mémoire mesurée est d'environ 0,8 Go, plus environ 0,7 Go par worker
supplémentaire. Avec 8 vCPU (7 workers), prévoir environ 6 Go, d'où le maximum à 8 Gi.

Enregistrer (Save), puis lancer (Launch). Enregistrer de nouveau juste avant le lancement
pour que toute la configuration soit conservée (les processus du playground peuvent être
supprimés lors des maintenances).

## 4. Tester dans le Process Playground

Le processus apparaît sous 5 minutes dans Process catalog, catalogue « Playground »
(identifiant `process-playground`), en cherchant « carpediem ». Les événements du job
add-your-process qui suivent `Job completed` (`MountVolume.SetUp failed`…) sont des messages
de nettoyage de Kubernetes, sans effet.

1. **Test 1, analyse simple** : lancer avec les valeurs par défaut, en choisissant le bucket
   de sortie dans la configuration S3.
   * Durée attendue : 1 à 2 minutes (51 s en test local sur 2 CPU, lecture S3 comprise).
   * Journal : lignes `[carpediem] … Run <run id> started`, `Downloading study…`,
     `Running pipeline…`, `End: exit code 0, duration …`, `Writing run folder to …`.
   * Dans le bucket : `oidc-<compte>/carpediem/runs/<run id>/` avec `run.json`, `run.log`,
     `study/` et `outputs/` (`report.html`, `carte_interactive.html`,
     `resultats_analyse_*.gpkg`, `maps/`, `graphs/`). `run.json` apparaît dès le début du run
     (statut `running`). Rien à la racine du bucket : `/output-data` reste vide.
2. **Test 2, run complet** : relancer avec `CARPEDIEM_CONFIG=config/config.yml`.
   * Durée locale : 32 minutes sur 2 CPU avec 1 worker. Sur EDITO, noter la durée et le
     nombre de workers affiché dans le journal (`Workers : …`) : ces mesures alimentent
     le choix des analyses par défaut (prérequis #27).

Me transmettre `run.log` et la durée de chaque test.

## En cas d'erreur

* `Onyxia API Error: Put /api/my-lab/app Response:500` au lancement : erreur dans le
  formulaire (souvent un chemin). Ouvrir un notebook avec le rôle « edit » pour voir
  l'erreur exacte.
* Le run s'arrête au téléchargement : vérifier `CARPEDIEM_STUDY` (nom du bucket, préfixe).
* `Pipeline exit code` différent de 0 : le `run.log` est tout de même copié dans le bucket.

## Déjà vérifié en local (20 octobre 2026)

Même image, étude lue sur un serveur S3 de test avec les variables `AWS_*`, sortie dans
`/output`, configuration `config_analyse_simple.yml` : succès en 51 s sur 2 CPU.

## Historique des déploiements

| Version | Image | Résultat |
|---|---|---|
| 2.0.0 | `easyglobalmarket/carpediem:2.0.0-e5b6877` | Processus publié dans le catalogue « Playground ». Echec au lancement : `bash: /usr/local/bin/carpediem-entrypoint: Permission denied`. Le script d'entrée était copié depuis un poste où il avait les droits 600 ; dans l'image il n'était pas lisible par l'utilisateur 1000. Ressources publiées : 1 CPU et 1 Gi (requêtes et limites). |
| 2.0.1 | `easyglobalmarket/carpediem:2.0.1-e5b6877` | `Dockerfile` corrigé (droits fixés explicitement à la construction). Analyse simple réussie sur EDITO (run `carpediem-425229`, environ 55 s, 1 CPU, 1 Gi ; indicateurs identiques au run local). Mais aucune sortie dans le bucket : le chart généré par add-your-process monte le volume partagé sur `/output-data` (et non `/output` comme indiqué par le support), puis son conteneur `copy-output` copie ce dossier vers `s3://oidc-<compte>/<nom du run>/`. Les sorties écrites dans `/output` sont perdues. |
| 2.0.2 | `easyglobalmarket/carpediem:2.0.2-e5b6877` | Sans `CARPEDIEM_OUTPUT`, sortie dans `/output-data` quand ce dossier existe (sinon `/output`). Texte `${AWS_BUCKET_NAME}` remplacé par le bucket de l'utilisateur dans `CARPEDIEM_STUDY` et `CARPEDIEM_OUTPUT`. `run.log` rangé dans le dossier `results_<date>/` du run. Valeurs par défaut du formulaire : étude et sorties sous `s3://${AWS_BUCKET_NAME}/carpediem/mer_du_nord/`. Vérifié en local (serveur S3 de test, 1 CPU, 1 Go ; `/output-data` reste vide), puis sur EDITO le 21 octobre 2026 : résultats dans `oidc-antoningimard/carpediem/mer_du_nord/outputs/results_<date>/`. Avant ce déploiement, la 2.0.1 fonctionnait si on lance avec `CARPEDIEM_OUTPUT=/output-data` : vérifié sur EDITO le 20 octobre (run `carpediem-743008`, résultats dans `oidc-antoningimard/carpediem-743008/results_2026-10-20_16-04/`). |
| 2.0.2 (run complet) | `easyglobalmarket/carpediem:2.0.2-e5b6877` | Run complet (`config/config.yml`) sur EDITO le 21 octobre 2026, mémoire portée à 8 Gi : succès, 6 workers. Début à 15:26 UTC, analyse simple terminée vers 15:28, Monte Carlo terminé vers 15:41 (environ 13 min), fin non horodatée (moins d'une heure d'après Benoit). Indicateurs de robustesse déterministe identiques au run local. Journal : `Donnees Mer du Nord/V2/runs/2026-10-21_edito_complet/run.log`. |
| 2.0.3 | `easyglobalmarket/carpediem:2.0.3-0ca0789` | Journal horodaté ; au début, limites CPU et mémoire vues par le conteneur ; à la fin, durée totale et mémoire maximale (lignes aussi écrites dans `run.log`). Code avec la licence (commit `0ca0789`). Vérifié en local : l'analyse simple atteint 0,90 Gio de mémoire, trop près d'une limite de 1 Gi, d'où un minimum conseillé de 2 Gi. Image publiée le 22 octobre 2026. |
| 2.0.4 | `easyglobalmarket/carpediem:2.0.4-<commit>` | Un dossier par run sous `CARPEDIEM_OUTPUT` (`s3://${AWS_BUCKET_NAME}/carpediem/runs/<run id>/`) avec `run.json` (écrit au début avec le statut `running`, complété à la fin), `run.log`, `study/` (copie de l'étude utilisée) et `outputs/`. Nouvelle variable `CARPEDIEM_RUN_DESCRIPTION`. Version et commit inscrits dans `run.json`. Licence incluse dans l'image. Spécification : `RUNS_LAYOUT.md`. Vérifié en local (dossier local et serveur S3 de test ; run réussi, run en échec, `run.json` visible pendant le run), puis sur EDITO le 22 octobre 2026 : run complet `20261022T093031Z-test-bor-xxx` réussi en 17 min 55 s (6 CPU, 5 workers, 8 Gi), dossier complet sous `oidc-antoningimard/carpediem/runs/`, plus rien à la racine du bucket. Mémoire maximale non disponible sur les nœuds EDITO (`memory_peak_gib` vide). |
| 2.1.0 | `easyglobalmarket/carpediem:2.1.0` | Plus d'étude dans l'image (exemple Baie de Seine retiré) ; `CARPEDIEM_STUDY` obligatoire et accepte une archive `.tar.gz` (URL `https://`, clé `s3://` ou chemin local). Etude par défaut : archive publique Mer du Nord. Configuration par défaut : analyse simple. Vérifié en local (archive servie en HTTP : run réussi en 1 min 28 s ; étude absente et URL inexistante : runs en échec avec `run.json`). Déployée le 22 octobre 2026 : image publiée (étiquette `2.1.0`, sans commit), archive `study.tar.gz` publique (accès sans identifiants vérifié), processus 2.1.0 déclaré dans le Playground, run réussi avec tous les fichiers attendus. |

## Accès aux runs depuis l'extérieur d'EDITO (développement)

Pour lire `carpediem/runs/` depuis un poste de développement, utiliser les identifiants S3
temporaires du compte EDITO : dans le Datalab, menu du compte (« My account » / « Mon
compte »), onglet de connexion au stockage (« Connect to storage »). La page donne
`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_S3_ENDPOINT`
(`minio.dive.edito.eu`), `AWS_DEFAULT_REGION`, la date d'expiration, et des exemples de
configuration (variables d'environnement, rclone, MinIO client, Python). Ces identifiants
expirent : les régénérer sur la même page. Ne pas les enregistrer dans un dépôt de code.

## 5. Publication dans le catalogue officiel

Chart préparé : `edito-docker/helm/metrics-and-indicators/carpediem/` (version 2.1.0,
validé avec `helm lint` et `helm template`). Par rapport au chart généré par add-your-process :
mots-clés complets, descriptions et titres des variables, page d'accueil sur la branche
`carpediem_v2`, responsables (Antonin Gimard, Cerema ; Frédéric Quemmerais-Amice, OFB),
`README.md` (présentation, contacts, paramètres, résultats, références), pas de relance
automatique d'un run en échec (`backoffLimit: 0`, au lieu de 4 qui créerait jusqu'à cinq
dossiers de run), ressources réduites pour le conteneur `copy-output`.

Etapes (compte GitLab d'Antonin, ajouté par le support EDITO) :

1. Déployer d'abord la 2.1.0 dans le Playground (étapes 1 à 4) et la tester avec les valeurs
   par défaut, depuis un autre compte si possible (pour vérifier l'accès public à l'archive).
2. Cloner `https://gitlab.mercator-ocean.fr/pub/edito-infra/process-helm-charts`, créer une
   branche (par exemple `add-carpediem`).
3. Copier `edito-docker/helm/metrics-and-indicators/carpediem/` dans le dossier
   `metrics-and-indicators/` du dépôt.
4. Optionnel, avant de pousser : `pre-commit run --all-files` (même contrôle que la CI :
   espaces en fin de ligne, fin de fichier, format JSON).
5. Commit, push, merge request vers `main`, en mentionnant `@pub/edito-infra/codeowners`.
   La CI vérifie le format et que la version 2.1.0 n'existe pas encore dans le catalogue ;
   à la fusion, le chart est publié dans « Metrics and indicators ».

Autre voie, plus rapide mais sans ces corrections : demander au support EDITO de copier la
version du Playground dans le catalogue officiel
(`make move-chart-from-playground-to-official-catalog TARGET_CATALOG=metrics-and-indicators
CHART_NAME=carpediem CHART_VERSION=2.1.0`).

Etat : merge request !50
(https://gitlab.mercator-ocean.fr/pub/edito-infra/process-helm-charts/-/merge_requests/50,
branche `feature/add-carpediem`) fusionnée le 22 octobre 2026 ; Carpediem 2.1.0 est publié
dans le catalogue officiel « Metrics and indicators ». Une nouvelle version passe par une
nouvelle merge request avec un numéro de version supérieur dans `Chart.yaml`.
