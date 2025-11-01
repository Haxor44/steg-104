# MEAN Stack Deployment on AWS EC2

This project demonstrates the deployment of a MEAN stack application (MongoDB, Express.js, Angular, Node.js) on AWS EC2 instance.
<img width="1920" height="1080" alt="Screenshot from 2025-11-01 14-38-13" src="https://github.com/user-attachments/assets/03a2e921-3b83-4483-9720-b46f46ba1869" />


## 📋 Prerequisites

Before you begin, ensure you have:
- An AWS account with EC2 access
- Basic knowledge of Linux commands
- SSH client installed on your local machine
- Domain name (optional, for production)

## 🚀 Quick Start

### 1. EC2 Instance Setup

#### Launch EC2 Instance
1. Log into AWS Management Console
2. Navigate to EC2 Dashboard
3. Click "Launch Instance"
4. Configure:
   - **Name**: `mean-stack-server`
   - **AMI**: Ubuntu Server 22.04 LTS
   - **Instance Type**: t2.micro (free tier eligible)
   - **Key Pair**: Create new or select existing
   - **Security Group**: Configure ports (see below)

#### Security Group Configuration
Open the following ports:
- **SSH**: 22
- **HTTP**: 80
- **HTTPS**: 443 (optional)
- **Custom TCP**: 3000 (Node.js app)
- **Custom TCP**: 27017 (MongoDB - restrict to your IP for production)

### 2. Server Configuration

#### Connect to EC2 Instance
```bash
ssh -i "your-key.pem" ubuntu@your-ec2-public-ip
```

#### Update System
```bash
sudo apt update && sudo apt upgrade -y
```

### 3. MongoDB Installation

#### Install MongoDB
```bash
# Import MongoDB public key
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -

# Create list file
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

# Update and install
sudo apt update
sudo apt install -y mongodb-org

# Start and enable MongoDB
sudo systemctl start mongod
sudo systemctl enable mongod
```

#### Secure MongoDB (Production)
```bash
# Connect to MongoDB
mongosh

# Create admin user
use admin
db.createUser({
  user: "admin",
  pwd: "secure-password",
  roles: ["root"]
})

# Exit and enable authentication
exit
```

Edit MongoDB configuration:
```bash
sudo nano /etc/mongod.conf
```

Add/Modify:
```yaml
security:
  authorization: enabled

net:
  bindIp: 127.0.0.1
```

Restart MongoDB:
```bash
sudo systemctl restart mongod
```

### 4. Node.js Installation

#### Install Node.js
```bash
# Install Node.js 18.x
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify installation
node --version
npm --version
```

#### Install PM2 (Process Manager)
```bash
sudo npm install -g pm2
```

### 5. Application Deployment

#### Clone or Upload Your Application
```bash
# Clone from Git
git clone https://github.com/your-username/your-mean-app.git
cd your-mean-app

# Or upload your application files using SCP
# scp -i "your-key.pem" -r ./your-app ubuntu@your-ec2-ip:/home/ubuntu/
```

#### Install Dependencies
```bash
# Backend dependencies
cd backend
npm install

# Frontend dependencies
cd ../frontend
npm install
```

#### Build Angular Application
```bash
npm run build
```

#### Configure Environment Variables
Create `.env` file in backend directory:
```env
NODE_ENV=production
PORT=3000
MONGODB_URI=mongodb://localhost:27017/meanapp
JWT_SECRET=your-jwt-secret
```

### 6. Nginx Configuration (Reverse Proxy)

#### Install Nginx
```bash
sudo apt install nginx -y
```

#### Configure Nginx
```bash
sudo nano /etc/nginx/sites-available/mean-app
```

Add configuration:
```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;

    # Angular frontend
    location / {
        root /home/ubuntu/your-mean-app/frontend/dist/frontend;
        try_files $uri $uri/ /index.html;
    }

    # API proxy
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

#### Enable Site
```bash
sudo ln -s /etc/nginx/sites-available/mean-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 7. Start Application with PM2

#### Create PM2 Ecosystem File
Create `ecosystem.config.js`:
```javascript
module.exports = {
  apps: [{
    name: 'mean-backend',
    script: './backend/server.js',
    instances: 'max',
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'development',
      PORT: 3000
    },
    env_production: {
      NODE_ENV: 'production',
      PORT: 3000
    }
  }]
};
```

#### Start Application
```bash
cd /home/ubuntu/your-mean-app
pm2 start ecosystem.config.js --env production
pm2 startup
pm2 save
```

## 🔧 Application Structure

```
your-mean-app/
├── backend/
│   ├── models/
│   ├── routes/
│   ├── middleware/
│   ├── config/
│   └── server.js
├── frontend/
│   ├── src/
│   ├── dist/
│   └── angular.json
├── ecosystem.config.js
└── README.md
```

## 📝 Environment Configuration

### Backend Environment Variables
```env
NODE_ENV=production
PORT=3000
MONGODB_URI=mongodb://localhost:27017/meanapp
JWT_SECRET=your-secret-key
JWT_EXPIRE=30d
```

### Angular Environment (src/environments/environment.prod.ts)
```typescript
export const environment = {
  production: true,
  apiUrl: 'http://your-domain.com/api'
};
```

## 🛠️ Maintenance

### Common Commands
```bash
# Check application status
pm2 status

# View logs
pm2 logs mean-backend

# Restart application
pm2 restart mean-backend

# MongoDB operations
sudo systemctl status mongod
sudo systemctl restart mongod

# Nginx operations
sudo systemctl status nginx
sudo systemctl restart nginx
```

### Backup MongoDB
```bash
# Create backup
mongodump --uri="mongodb://localhost:27017/meanapp" --out=/home/ubuntu/backup/

# Restore from backup
mongorestore --uri="mongodb://localhost:27017/meanapp" /home/ubuntu/backup/meanapp/
```

## 🔒 Security Considerations

1. **Firewall**: Configure security groups to allow only necessary ports
2. **SSH**: Use key-based authentication, disable root login
3. **MongoDB**: Bind to localhost, enable authentication
4. **Node.js**: Run as non-root user, use environment variables for secrets
5. **SSL**: Install SSL certificate (Let's Encrypt) for HTTPS
6. **Updates**: Regularly update system packages

## 🚨 Troubleshooting

### Common Issues

1. **Application not starting**
   - Check PM2 logs: `pm2 logs`
   - Verify environment variables
   - Check MongoDB connection

2. **Nginx 502 Bad Gateway**
   - Verify Node.js app is running: `pm2 status`
   - Check if port 3000 is accessible

3. **MongoDB connection refused**
   - Check if MongoDB is running: `sudo systemctl status mongod`
   - Verify bind IP in mongod.conf

4. **Angular routing issues**
   - Ensure Nginx try_files directive includes `/index.html`

## 📞 Support

For issues related to:
- AWS EC2: Check AWS documentation and instance status
- MongoDB: Refer to [MongoDB Manual](https://docs.mongodb.com/)
- Node.js: Check [Node.js Documentation](https://nodejs.org/docs/)
- Angular: Refer to [Angular Guide](https://angular.io/guide)

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.
