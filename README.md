![Docker](https://img.shields.io/badge/docker-24.x-blue)
![CKAN](https://img.shields.io/badge/CKAN-2.11.5-orange)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-14-blueviolet)
![Redis](https://img.shields.io/badge/Redis-7-red)

# Fair3R CKAN — local development environment

Docker Compose stack to run [Fair3R](https://fair3r.fr) (CKAN 2.11) locally,
for developing and testing its extensions:
[ckanext-fair3r](https://github.com/Phen-ICS/ckanext-fair3r),
[ckanext-doi](https://github.com/Phen-ICS/ckanext-doi),
[ckanext-pages](https://github.com/Phen-ICS/ckanext-pages),
[ckanext-plotly](https://github.com/Phen-ICS/ckanext-plotly).

This repo covers local development only. Validation/production deployment is
managed separately.

Includes:

- `ckan` (CKAN web instance + xloader worker, managed by `supervisord`)
- `db` (`postgres:14`)
- `solr` (`ckan/ckan-solr:2.11-solr9`)
- `redis` (`redis:7-alpine`)
- `nginx` (local HTTPS access)

## 1) Prerequisites

- Docker Engine + Docker Compose plugin
- Ports used: `8085` (HTTPS through nginx), `5000` (direct CKAN, dev override), `5435` (optional DB access from host)

## 2) Setup `.env`

```bash
cp .env.example .env
```

Update at least:

- Secrets: `CKAN_SESSION_SECRET`, `CKAN_APP_INSTANCE_UUID`, `CKAN_DB_PASSWORD`,
  `CKAN_DATASTORE_DB_PASSWORD`, `CKAN_DATASTORE_READONLY_PASSWORD`
- `CKAN_SITE_URL` (e.g. `https://localhost:8085`)
- `FAIR3R_CONTEXT=DEV`
- `FDF_SCHEMA_LOCAL_PATH` — absolute host path to a clone of
  [`fair3r-fdf-schema`](https://github.com/Phen-ICS/fair3r-fdf-schema) (the
  directory containing `fdf_schema.json`). Leave empty to have the entrypoint
  download the latest schema from GitHub instead.

## 3) Get the extensions

```bash
git clone https://github.com/Phen-ICS/ckanext-fair3r.git src_extensions/ckanext-fair3r
git clone https://github.com/Phen-ICS/ckanext-doi.git src_extensions/ckanext-doi
git clone https://github.com/Phen-ICS/ckanext-pages.git src_extensions/ckanext-pages
git clone https://github.com/Phen-ICS/ckanext-plotly.git src_extensions/ckanext-plotly
```

`src_extensions` is mounted at `/plugins`; the entrypoint installs each
mounted directory with `pip install -e` on every container start (directory
name doesn't need to match the pip distribution name).

Optionally, clone the FDF schema repo anywhere outside `src_extensions` and
point `FDF_SCHEMA_LOCAL_PATH` at it to edit schema/translations live without
rebuilding:

```bash
git clone https://github.com/Phen-ICS/fair3r-fdf-schema.git /path/to/fair3r-fdf-schema
```

After changing `FDF_SCHEMA_LOCAL_PATH`, recreate the CKAN container so Compose
reattaches the bind mount (`docker compose up -d` is enough; `restart` is
not). Schema JSON edits are picked up on the next page load — no restart
needed.

## 4) Start

```bash
docker compose up -d --build
```

Editable installs re-apply on every container start; CKAN's dev reloader
watches code changes where supported.

Other useful commands:

```bash
docker compose ps
docker compose logs -f ckan
docker compose restart          # restart without rebuilding
docker compose down             # stop and remove containers/networks
docker compose down -v --remove-orphans   # full reset, also removes volumes
```

## 5) Create users

Regular user:

```bash
docker compose exec ckan ckan -c /srv/app/ckan.ini user add jdoe \
  email=jdoe@example.com \
  password=StrongPassword123! \
  fullname="John Doe"
```

Sysadmin:

```bash
docker compose exec ckan /srv/app/scripts/create_sysadmin.sh admin admin@example.com "StrongPassword123!"
```

Or set `CKAN_BOOTSTRAP_SYSADMIN_NAME` / `_EMAIL` / `_PASSWORD` in `.env` for
automatic bootstrap at startup.

## 6) Notes

- `CKAN_INTERNAL_SITE_URL` must stay reachable from inside the `ckan`
  container (default `http://ckan:5000`), otherwise xloader fails (e.g. CSV
  uploads to the datastore).
- Startup is idempotent: fresh DB runs `ckan db init`, existing DB runs `ckan db upgrade`.
- DEV uses Python 3.12.13 (from the `ckan-base:2.11.5` image, itself on
  Debian trixie).

## 7) Updating translations

The UI is available in English (default) and French. Maintain translations in
`ckanext-fair3r`, `ckanext-doi`, and `ckanext-pages` (each ships
`i18n/<domain>.pot` and `i18n/fr/LC_MESSAGES/<domain>.{po,mo}` — commit both).

Wrap new user-visible strings before extracting: Python `toolkit._()`, Jinja
`{{ _('…') }}` / `{% trans %}`, CKAN JS modules `_("…")`.

```bash
docker compose exec ckan bash -lc '
  for ext in fair3r doi pages; do
    cd /plugins/ckanext-$ext
    python setup.py extract_messages
    python setup.py update_catalog -l fr
  done
'
```

Edit `i18n/fr/LC_MESSAGES/*.po` — fill empty `msgstr`, remove fuzzy flags — then:

```bash
docker compose exec ckan bash -lc '
  for ext in fair3r doi pages; do
    cd /plugins/ckanext-$ext
    python setup.py compile_catalog -l fr
  done
'
docker compose restart ckan
```

FDF schema strings (`fdf_schema.json`) are translated separately, via sidecar
`i18n/<locale>.json` files maintained in
[`fair3r-fdf-schema`](https://github.com/Phen-ICS/fair3r-fdf-schema):

```bash
cd /path/to/fair3r-fdf-schema
python tools/i18n.py template --locale fr   # refresh keys after schema edits
python tools/i18n.py check --locale fr      # verify all keys are translated
```

Sidecar keys use stable ids, e.g.
`sections.title.fields.publication_year.help`. English text in
`fdf_schema.json` is the fallback when a translation is missing.
