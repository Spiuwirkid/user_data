✅ MODUL 1 – INFRASTRUKTUR CLOUD TERINTEGRASI UNTUK WEB MODERN (VERSI IMPROVED)🎯 TujuanMembangun infrastruktur cloud modern berbasis AWS untuk aplikasi web production-ready. Infrastruktur ini mendemonstrasikan kemampuan utama seperti high availability, auto scaling, monitoring, dan keamanan.🧩 Layanan yang DigunakanBerikut adalah layanan AWS yang digunakan dalam modul ini, dikelompokkan berdasarkan kategori:Compute:EC2Load BalancerAuto ScalingNetworking:VPCSubnetInternet/NAT GatewaySecurity:Certificate ManagerIAMSecurity GroupDatabase:RDSStorage:S3 (untuk static assets / log backup)Monitoring:CloudWatch (untuk monitoring + dashboard)DevOps:Cloud9🔧 Langkah-Langkah Implementasi1️⃣ EC2 Instance (Web Server)Launch EC2 instance (Amazon Linux 2023).Security Group: Izinkan port 22 (SSH), 80 (HTTP), 443 (HTTPS), 3000 (App-JS).SSH ke EC2, lalu instal Nginx + Node.js + ExpressJS:sudo apt update -y
sudo apt install -y nginx git
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo apt install -y nodejs
Deploy aplikasi ExpressJS dengan PM2:sudo npm install -g pm2
pm2 start app.js
pm2 startup
pm2 save
2️⃣ Nginx Reverse ProxyKonfigurasi /etc/nginx/sites-available/default:server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name _;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
Reload Nginx:sudo nginx -t
sudo systemctl reload nginx
Verifikasi / dan /users route bekerja.3️⃣ ExpressJS App + Source Codeapp.jsconst express = require('express');
const mysql = require('mysql2');
const app = express();
const port = 3000;

const connection = mysql.createConnection({
    host: '<RDS_ENDPOINT>',
    user: '<DB_USERNAME>',
    password: '<DB_PASSWORD>',
    database: 'lksdb'
});

connection.connect(err => {
    if (err) {
        console.error('Error connecting to RDS:', err.stack);
        return;
    }
    console.log('Connected to RDS as id ' + connection.threadId);
});

app.get('/', (req, res) => {
    res.send('<h1>Web Service LKS - Success!</h1>');
});

app.get('/users', (req, res) => {
    connection.query('SELECT * FROM users', (err, results) => {
        if (err) {
            console.error('Error fetching data:', err.stack);
            return res.status(500).send('Error fetching data from DB');
        }
        res.json(results);
    });
});

app.listen(port, () => {
    console.log(`App listening at http://localhost:${port}`);
});
package.json{
    "name": "lks-web-app",
    "version": "1.0.0",
    "main": "app.js",
    "dependencies": {
        "express": "^4.18.2",
        "mysql2": "^2.3.3"
    }
}
4️⃣ RDS DatabaseBuat instance RDS MySQL (Multi-AZ enabled).Security Group: Izinkan port 3306 dari EC2 Security Group.Buat database lksdb, table users, dan insert sample data:CREATE DATABASE lksdb;
USE lksdb;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100)
);

INSERT INTO users (name) VALUES ('Alice'), ('Bob'), ('Charlie');
5️⃣ Route 53 + SSL (ACM)Buat public hosted zone di Route 53.Request public SSL certificate di ACM.Validasi certificate.Konfigurasi Nginx untuk HTTPS dengan Certbot / Let's Encrypt (jika menggunakan EC2 SSL):sudo yum install -y certbot python3-certbot-nginx
sudo certbot --nginx
6️⃣ Log Backup ke S3Buat S3 bucket lks-nginx-log-bucket.Attach IAM Role ke EC2 dengan S3 PutObject permission.Konfigurasi cronjob untuk upload Nginx logs harian ke S3:crontab -e
# Baris berikut untuk cronjob
0 0 * * * aws s3 cp /var/log/nginx/access.log s3://lks-nginx-log-bucket/access-$(date +\%F).log
0 0 * * * aws s3 cp /var/log/nginx/error.log s3://lks-nginx-log-bucket/error-$(date +\%F).log
7️⃣ Load Balancer (Improvement)Buat ALB (Application Load Balancer).Target group: EC2 instances (port 3000).Listener: HTTP → HTTPS (jika ACM digunakan).Update DNS A record untuk mengarahkan ke ALB DNS.8️⃣ CloudWatch Dashboard (Improvement)Buat CloudWatch Dashboard.Tambahkan widgets:EC2 CPUUtilizationEC2 NetworkIn / NetworkOutRDS CPUUtilizationRDS DatabaseConnections🚦 Testing ChecklistPastikan semua poin berikut berfungsi dengan baik setelah implementasi:https://yourdomain.com → halaman memuat via HTTPS.https://yourdomain.com/users → mengembalikan JSON dari RDS.Nginx logs berhasil diunggah ke S3 setiap hari.CloudWatch Dashboard menunjukkan metrik live.Catatan Peningkatan (Improvement Notes)Load Balancer → menunjukkan arsitektur yang scalable & resilient.CloudWatch Dashboard → menunjukkan observability & monitoring best practice.Dengan improvement ini, peserta menunjukkan pemahaman yang melebihi basic deployment, sesuai arahan juri LKS bahwa improvisasi mendapatkan nilai tambahan.✅ Modul 1 selesai — siap untuk Studi Kasus 1 di LKS Provinsi 🚀
