
# Clair scanning quickplay

This repository contains a minimal setup and instructions to run Clair (v4) locally with Docker Compose and to scan a Docker image using `clairctl`.

This README is organized into: prerequisites, installing and running Clair, installing and configuring `clairctl`, pulling/using an image to scan, creating a JSON report, and troubleshooting notes.

## Prerequisites

- Docker and Docker Compose installed and working
- Internet access to download images and the `clairctl` binary
- Basic shell (bash) familiarity

## 1) Start Clair (server)

1. Clone this repository (if you haven't already):

```bash
git clone <this-repo-url>
cd clair-setup
```

1. Start Clair with Docker Compose (runs in the background):

```bash
sudo docker compose up -d
```

1. Verify containers are running:

```bash
sudo docker ps
# Look for the Clair scanner container and ensure the STATUS is "Up"
```

1. If a container is not `Up`, check its logs (example):

```bash
sudo docker logs clair_scanner
# or follow logs live
sudo docker logs -f clair_scanner
```

What happens: Clair will expose its API (typically on port 6063) and start the vulnerability indexer and scanner services. If the scanner fails to start, logs will show missing dependencies or port conflicts.

## 2) Install clairctl (CLI helper for Clair)

`clairctl` is a lightweight CLI used to interact with Clair and to run image reports.

Download and install:

```bash
wget https://github.com/quay/clair/releases/download/v4.7.3/clairctl-linux-amd64
mv clairctl-linux-amd64 clairctl
chmod +x clairctl
sudo mv clairctl /usr/local/bin/
```

Verify installation:

```bash
clairctl --version
```

What happens: You now have a small binary (`clairctl`) that will call Clair's HTTP API to analyze container images and produce reports.

## 3) Configure `clairctl`

Create a configuration directory and file for `clairctl`:

```bash
mkdir -p ~/.clairctl
```

Create `~/.clairctl/config.yml` with contents similar to:

```yaml
clair:
   uri: "http://localhost:6063"
# Optional: timeout, retries, or authentication (if Clair is protected)
```

What happens: `clairctl` reads this file to know how to reach the Clair server. If Clair is running on a different host/port, update the `uri` accordingly.

## 4) Pull the image to scan (example: NextCloud custom image)

Pull the sample image used in this repo's workflow:

```bash
sudo docker pull riazul99/nextcloud-custom:v1
```

Confirm the image is present:

```bash
sudo docker images | grep nextcloud-custom
```

What happens: The image layers are downloaded locally so `clairctl` can analyze them.

## 5) Scan the image and produce reports

Scan and show a text report in your terminal (may take a few minutes depending on image size and Clair indexing):

```bash
clairctl report --host http://localhost:6063 --out text riazul99/nextcloud-custom:v1
```

Stream the scanner logs while scanning (optional):

```bash
sudo docker logs -f clair_scanner
```

Create a JSON report (file will be created in your current directory):

```bash
clairctl report --host http://localhost:6063 --out json riazul99/nextcloud-custom:v1 > nextcloud-vulnerability-report.json
```

Pretty-print the JSON report for review:

```bash
python3 -m json.tool nextcloud-vulnerability-report.json
```

What happens: `clairctl` downloads image layers (if needed), submits them to Clair, and Clair responds with detected vulnerabilities. The JSON file contains details about each finding (severity, package, version, fixed-by info when available).

## 6) Files in this repository

- `docker-compose.yml` — Compose config that runs Clair and any required services
- `clair-config/config.yaml` — Clair's configuration file used by the service (if present)
- `nextcloud-vulnerability-report.json` — Example JSON output (if you generated it locally)

## Troubleshooting & tips

- If `clairctl` cannot connect, confirm Clair is listening on the configured host/port. Use `sudo docker ps` and `sudo docker logs clair_scanner`.
- If scan fails with timeouts, increase network or API timeouts in `clairctl` (or run locally on the same host to reduce latency).
- If you see many "unknown" or "No fix" results, that means the vulnerability database doesn't list a known fix for that package/version.
- Keep Clair's database up to date and ensure network access for fetching vulnerability feeds.

## Example quick checklist

1. Start Clair: `sudo docker compose up -d`
1. Install `clairctl` and create `~/.clairctl/config.yml` pointing to `http://localhost:6063`
1. Pull and scan an image: `sudo docker pull <image>` then `clairctl report --host http://localhost:6063 --out json <image> > report.json`


