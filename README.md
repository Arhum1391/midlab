# Midterm ML pipeline (FA23-BAI-006)

Repository for CSC413 lab midterm: GitHub stores `dataset/train.csv`; Jenkins on EC2 retrains, rebuilds Docker, and serves FastAPI on port **8000** (`GET /metrics`).

## Layout

| Path | Purpose |
|------|---------|
| `config.json` | Student config (same values as `configs/FA23-BAI-006_config.json`). |
| `dataset/train.csv` | Training data (commit this; instructor may push updates). |
| `generate_dataset.py` | Regenerate `train.csv` locally from `config.json` if needed. |
| `train.py` | Produces `model.pkl` and `metrics.json` (not committed). |
| `Jenkinsfile` | Pipeline: checkout → train → docker build → run container. |

## Local (optional)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python generate_dataset.py
python train.py
docker build -t midlab-api:test .
docker run --rm -p 8000:8000 midlab-api:test
```

## EC2 checklist

1. **Security group:** inbound TCP **8000** from the internet (or evaluator IP if restricted).
2. **Jenkins:** install Git, Python 3, Docker, `curl`. Add user `jenkins` to group `docker`, then restart Jenkins (`sudo usermod -aG docker jenkins`).
3. **Job:** New Item → Pipeline → Pipeline script from SCM → Git URL of this repo → branch `main` → Script Path `Jenkinsfile`.
4. **GitHub webhook (required for auto-build on push):** see section below.
5. **Public API URL for the form:** `http://YOUR_EC2_PUBLIC_IP:8000/metrics` (or HTTPS if you terminate TLS elsewhere).

## GitHub webhook + Jenkins settings

### Plugins (Manage Jenkins → Plugins)

Install if missing:

- **GitHub Integration** (often named **GitHub** / `github` plugin) — provides `/github-webhook/` and the “GitHub hook trigger for GITScm polling” option.
- **Git** — Pipeline checkout from GitHub.

Restart Jenkins after installing plugins.

### Jenkins job (Pipeline from SCM)

1. **New Item** → **Pipeline** → name your job → OK.
2. Under **Pipeline**, choose **Pipeline script from SCM**.
3. **SCM:** Git. **Repository URL:** your `midlab` repo HTTPS or SSH URL.
4. **Credentials:** add GitHub PAT or SSH key if the repo is private.
5. **Branch specifier:** `*/main`.
6. **Script Path:** `Jenkinsfile`.
7. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling** (wording may be slightly different depending on plugin version).
8. Save the job.

Optional but helpful: under the job’s **General** section, if you see **GitHub project**, set **Project url** to `https://github.com/OWNER/REPO/` so Jenkins links builds to the repo.

### EC2 network for webhooks

- GitHub must reach Jenkins on **port 8080** (or whatever you use). Open that port in the **EC2 security group** from **GitHub’s IPs** (narrow) or temporarily from `0.0.0.0/0` for testing — tighten later if your org allows.
- If Jenkins is only on `localhost`, use a **reverse proxy + HTTPS** or **Tailscale**/VPN; GitHub cannot call `http://127.0.0.1:8080/...`.

### GitHub repository

1. Repo → **Settings** → **Webhooks** → **Add webhook**.
2. **Payload URL:** `http://YOUR_EC2_PUBLIC_DNS_OR_IP:8080/github-webhook/`  
   (Use `https://` if Jenkins is behind HTTPS.)
3. **Content type:** `application/json`.
4. **Events:** “Just the push event” (or “Let me select…” and enable **Pushes**).
5. **Active:** checked → **Add webhook**.

After a push to `main`, GitHub POSTs to Jenkins; the job should queue within seconds. Use **Recent Deliveries** on the webhook to debug HTTP errors (401/403/404/timeout).

### If webhooks are blocked

Use **Polling SCM** on the job with a schedule (e.g. `H/5 * * * *`) only as a fallback — your instructor expects near-immediate retrains after they push; webhook is the right default.

## Config copy note

Scripts read **`config.json` at the repo root**. This repo includes both `config.json` and `configs/FA23-BAI-006_config.json` so you can restore from `configs/` if needed.
