# 3-Tier DevSecOps Project

This repository contains a simple Node.js API and a React client used for a user management demo. Follow the steps below to get the project running locally.

## Setup

1. Install Node.js (version 18 or later is recommended).
2. Install dependencies for both the API and client:

   ```bash
   cd api && npm install
   cd ../client && npm install
   ```

3. Start the API server:

   ```bash
   cd api
   npm start
   ```

4. In a separate terminal, start the React client:

   ```bash
   cd client
   npm start
   ```

5. Open `http://localhost:3000` in your browser to use the application.

The client now displays an animated banner welcoming you to **DevOps Shack**.



---

Got it 👍 I understand your context and notes.
You want a **clean, human-written README.md for `local-dev` branch** — simple, practical, and not “AI-looking”.

I’ll write it in your style: **clear, to the point, beginner-friendly, real DevOps thinking**.

---

# Local Development Setup (local-dev branch)

This branch is used to run the application locally without Docker or Kubernetes.
The goal is to understand how frontend, backend, and database work together.

---

## 🧠 Architecture Overview

This is a simple 3-tier application:

* **Frontend**

  * Built using React (JavaScript library)
  * Runs in browser
  * Port: `3000`

* **Backend**

  * Built using Node.js + Express
  * Handles API requests
  * Connects to database

* **Database**

  * MySQL
  * Stores user data

👉 Important:

* Frontend talks to Backend
* Backend talks to Database
* Frontend NEVER talks directly to database

---

## 📦 package.json & package-lock.json

### package.json

Defines:

* Dependencies (React, Express, MySQL)
* Scripts:

  * `npm start`
  * `npm install`
  * `npm test`

### package-lock.json

* Auto-generated
* Locks exact versions of dependencies
* Ensures same behavior on all machines (local, CI/CD, production)

---

## ⚙️ Prerequisites

* Linux machine / VM / EC2
* Node.js (using NVM)
* MySQL
* Git

---

## 🟢 Step 1: Install Node.js (using NVM)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
```

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

```bash
\. "$HOME/.nvm/nvm.sh"
nvm install 25
```

Verify:

```bash
node -v
npm -v
```

---

## 📥 Step 2: Clone Repository

```bash
git clone https://github.com/bilalamjad-devops/3-Tier-DevSecOps-Mega-Project
cd 3-Tier-DevSecOps-Mega-Project
git checkout local-dev
```

---

## 🗄 Step 3: Setup MySQL

Install:

```bash
sudo apt install mysql-server -y
```

Login:

```bash
sudo mysql
```

Set password:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'admin';
FLUSH PRIVILEGES;
EXIT;
```

Login again:

```bash
sudo mysql -u root -p
```

Create database and table:

```sql
CREATE DATABASE IF NOT EXISTS crud_app;
USE crud_app;

CREATE TABLE IF NOT EXISTS users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  role ENUM('admin', 'viewer') NOT NULL DEFAULT 'viewer',
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Exit:

```bash
exit
```

---

## 🔐 Step 4: Configure Environment Variables

### Frontend (.env)

```bash
cd client
vi .env
```

Update:

```
REACT_APP_API=http://<your-public-ip>:5000
```

👉 This connects frontend to backend

---

## ⚙️ Step 5: Run Backend

```bash
cd api
npm install
npm start
```

* Installs dependencies → `node_modules/`
* Starts backend server

---

## ⚙️ Step 6: Run Frontend

```bash
cd client
npm install
npm start
```

---

## 🌐 Access Application

Open browser:

```
http://<your-public-ip>:3000
```

---

## 🧪 Testing

Run test cases:

```bash
npm test
```

Example test file:

```
src/AlwaysPass.test.js
```

---

## ⚠️ Common Issues

* Port 3000 not accessible → check security group / firewall
* Backend not connecting → check MySQL credentials
* API not working → verify `.env` file
* Node not found → reload NVM

---

## 🎯 Key Learning

* How frontend, backend, and DB connect
* How Node.js runs backend
* How React consumes APIs
* How environment variables work
* Basic local development workflow

---

This setup is important before moving to:

* Docker
* Kubernetes
* CI/CD

---

## 👍 Next Step

This is a **solid README for `local-dev`**.

Now we can move step-by-step to next branches:

👉 `docker-build-deploy` (very important transition)
👉 then Kubernetes branches

Tell me when you're ready, and I’ll write next README with same quality 🚀


