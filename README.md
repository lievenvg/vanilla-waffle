# Vanilla Waffle — Postman API Tests

Deze repo bevat:
- Postman collectie(s) in `postman/collections/`
- Environments in `postman/environments/`
- (Optioneel) datafiles in `postman/data/`

## Shared workflow
De CI workflow gebruikt een **shared reusable workflow** via:

```yaml
uses: lievenvg/workflows/.github/workflows/postman-shared.yml@v1
```

Pas dit aan indien jouw shared workflow elders staat, of maak in `lievenvg/workflows` een bestand `.github/workflows/postman-shared.yml` (zie voorbeeld onderaan).

## Secrets
Zet je secrets (bv. API key) in **Settings → Secrets and variables → Actions** als `API_KEY`.

In de Postman environments staat `apiKey` op `{{API_KEY}}`; de workflow injecteert die via `--env-var`.

## Lokaal draaien
1. Installeer Newman: `npm i -g newman newman-reporter-htmlextra`
2. Run:
   ```bash
   newman run postman/collections/my-api.postman_collection.json          -e postman/environments/dev.postman_environment.json          --env-var "API_KEY=$API_KEY"          --reporters cli,htmlextra          --reporter-htmlextra-export newman-report.html
   ```

## Voorbeeld shared workflow (in je shared repo)
Maak in repo `lievenvg/workflows` dit bestand: `.github/workflows/postman-shared.yml`

```yaml
name: Shared - Run Postman Tests

on:
  workflow_call:
    inputs:
      collection_path:
        required: true
        type: string
      environment_path:
        required: true
        type: string
      reporters:
        required: false
        type: string
        default: 'cli'
      fail_on_error:
        required: false
        type: boolean
        default: true
      node_version:
        required: false
        type: string
        default: '20'
    secrets:
      API_KEY:
        required: false

jobs:
  run-postman:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node_version }}
      - name: Install Newman + reporters
        run: |
          npm i -g newman
          npm i -g newman-reporter-htmlextra || true
          npm i -g newman-reporter-junitfull || npm i -g newman-reporter-junit || true
      - name: Run Newman
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          mkdir -p artifacts
          REPORTERS="${{ inputs.reporters }}"
          EXTRA_FLAGS=""
          case ",${REPORTERS}," in
            *,junit,*) EXTRA_FLAGS="$EXTRA_FLAGS --reporter-junit-export artifacts/newman-results.xml";;
          esac
          case ",${REPORTERS}," in
            *,htmlextra,*) EXTRA_FLAGS="$EXTRA_FLAGS --reporter-htmlextra-export artifacts/newman-report.html";;
          esac
          newman run "${{ inputs.collection_path }}"                 -e "${{ inputs.environment_path }}"                 --reporters "$REPORTERS"                 --env-var "API_KEY=$API_KEY"                 $EXTRA_FLAGS || EXIT_CODE=$?
          if [ "${{ inputs.fail_on_error }}" = "true" ] && [ "${EXIT_CODE:-0}" -ne 0 ]; then
            exit $EXIT_CODE
          fi
      - name: Upload artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: postman-results
          path: artifacts/
          if-no-files-found: ignore
```

---

## Quickstart met deze repo
1) Zet `API_KEY` als repository secret.
2) Pas eventueel `baseUrl` waarden in de environment JSON's aan.
3) Push naar `main` of start handmatig via **Actions → CI - Postman via Shared Workflow → Run workflow**.

### Git-commando's (als de repo leeg is)
```bash
git init
git branch -M main
git remote add origin https://github.com/lievenvg/vanilla-waffle.git
git add .
git commit -m "Initialize Postman tests with reusable workflow"
git push -u origin main
```
