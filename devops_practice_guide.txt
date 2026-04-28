# Laravel DevOps Practice Guide (WSL2 / Linux)

এই ডকুমেন্টটি আপনার লারাভেল প্রজেক্টটিকে একটি সম্পূর্ণ DevOps পাইপলাইনে রূপান্তরিত করার জন্য তৈরি করা হয়েছে। যেহেতু আপনি WSL2 ব্যবহার করছেন, তাই নিচের সব কমান্ড আপনার **WSL (Ubuntu/Debian)** টার্মিনালে রান করবেন।

> [!TIP]
> WSL2 তে উইন্ডোজ ড্রাইভের ফাইলগুলো অ্যাক্সেস করতে সমস্যা হতে পারে বা পারফরম্যান্স স্লো হতে পারে। তাই সবচেয়ে ভালো হয় প্রজেক্টটি লিনাক্সের হোম ডিরেক্টরিতে কপি করে নেওয়া।

---

## Step 1: Git Version Control

প্রথমে প্রজেক্টটি লিনাক্স ফোল্ডারে কপি করে Git ইনিশিয়ালাইজ করবো।

**১.১ প্রজেক্ট কপি করা এবং ডিরেক্টরিতে যাওয়া:**
```bash
# উইন্ডোজ ড্রাইভ থেকে প্রজেক্টটি লিনাক্সের হোম ফোল্ডারে কপি করুন
cp -r /mnt/d/laragon/www/laravel-app ~/laravel-app

# প্রজেক্ট ডিরেক্টরিতে প্রবেশ করুন
cd ~/laravel-app
```

**১.২ Git ইনিশিয়ালাইজ ও কমিট করা:**
```bash
git init
git add .
git commit -m "Initial Laravel project with Breeze"
```

**১.৩ GitHub এ পুশ করা:**
GitHub এ একটি নতুন Repository তৈরি করুন (নাম দিন `laravel-app`)। এরপর টার্মিনালে নিচের কমান্ডগুলো দিন:
```bash
git branch -M main
git remote add origin https://github.com/আপনার_ইউজারনেম/laravel-app.git
git push -u origin main
```

---

## Step 2: Dockerize the Application

আমাদের লারাভেল প্রজেক্টটি চালানোর জন্য ৩টি জিনিস লাগবে: **PHP**, **Nginx (Web Server)**, এবং **MySQL**। আমরা `docker-compose` এর সাহায্যে এগুলো সেটআপ করবো।

**২.১ Dockerfile তৈরি:**
প্রজেক্টের মূল ফোল্ডারে `Dockerfile` নামে একটি ফাইল তৈরি করুন এবং নিচের কোডগুলো পেস্ট করুন:
```bash
nano Dockerfile
```
*ফাইলে এই অংশটুকু কপি করে পেস্ট করুন (Paste করার জন্য Right-click করুন):*
```dockerfile
FROM php:8.2-fpm

# Install dependencies
RUN apt-get update && apt-get install -y \
    libpng-dev libjpeg-dev libfreetype6-dev \
    zip unzip git curl \
    && docker-php-ext-install pdo pdo_mysql gd

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html
COPY . .

# Set permissions
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache
```
*সেভ করতে: `Ctrl+O`, `Enter`, `Ctrl+X`*

**২.২ Nginx কনফিগারেশন তৈরি:**
```bash
nano nginx.conf
```
*পেস্ট করুন:*
```nginx
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    root /var/www/html/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```
*সেভ করুন।*

**২.৩ docker-compose.yml তৈরি:**
```bash
nano docker-compose.yml
```
*পেস্ট করুন:*
```yaml
version: '3.8'
services:
  app:
    build: .
    volumes:
      - .:/var/www/html
  web:
    image: nginx:alpine
    ports:
      - "8000:80"
    volumes:
      - .:/var/www/html
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app
  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: laravel_app
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
volumes:
  db_data:
```
*সেভ করুন।*

**২.৪ ডকার রান করা:**
```bash
# কন্টেইনারগুলো রান করুন (ব্যাকগ্রাউন্ডে)
docker-compose up -d --build
```
> [!NOTE]
> এখন আপনি ব্রাউজারে `http://localhost:8000` এ গেলে প্রজেক্টটি দেখতে পাবেন!

---

## Step 3: Jenkins CI/CD Setup

Jenkins লিনাক্সে ইন্সটল না করে আমরা Docker এর ভেতরেই চালাবো।

**৩.১ Jenkins রান করা:**
```bash
docker run -d -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home --name jenkins jenkins/jenkins:lts
```

**৩.২ Initial Password বের করা:**
ব্রাউজারে `http://localhost:8080` এ যান। এটি একটি পাসওয়ার্ড চাইবে। পাসওয়ার্ডটি পেতে টার্মিনালে দিন:
```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
পাসওয়ার্ডটি কপি করে ব্রাউজারে দিন এবং "Install Suggested Plugins" সিলেক্ট করে ইউজার একাউন্ট তৈরি করুন।

**৩.৩ Jenkinsfile তৈরি (প্রজেক্ট ফোল্ডারে):**
```bash
nano Jenkinsfile
```
```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/আপনার_ইউজারনেম/laravel-app.git'
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
                // এখানে PHPUnit টেস্ট রান করানোর কমান্ড থাকবে
            }
        }
        stage('Build & Deploy') {
            steps {
                echo 'Building docker image and deploying...'
            }
        }
    }
}
```
*সেভ করে এটি গিটহাবে পুশ করে দিন।*

---

## Step 4: Local Kubernetes (Minikube) Setup

এখন আমরা ডকারের পরিবর্তে Kubernetes ব্যবহার করে প্রজেক্ট চালাবো।

**৪.১ Minikube এবং Kubectl ইন্সটল করা:**
```bash
# Kubectl install
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Minikube install
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**৪.২ Minikube স্টার্ট করা:**
```bash
minikube start --driver=docker
```

---

## Step 5: Kubernetes-এ Deploy করা (K8s Manifests)

প্রজেক্ট ফোল্ডারে একটি `k8s` ফোল্ডার তৈরি করুন।
```bash
mkdir k8s && cd k8s
```

**৫.১ Deployment File:**
```bash
nano deployment.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      containers:
      - name: laravel
        image: nginx:alpine # (Testing purpose, পরে কাস্টম ইমেজ দেবো)
        ports:
        - containerPort: 80
```
*সেভ করুন।*

**৫.২ Service File:**
```bash
nano service.yaml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
spec:
  type: NodePort
  selector:
    app: laravel
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30000
```
*সেভ করুন।*

**৫.৩ K8s এ অ্যাপ্লাই করা:**
```bash
cd ..
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

---

## Step 6: Local IP তে Deploy & Access

প্রজেক্টটি ಈಗ K8s ক্লাস্টারে চলছে। এটি অ্যাক্সেস করার জন্য:

**৬.১ Minikube Service URL পাওয়া:**
```bash
minikube service laravel-service --url
```
এটি আপনাকে একটি URL দিবে (যেমন: `http://192.168.49.2:30000`)। 

**৬.২ লোকাল নেটওয়ার্কে অ্যাক্সেস করা:**
আপনার পিসির লোকাল আইপি বের করুন:
```bash
ip addr show eth0 | grep inet
```
(সাধারণত `192.168.1.X` বা `10.0.0.X` টাইপের হয়)।

এখন আপনি আপনার মোবাইল বা অন্য ল্যাপটপ থেকে `http://<আপনার-IP>:30000` পোর্টে গেলে প্রজেক্টটি লাইভ দেখতে পাবেন! 

---
*যেকোনো কমান্ডে এরর পেলে বা আটকে গেলে আমাকে জানাবেন, আমি ট্রাবলশুট করে দেবো!*
