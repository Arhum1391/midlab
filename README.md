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
4. **Webhook (recommended):** GitHub repo → Settings → Webhooks → Payload URL `http://YOUR_JENKINS:8080/github-webhook/` (or your Multibranch webhook URL). In the job, enable “GitHub hook trigger for GITScm polling” if using that plugin.
5. **Public API URL for the form:** `http://YOUR_EC2_PUBLIC_IP:8000/metrics` (or HTTPS if you terminate TLS elsewhere).

## Config copy note

Scripts read **`config.json` at the repo root**. This repo includes both `config.json` and `configs/FA23-BAI-006_config.json` so you can restore from `configs/` if needed.
