# RAYZO Telegram Bot Control Panel

This package separates the static dashboard from the Python bot backend.

## Files
- `index.html`: static dashboard. Deploy this file/site to Netlify.
- `control_server.py`: private API that starts/stops the existing `app.py` and serves logs.
- `app.py`: existing Telegram hosting-manager bot, configured by the backend API.
- `requirements.txt`: Python dependencies for the backend and bot.

## Important hosting limitation
Netlify hosts the static HTML only. It cannot keep this Telegram polling bot alive as a normal 24/7 Python process. The backend must run on a Linux host that supports long-running processes and has persistent storage. The bot and API run on that backend host; the HTML runs on Netlify.

This project uses `SANDBOX_MODE=process`. That mode does **not** isolate user-uploaded code. The patched `app.py` blocks non-admin project execution in process mode, but the manager still runs on the host. Use only on a trusted/private setup. For genuinely multi-user untrusted code hosting, use a Docker-enabled host with proper isolation.

## Backend environment variables (set in the backend provider dashboard, not in a .env file)
Required:
- `PANEL_PASSWORD`: a long, unique password used to access the web control panel.
- `PANEL_ENCRYPTION_KEY`: Fernet key used to encrypt the saved bot token/admin IDs at rest.
- `FRONTEND_ORIGIN`: exact Netlify origin, e.g. `https://your-site-name.netlify.app` (no trailing slash).

Generate an encryption key locally:
```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```
Copy the printed key into `PANEL_ENCRYPTION_KEY` in the backend provider's environment settings. Save it securely; if you lose/change it, the previously encrypted configuration cannot be decrypted.

Optional:
- `PANEL_DATA_DIR`: persistent directory for encrypted configuration, PID, and logs. Use the provider's persistent disk path if available.
- `PORT`: usually set automatically by the provider.
- `LOG_LEVEL`: defaults to INFO.

Do not put these values in `index.html`, GitHub, or a public repository. Do not expose `PANEL_PASSWORD` or `PANEL_ENCRYPTION_KEY` to the browser.

## Backend deploy
Run on a Linux service that allows a long-running Python process and subprocesses. Upload/push this folder to the backend host, set the environment variables above, and install requirements.

Start command (one worker only; process control is in-memory):
```bash
gunicorn --workers 1 --threads 8 --bind 0.0.0.0:$PORT control_server:app
```
Use a persistent disk for `PANEL_DATA_DIR` if you want encrypted settings and logs to survive redeploys. Do not use a serverless/function-only runtime. Keep the backend on HTTPS.

If running manually:
```bash
pip install -r requirements.txt
export PANEL_PASSWORD='use-a-long-unique-password'
export PANEL_ENCRYPTION_KEY='paste-generated-fernet-key'
export FRONTEND_ORIGIN='https://your-site-name.netlify.app'
export PORT=8080
gunicorn --workers 1 --threads 8 --bind 0.0.0.0:$PORT control_server:app
```
These are shell environment variables, not a `.env` file.

## Netlify frontend deploy
Deploy `index.html` as the site's root `index.html`. The UI asks for the backend API URL and panel password. Paste your backend URL into the dashboard. Configure `FRONTEND_ORIGIN` on the backend to match the Netlify URL exactly.

## Use
1. Open the Netlify dashboard.
2. Enter the backend API URL and the private `PANEL_PASSWORD`.
3. Enter the BotFather token and numeric Telegram admin ID(s).
4. Click **Start bot**. The token/admin IDs are saved encrypted on the backend and passed to `app.py` as environment variables; no `.env` file is required.
5. Use **Stop bot** and live logs from the same dashboard.

## Operational notes
- The backend API does not return the saved token to the browser.
- The bot token field is cleared after a successful start.
- Never share the panel password or publish the backend URL with credentials.
- This package has not been live-deployed or end-to-end tested against Telegram. Python syntax was checked locally; deployment still requires configuring a real backend host.
