# Swiss Bank Project

> **Important**
>
> - This bot is designed **only for fraudulent activity escalations**. It is not a general customer-support or banking assistant. Use it only to report, triage, and escalate suspected fraud.
> - Gmail may put the verification code in **Spam**. If you do not see the OTP in Inbox, check **Spam / Promotions**.

This project has two main parts:
- Frontend: [swiss_bank_UI](swiss_bank_UI)
- Backend: [swiss_bank_agent/backend](swiss_bank_agent/backend)

The backend expects these environment variables:
- `MONGODB_CONNECTION_STRING`
- `MONGODB_DATABASE_NAME` (optional, defaults to `swiss_bank`)
- `ANTHROPIC_API_KEY` (required — all AI agents fail without this)
- `SMTP_SERVER`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`
- `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER`
- `VITE_BACKEND_URL` for the frontend build

---

### AWS Deployment

### 1. Launch EC2

1. Go to AWS Console → EC2 → Launch Instance
2. Choose **Ubuntu 22.04 LTS**
3. Instance type: **t2.xlarge** or **t3.large** (minimum — ML models need 8GB RAM)
4. Create a key pair — download the `.pem` file, you cannot get it again
5. Open inbound ports in Network settings:
   - SSH: `22`
   - HTTP: `80`
   - HTTPS: `443`
   - Backend API: `8001`

Connect from your Mac terminal. After SSH, every command runs on the EC2 server:
```bash
chmod 400 ~/Downloads/your-key.pem
ssh -i ~/Downloads/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

---

### 2. Install system packages

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl python3-pip python3-venv nginx screen python3.11 python3.11-venv
```

Install Node.js 20:
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

---

### 3. Clone the repository

```bash
cd /home/ubuntu
git clone https://github.com/IQuest-Solutions-Corp/swiss_bank.git
```

If prompted for credentials, use your GitHub username and a Personal Access Token (not your password). Generate one at https://github.com/settings/tokens with `repo` scope.

---

### 4. Install MongoDB

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

### 4b. Add a test customer (required for EVA login)

EVA only works for emails that exist in MongoDB `customers`. Add your test email:

```bash
cd /home/ubuntu/swiss_bank/swiss_bank_agent/backend
source venv/bin/activate
python3 -c "
import asyncio, os, uuid
from datetime import datetime
from motor.motor_asyncio import AsyncIOMotorClient
from dotenv import load_dotenv
load_dotenv()

async def main():
    client = AsyncIOMotorClient(os.getenv('MONGODB_CONNECTION_STRING'))
    db = client[os.getenv('MONGODB_DATABASE_NAME', 'swiss_bank')]
    await db.customers.update_one(
        {'email': 'YOUR_EMAIL@gmail.com'},
        {'\$set': {
            'customer_id': str(uuid.uuid4()),
            'name': 'Test User',
            'email': 'YOUR_EMAIL@gmail.com',
            'phone': '+1234567890',
            'account_type': 'premium',
            'account_status': 'active',
            'is_verified': True,
            'registration_date': datetime.now().isoformat()
        }},
        upsert=True
    )
    print('Customer added')
    client.close()

asyncio.run(main())
"
---

### 5. Set up Python backend

```bash
cd /home/ubuntu/swiss_bank/swiss_bank_agent/backend
python3.11 -m venv venv
source venv/bin/activate
pip install torch --index-url https://download.pytorch.org/whl/cpu
pip install -r ../../requirements.txt
python -m spacy download en_core_web_sm
```

---

### 6. Create the .env file

```bash
nano .env
```

Paste your credentials:
```env
MONGODB_CONNECTION_STRING=mongodb://localhost:27017/
MONGODB_DATABASE_NAME=swiss_bank
REDIS_URL=redis://localhost:6379
USE_REDIS=false
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-gmail@gmail.com
SMTP_PASSWORD=your-gmail-app-password
TWILIO_ACCOUNT_SID=your-twilio-sid
TWILIO_AUTH_TOKEN=your-twilio-auth-token
TWILIO_PHONE_NUMBER=+1234567890
ANTHROPIC_API_KEY=sk-ant-your-key-here
EVA_API_KEY=sk-ant-your-eva-key
TRIAGE_API_KEY=sk-ant-your-triage-key
AGENT_API_KEY=sk-ant-your-agent-key
RAG_API_KEY=sk-ant-your-rag-key
CHROMA_DB_PATH=./chroma_db
CHROMA_COLLECTION_NAME=project_documents
USE_ANTHROPIC=true
FROM_EMAIL=noreply@swissbank.com
FROM_NAME=Swiss Bank Customer Service
AUTH_SESSION_TIMEOUT_MINUTES=30
PORT=8001
ENVIRONMENT=production
HOST=0.0.0.0
HF_HUB_DISABLE_SYMLINKS_WARNING=1
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

> `HOST` must be `0.0.0.0` (not `127.0.0.1`) or the browser cannot reach the API on port `8001`.

---

### 7. Start the backend permanently (systemd)

> Do **not** rely on `screen` / Mac terminal. Use systemd so the backend keeps running after you disconnect and after reboot.

Create the service:
```bash
sudo nano /etc/systemd/system/swissbank.service
```

Paste:
```ini
[Unit]
Description=Swiss Bank FastAPI Backend
After=network.target mongod.service

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/swiss_bank/swiss_bank_agent/backend
EnvironmentFile=/home/ubuntu/swiss_bank/swiss_bank_agent/backend/.env
ExecStart=/home/ubuntu/swiss_bank/swiss_bank_agent/backend/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8001
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable swissbank
sudo systemctl start swissbank
sudo systemctl status swissbank --no-pager
```

You should see `Active: active (running)`.

Test:
```bash
curl http://127.0.0.1:8001/health
```

Useful commands later:
```bash
sudo systemctl restart swissbank
sudo systemctl status swissbank --no-pager
sudo journalctl -u swissbank -n 50 --no-pager
```

---

### 7b. If disk runs out during pip install

Default EC2 root disk (8GB) is too small for torch. Expand the EBS volume to 30GB in AWS, then on EC2:
```bash
sudo growpart /dev/xvda 1
sudo resize2fs /dev/root
df -h
```

---

### 8. Build the frontend

```bash
cd /home/ubuntu/swiss_bank/swiss_bank_UI
npm install
VITE_BACKEND_URL=http://YOUR_EC2_PUBLIC_IP:8001 npm run build
```

---

### 9. Serve with Nginx

```bash
sudo nano /etc/nginx/sites-available/swissbank
```

Paste:
```nginx
server {
    listen 80;
    server_name YOUR_EC2_PUBLIC_IP;

    location / {
        root /home/ubuntu/swiss_bank/swiss_bank_UI/dist;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Save, then enable:
```bash
sudo ln -s /etc/nginx/sites-available/swissbank /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo chmod 755 /home/ubuntu
sudo chmod -R 755 /home/ubuntu/swiss_bank/swiss_bank_UI/dist
sudo nginx -t
sudo systemctl restart nginx
```

Open `http://YOUR_EC2_PUBLIC_IP` in your browser.

---

### 10. Checklist

- [ ] EC2 running, ports 22/80/443/8001 open
- [ ] MongoDB running: `sudo systemctl status mongod`
- [ ] Backend systemd service running: `sudo systemctl status swissbank`
- [ ] Backend health locally: `curl http://127.0.0.1:8001/health`
- [ ] Public health check works in browser: `http://YOUR_EC2_PUBLIC_IP:8001/health`
- [ ] Frontend built with correct `VITE_BACKEND_URL`
- [ ] Nginx serving from `/dist`: `sudo systemctl status nginx`
- [ ] Test customer exists in MongoDB (for EVA verify)

---

## Local Development

### Terminal 1 — Backend
```bash
cd swiss_bank_agent/backend
python3.11 -m venv venv
source venv/bin/activate
python main.py
```

### Terminal 2 — Frontend
```bash
cd swiss_bank_UI
npm install
npm run dev
```

> Customer records must exist in the `customers` MongoDB collection for EVA authentication to work.

Frontend: `http://localhost:8080` — Backend: `http://localhost:8001`

> For local dev, MongoDB must be running (`brew services start mongodb-community` on Mac).
> Customer records must exist in the `customers` MongoDB collection for EVA authentication to work.
```
