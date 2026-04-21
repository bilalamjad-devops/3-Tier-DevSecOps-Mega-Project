

# Local Development Setup 

This branch (local-dev) is used to run the application locally without Docker or Kubernetes. The goal is to understand how frontend, backend, and database work together.





## Architecture Overview

This is a simple 3-tier application:

**Frontend Architecture**

  - Language: JavaScript
  - Library: React.js
  - Runtime Environment: Browser


**Backend Architecture**


- Language: JavaScript
- Framework: Express.js
- Runtime Environment: Node.js


**Database Architecture**

- Database: MySQL
- Backend connects directly to the database
- Frontend NEVER connects directly to database


Important difference:

In React, you call the library. In Express, the framework calls your code (routes, middleware)

---

### Important files: package.json & package-lock.json

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





## Prerequisites

* Linux machine / VM / EC2
* Node.js (version 18 or later is recommended)
* MySQL
* Git


## Steps:

1. Linux machine / VM / EC2
2. Node.js 
3. MySQL
4. Fork and Clone Repo
5. Configure Environment Variables
6. Run Backend
7. Run Frontend
8. Access Application





### Step 1. Linux machine / VM / EC2


### Step 2: Node.js 

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


### Step 3: MySQL

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
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Aditya';
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


### Step 4: Fork and Clone Repo

```bash
git clone https://github.com/bilalamjad-devops/3-Tier-DevSecOps-Mega-Project
cd 3-Tier-DevSecOps-Mega-Project
git checkout local-dev
```



### Step 5: Configure Environment Variables

**Frontend (.env)**

```bash
cd client
vi .env
```

Update:

```
REACT_APP_API=http://<your-public-ip>:5000
```

This connects frontend to backend.


### Step 6: Run Backend

```bash
cd api
npm install
npm start
```

* Installs dependencies → `node_modules/`
* Starts backend server


### Step 7: Run Frontend

```bash
cd client
npm install
npm start
```



### Step 8: Access Application

Open browser:

```
http://<your-public-ip>:3000
```



In the next branch (`docker-build-deploy`), this same setup is containerized using Docker and automated using a CI/CD pipeline.

Commit Date: 21-April-2026
