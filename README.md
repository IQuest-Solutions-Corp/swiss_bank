# Swiss Bank Project

This project has two main parts:
- Frontend: [swiss_bank_UI](swiss_bank_UI)
- Backend: [swiss_bank_agent/backend](swiss_bank_agent/backend)

For a production-style AWS deployment, the simplest setup is:
- 1 EC2 instance to host the backend and frontend
- 1 MongoDB database (MongoDB Atlas or Amazon DocumentDB)
- Frontend configured to call the backend through the EC2 public IP or a domain

The backend expects these environment variables:
- `MONGODB_CONNECTION_STRING` (important: this project uses this name, not `MONGODB_URI`)
- `MONGODB_DATABASE_NAME` (optional, defaults to `swiss_bank`)
- `REDIS_URL` (optional for local Redis, but recommended for production)
- `SMTP_SERVER`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`
- `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER`
- `VITE_BACKEND_URL` for the frontend

---

## 1. Create an EC2 instance on AWS

### Launch the instance
1. Go to AWS Console → EC2 → Launch Instance.
2. Choose Ubuntu 22.04 LTS.
3. Pick a small instance size such as `t3.small` or `t3.medium`.
4. Create or select a key pair.
5. In Network settings, allow these inbound rules:
   - SSH: `22`
   - HTTP: `80`
   - HTTPS: `443`
   - Backend API: `8001`

### Connect to the instance
```bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

### Install required packages
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl python3-pip python3-venv nginx nodejs npm
```

If Node.js is not available as expected, install it with:
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

---

## 2. Create MongoDB on AWS

You have two good options:

### Option A: MongoDB Atlas (recommended for easiest setup)
1. Create a MongoDB Atlas account.
2. Create a new cluster in the same AWS region as your EC2 instance.
3. Create a database user.
4. Add your EC2 public IP to the Atlas Network Access list.
5. Copy the connection string.

Example connection string:
```text
mongodb+srv://<username>:<password>@<cluster-url>/swiss_bank?retryWrites=true&w=majority
```

### Option B: Amazon DocumentDB
If you want all services inside AWS, use Amazon DocumentDB with MongoDB compatibility.
1. Create a DocumentDB cluster.
2. Set the username/password.
3. Allow inbound access from your EC2 security group.
4. Use the DocumentDB endpoint in the backend environment.

> For this project, the backend reads `MONGODB_CONNECTION_STRING`, so make sure the value you place there matches the database you created.

---

## 3. Connect MongoDB to the backend

On the EC2 instance, go to the backend folder:
```bash
cd /home/ubuntu
git clone <your-repo-url> swiss_bank
cd swiss_bank/swiss_bank_agent/backend
python3 -m venv venv
source venv/bin/activate
pip install -r ../../requirements.txt
```

Create a `.env` file in the backend folder:
```bash
nano .env
```

Example contents:
```env
MONGODB_CONNECTION_STRING=mongodb+srv://<username>:<password>@<cluster-url>/swiss_bank?retryWrites=true&w=majority
MONGODB_DATABASE_NAME=swiss_bank
REDIS_URL=redis://localhost:6379
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your-email@gmail.com
SMTP_PASSWORD=your-app-password
TWILIO_ACCOUNT_SID=your-twilio-sid
TWILIO_AUTH_TOKEN=your-twilio-token
TWILIO_PHONE_NUMBER=+1234567890
```

Start the backend:
```bash
source venv/bin/activate
uvicorn main:app --host 0.0.0.0 --port 8001
```

You can also run it in the background with `tmux` or `screen`.

Test the backend:
```bash
curl http://127.0.0.1:8001/docs
```

If the backend cannot connect to MongoDB, check:
- The connection string is correct
- The database user has permission to the database
- The security group or network access allows the connection

---

## 4. Build and connect the frontend

Go to the frontend folder:
```bash
cd /home/ubuntu/swiss_bank/swiss_bank_UI
npm install
```

Build the frontend with the backend URL:
```bash
VITE_BACKEND_URL=http://YOUR_EC2_PUBLIC_IP:8001 npm run build
```

If you are using a domain, use that instead:
```bash
VITE_BACKEND_URL=https://your-domain.com npm run build
```

The frontend uses `VITE_BACKEND_URL` from [swiss_bank_UI/src/lib/config.ts](swiss_bank_UI/src/lib/config.ts), so this value must be set correctly or the app will call the wrong backend.

---

## 5. Serve the frontend with Nginx

Create an Nginx config:
```bash
sudo nano /etc/nginx/sites-available/swissbank
```

Example config:
```nginx
server {
    listen 80;
    server_name your-domain.com;  # or your EC2 public IP

    location / {
        root /home/ubuntu/swiss_bank/swiss_bank_UI/dist;
        try_files $uri $uri/ /index.html;
    }
}
```

Enable it:
```bash
sudo ln -s /etc/nginx/sites-available/swissbank /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 6. Make sure everything is connected

Use this checklist to confirm the full flow:
- EC2 instance is running
- Security group allows `22`, `80`, `443`, and `8001`
- MongoDB is reachable from the EC2 instance
- Backend `.env` contains the correct `MONGODB_CONNECTION_STRING`
- Frontend was built with the correct `VITE_BACKEND_URL`
- The frontend can reach the backend over HTTP/HTTPS
- The backend can read/write data in the MongoDB database

### Quick health checks
```bash
curl http://YOUR_EC2_PUBLIC_IP:8001/docs
```

Open the frontend in your browser and confirm the app loads. If the UI cannot talk to the backend, the most common cause is an incorrect `VITE_BACKEND_URL` or missing CORS settings.

---

## 7. Local development (short version)

### Terminal 1
```bash
conda activate swiss_bank
cd swiss_bank_agent/backend
python main.py
```

### Terminal 2
```bash
cd swiss_bank_UI
npm run dev
```
