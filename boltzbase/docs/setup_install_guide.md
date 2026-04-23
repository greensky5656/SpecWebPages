# Setup And Install Guide

This guide covers the current local-development setup for the root Python CLI
and the `boltz-ui` web application.

## Environment Summary

- Root Python package: `boltz`
- Root package Python range: `>=3.10,<3.13`
- UI bootstrap Python version: `3.12`
- Web app directory: `boltz-ui/`
- CLI entrypoint: `boltz`

If you need both the CLI and the UI locally, install Python `3.12` so the UI's
postinstall flow and the root package can share the same `../.venv`.

## Prerequisites

Install these before bootstrapping the repo:

- Python `3.12` recommended
- `pip`, `venv`, and build tooling for Python packages
- Node.js and `npm` for `boltz-ui/`
- A GPU-capable PyTorch environment if you plan to run production-size
  predictions locally

Optional external tools are needed only for specific workflows:

- MMseqs2-compatible server access when using `--use_msa_server`
- RCSB network access for batch PDB fetch in the UI
- whatever environment your Vina workflow requires if you enable Vina
  post-processing

## Root Python Setup

Create and activate a virtual environment in the repository root:

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
python -m pip install -e .
```

Optional extras:

```bash
python -m pip install -e ".[test]"
python -m pip install -e ".[lint]"
python -m pip install -e ".[cuda]"
```

Verify the CLI:

```bash
which boltz
boltz --help
```

## Web UI Setup

Install the frontend dependencies from `boltz-ui/`:

```bash
cd boltz-ui
npm install
```

The `postinstall` hook runs `npm run install:python`, which currently:

1. switches to the repository root
2. requires Python `3.12`
3. creates `../.venv` if needed
4. upgrades `pip`, `setuptools`, and `wheel`
5. attempts `pip install -e .`
6. attempts to install additional analysis packages such as `MDAnalysis`,
   `numpy`, `scipy`, `biopython`, `gemmi`, `tmscoring`, and `iminuit`

The install script is best-effort for some packages, so you should still verify
the runtime explicitly after install.

## Run The Web App

From `boltz-ui/`:

```bash
npm run dev
```

The development app starts on `http://localhost:3000`.

Production build commands:

```bash
npm run build
npm start
```

## Run The CLI

Example CPU run:

```bash
boltz predict examples/prot_no_msa.yaml \
  --model boltz2 \
  --accelerator cpu \
  --out_dir ./results
```

Example MSA-server run:

```bash
boltz predict examples/multimer.yaml \
  --model boltz2 \
  --use_msa_server \
  --msa_server_url https://api.colabfold.com \
  --out_dir ./results
```

## Environment Variables

Common environment variables used by the current code:

- `BOLTZ_CACHE`: absolute path override for the model cache
- `BOLTZ_MSA_USERNAME`: basic-auth username for the MSA server
- `BOLTZ_MSA_PASSWORD`: basic-auth password for the MSA server
- `MSA_API_KEY_VALUE`: API-key value for authenticated MSA servers

Do not set both basic-auth and API-key authentication for the same run. The
runtime treats that as a configuration error.

## UI Runtime Notes

The UI does not execute the root console script directly. Instead it:

1. uses `../.venv/bin/python`
2. sets `PYTHONPATH` to `boltz-main/src`
3. launches `python -m boltz.main`

That means the web app intentionally runs the `boltz-main/src/boltz` tree, even
though the repository root also contains `src/boltz`.

## Authentication And Deployment Notes

Additional app-specific notes already live in `boltz-ui/`:

- `boltz-ui/SETUP_AUTH.md`
- `boltz-ui/AUTH_FIX.md`
- `boltz-ui/HTTPS_SETUP.md`
- `boltz-ui/DEPLOYMENT_CHECKLIST.md`

Use those files for auth and deployment details that are outside the core local
prediction workflow.

## Verification Checklist

After setup, these checks should pass:

```bash
python -c "import boltz"
boltz --help
cd boltz-ui
npm test
```

If you are only validating the docs and setup wiring, `boltz --help` and the UI
test command are the quickest smoke checks.
