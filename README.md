# Security Scan Pipeline

Pipeline GitHub Actions d'audit de sécurité automatisé : reconnaissance →
scan de vulnérabilités → agrégation → rapport (Markdown + PDF + SARIF).

## ⚠️ Avant d'utiliser ce pipeline

Ce workflow lance des outils de scan **actifs** (Nikto, OWASP ZAP, SQLMap)
qui envoient du trafic d'attaque réel vers la cible renseignée. Ne l'exécute
que contre :

- ton propre domaine / infrastructure,
- une cible pour laquelle tu as un mandat écrit (contrat de pentest),
- un programme de bug bounty qui autorise explicitement ce type de scan.

Le workflow inclut une case `authorization_confirmed` obligatoire qui bloque
le run si elle n'est pas cochée — mais cette case ne remplace pas une vraie
autorisation légale, elle ne fait que t'empêcher de lancer un scan par
accident.

## Structure du repo

```
.github/workflows/security-scan.yml   # le pipeline
scripts/aggregate.py                  # script d'agrégation des résultats
```

## Lancer le pipeline

Dans l'onglet **Actions** du repo → sélectionner "Security Scan Pipeline" →
**Run workflow**, puis renseigner :

| Input | Description |
|---|---|
| `target` | Domaine ou IP cible, sans `http://` (ex: `example.com`) |
| `scan_type` | `full` (recon + vuln complet, seule option actuellement) |
| `output_format` | `markdown+pdf` ou `markdown-only` |
| `runner` | `github-hosted` (recommandé) ou `self-hosted` |
| `authorization_confirmed` | Doit être cochée pour lancer le scan |

## Runner : github-hosted vs self-hosted

**`github-hosted` est recommandé par défaut** pour deux raisons :

1. **Isolation** — le runner est un conteneur éphémère détruit après le job.
   Si quelqu'un compromet le repo (PR malveillante, secret leak), il n'a
   pas un accès persistant à une machine que tu contrôles.
2. **Attribution réseau** — le trafic de scan part des IP de GitHub, pas de
   ton réseau perso ou de ton serveur. Utile si la cible bloque/loggue les
   IP suspectes : tu ne grilles pas ta propre IP.

Utilise `self-hosted` seulement si :
- la cible est sur un réseau interne non exposé sur Internet (donc
  inatteignable depuis les runners GitHub), ou
- tu as des contraintes de licence/outils qui nécessitent ta propre machine.

Pour configurer un runner self-hosted Debian, voir la
[doc officielle GitHub](https://docs.github.com/en/actions/hosting-your-own-runners).

## Secrets à configurer (Settings → Secrets and variables → Actions)

| Secret | Requis | Usage |
|---|---|---|
| `SHODAN_API_KEY` | Optionnel | Si absent, l'étape Shodan est sautée proprement (pas d'échec du job) |

## Limites connues

- **OWASP ZAP** est lancé en mode `baseline` (passif/léger). Un scan actif
  complet (`zap-full-scan.py`) est plus long et plus intrusif — à activer
  manuellement si besoin et autorisé.
- **SQLMap** tourne avec `--crawl=1` et `--batch` : crawl minimal, pas
  d'interaction manuelle. Pour un audit plus poussé, il faut cibler des
  endpoints précis plutôt que crawler depuis la racine.
- **Trivy** ici scanne la config du repo (`scan-type: config`), pas l'image
  Docker de la cible — adapte `scan-ref`/`scan-type` si tu veux scanner une
  image ou un filesystem précis.
- Le **score CVSS** calculé par `aggregate.py` est une estimation dérivée
  de la sévérité native de chaque outil (et du score CVSS natif quand Trivy
  le fournit), pas un calcul vectoriel CVSS complet.
- Le **PDF n'est pas signé cryptographiquement** : un hash SHA256
  (`report.pdf.sha256`) est généré comme preuve d'intégrité basique. Si tu
  veux une vraie signature (GPG, certificat X.509), c'est une étape à
  ajouter séparément.
- Les artifacts intermédiaires (`recon-results`, `vuln-results`,
  `aggregated-report`) ont une rétention de **1 jour** ; seul l'artifact
  final `security-report-<target>` est gardé **30 jours**, conformément au
  schéma d'origine.

## Étendre le pipeline

- Pour ajouter un nouvel outil : ajoute son étape dans le job
  `reconnaissance` ou `vulnerability-scan`, fais-lui écrire sa sortie dans
  `scan-results/<outil>.<format>`, puis ajoute un parser correspondant dans
  `scripts/aggregate.py` (suis le pattern des fonctions `parse_*`
  existantes — chacune doit être défensive et ne jamais lever d'exception
  non gérée si le fichier est absent ou malformé).
