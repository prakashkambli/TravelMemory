Overview

Travel Memory is a full-stack MERN application (MongoDB, Express.js, React.js, Node.js) that allows users to save and view travel experiences.
This guide explains how to deploy the project on a single AWS EC2 instance (Ubuntu 22.04) using Nginx as a reverse proxy and PM2 for process management.

⚙️ Tech Stack
Component	Technology
Frontend	React.js
Backend	Node.js + Express.js
Database	MongoDB (Local on EC2)
Reverse Proxy	Nginx
Process Manager	PM2
Cloud Provider	AWS EC2 (Ubuntu 22.04 LTS)
🚀 Deployment Architecture
                    ┌───────────────────────────────┐
                    │        Browser / Client       │
                    │   (http://<ec2-public-ip>)    │
                    └──────────────┬────────────────┘
                                   │
                           [ Port 80 / Nginx ]
                                   │
                 ┌─────────────────┼─────────────────┐
                 │                                   │
           / (Frontend)                        /api (Backend)
                 │                                   │
 Serves React static files              Proxies to Node.js backend
 from /var/www/html                     running on localhost:3001
                 │                                   │
                 └─────────────────┬─────────────────┘
                                   │
                           [ MongoDB Local ]
                  mongodb://127.0.0.1:27017/travel_memory

🪄 Step-by-Step Setup
1️⃣ Launch EC2 Instance

Go to AWS Console → EC2 → Launch Instance

Choose Ubuntu 22.04 LTS (64-bit x86)

Select a t2.micro (free tier) instance

Configure Security Group:

Allow SSH (22) for your IP

Allow HTTP (80) for everyone

(Optional) Allow HTTPS (443) if you plan to enable SSL

Launch the instance and connect:

ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>

2️⃣ Install Dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl nginx mongodb


Install Node.js (LTS):

curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs


Check versions:

node -v
npm -v
mongod --version

3️⃣ Clone Repository
cd ~
git clone <your_repo_url> TravelMemory
cd TravelMemory

4️⃣ Backend Setup
cd backend
npm install


Create .env:

nano .env


Example:

PORT=3001
MONGO_URI=mongodb://127.0.0.1:27017/travel_memory
JWT_SECRET=your_secret_key


Test backend:

npm run dev


✅ Expected output:

Server started at http://localhost:3001
MongoDB connected


Stop it with Ctrl + C.

5️⃣ Frontend Setup
cd ../frontend
npm install


In src/url.js:

export const baseUrl = process.env.REACT_APP_BACKEND_URL || "http://<EC2_PUBLIC_IP>:3001";


Then create .env:

nano .env


Add:

REACT_APP_BACKEND_URL=/api


Build for production:

npm run build

6️⃣ Configure Nginx

Copy build files to web root:

sudo rm -rf /var/www/html/*
sudo cp -r build/* /var/www/html/


Edit Nginx config:

sudo nano /etc/nginx/sites-available/default


Replace contents with:

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:3001/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    access_log /var/log/nginx/travelmemory_access.log;
    error_log /var/log/nginx/travelmemory_error.log;
}


Save and test:

sudo nginx -t
sudo systemctl restart nginx

7️⃣ Run Backend Persistently (PM2)

Install PM2 globally:

sudo npm install -g pm2


Start backend:

cd ~/TravelMemory/backend
pm2 start index.js --name travel-backend
pm2 save
pm2 startup


View status:

pm2 status


Logs:

pm2 logs travel-backend

8️⃣ Verify Deployment

Open in your browser:

http://<EC2_PUBLIC_IP>/


✅ Frontend should load.
✅ API calls should work through /api.

Test backend directly:

curl http://localhost:3001/api/health

🧠 Common Commands
Task	Command
Restart Nginx	sudo systemctl restart nginx
Nginx logs	sudo tail -f /var/log/nginx/error.log
Backend logs	pm2 logs travel-backend
Restart backend	pm2 restart travel-backend
Stop backend	pm2 stop travel-backend
MongoDB status	sudo systemctl status mongodb
🧩 Folder Structure
TravelMemory/
├── backend/
│   ├── index.js
│   ├── package.json
│   ├── .env
│   └── ...
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   └── url.js
│   ├── .env
│   └── package.json
└── README.md

🔐 Security Recommendations

Use AWS Security Groups to limit access (SSH only from your IP).

Enable UFW firewall (optional):

sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable


Use HTTPS with Let’s Encrypt
 (optional).

Keep your MongoDB bound to 127.0.0.1 only (no external access).

Store all secrets (JWT, DB URI) in .env, never commit them.

✅ Final Deliverables

Deployed Application:
Access via http://<EC2_PUBLIC_IP>/

Persistent Backend:
Managed via PM2

Local MongoDB:
Running as a service on EC2

Nginx Reverse Proxy:
Serves frontend + proxies /api routes

Full Documentation:
This README.md
