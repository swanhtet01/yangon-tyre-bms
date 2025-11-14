# Yangon Tyre Factory - AWS Deployment Instructions

## Deployment Target
- **Domain**: ytf.supermega.dev
- **AWS Instance**: SuperMega-Production (t3.small) or Company-HQ-Final (t3.medium)
- **Instance ID**: i-0fa6ccb21590144ff (SuperMega-Production)
- **Region**: us-east-1

## Prerequisites Completed
✅ Application built successfully (dist/ folder ready)
✅ Database schema ready (21 tables)
✅ Test users created
✅ Environment variables configured
✅ AWS credentials configured

## Deployment Steps for GitHub Copilot Agents

### 1. Start EC2 Instance
```bash
aws ec2 start-instances --instance-ids i-0fa6ccb21590144ff
aws ec2 wait instance-running --instance-ids i-0fa6ccb21590144ff
```

### 2. Get Instance Public IP
```bash
aws ec2 describe-instances --instance-ids i-0fa6ccb21590144ff \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text
```

### 3. Install Dependencies on EC2
```bash
# SSH into instance
# Install Node.js 22.x
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install MySQL
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server

# Install PM2 for process management
sudo npm install -g pm2
```

### 4. Deploy Application
```bash
# Create app directory
sudo mkdir -p /var/www/yangon-tyre-bms
sudo chown -R ubuntu:ubuntu /var/www/yangon-tyre-bms

# Copy files (use scp or git clone)
cd /var/www/yangon-tyre-bms
git clone https://github.com/swanhtet01/yangon-tyre-bms.git .

# Install dependencies
npm install --production

# Copy environment file
cp .env.production .env

# Setup database
sudo mysql -e "CREATE DATABASE IF NOT EXISTS yangon_tyre_bms;"
# Apply migrations from drizzle/*.sql files

# Start with PM2
pm2 start dist/index.js --name yangon-tyre-bms
pm2 save
pm2 startup
```

### 5. Configure Nginx Reverse Proxy
```bash
sudo apt-get install -y nginx

# Create nginx config
sudo tee /etc/nginx/sites-available/ytf.supermega.dev << 'EOF'
server {
    listen 80;
    server_name ytf.supermega.dev;
    
    location / {
        proxy_pass http://localhost:3004;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

# Enable site
sudo ln -s /etc/nginx/sites-available/ytf.supermega.dev /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 6. Configure DNS
Update DNS A record for ytf.supermega.dev to point to the EC2 instance public IP.

### 7. Setup SSL (Optional but Recommended)
```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d ytf.supermega.dev
```

## Application Details
- **Port**: 3004
- **Database**: MySQL (yangon_tyre_bms)
- **Test Accounts**: supervisor, manager, executive, admin
- **Password**: test123

## Post-Deployment Verification
1. Check PM2 status: `pm2 status`
2. Check logs: `pm2 logs yangon-tyre-bms`
3. Test endpoint: `curl http://localhost:3004`
4. Access via domain: https://ytf.supermega.dev

## GitHub Repository
https://github.com/swanhtet01/yangon-tyre-bms

## Contact
For issues, check logs and AWS CloudWatch.
