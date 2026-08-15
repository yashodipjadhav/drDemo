# AWS Deployment Guide for Doctor Appointment Project

This document provides a step-by-step blueprint for deploying the Doctor-Appointment application on Amazon Web Services (AWS). It is designed to host the **Vite/React Frontend**, **Vite/React Admin Panel**, and **Node.js/Express Backend** securely and cost-effectively.

---

## 🏛️ Deployment Architecture Overview

Below is the standard, production-ready architecture pattern for this application. We separate the client-side static files from the server dynamic API to maximize speed, minimize costs, and enable scaling.

```mermaid
graph TD
    subgraph Clients (Users & Admins)
        User[Patient Browser]
        Admin[Admin Browser]
    end

    subgraph AWS Cloud
        subgraph Static Web Hosting
            AmplifyFE[AWS Amplify / S3 + CloudFront <br> (Frontend App)]
            AmplifyAdmin[AWS Amplify / S3 + CloudFront <br> (Admin App)]
        end

        subgraph API Hosting Layer
            LoadBalancer[Application Load Balancer]
            Backend[AWS App Runner / Elastic Beanstalk <br> (Node.js Server)]
        end
    end

    subgraph External Cloud Services
        DB[(MongoDB Atlas <br> Managed Cloud Database)]
        Cloudinary[Cloudinary <br> Image Storage]
        Razorpay[Razorpay <br> Payment Gateway]
        Stripe[Stripe <br> Payment Gateway]
    end

    %% Connections
    User -->|HTTPS Request| AmplifyFE
    Admin -->|HTTPS Request| AmplifyAdmin
    
    AmplifyFE -.->|API Calls / HTTPS| LoadBalancer
    AmplifyAdmin -.->|API Calls / HTTPS| LoadBalancer
    
    LoadBalancer --> Backend
    
    Backend -->|Database Queries| DB
    Backend -->|Image Uploads| Cloudinary
    Backend -->|Payments| Razorpay
    Backend -->|Payments| Stripe
```

---

## 🛠️ Phase 1: Database Setup (MongoDB Atlas)

Since your application uses **MongoDB Atlas** (configured in your backend [mongodb.js](file:///d:/Doctor-Appointment/backend/config/mongodb.js)), you do **not** need to install or run MongoDB on AWS! Instead, keep using MongoDB Atlas and configure network access:

1. **Log in to MongoDB Atlas** and navigate to your cluster.
2. Under **Security**, click **Network Access**.
3. **Add IP Address**:
   - For AWS App Runner or Elastic Beanstalk with dynamic outbound IPs, add `0.0.0.0/0` (Allow Access from Anywhere).
   - *Tip:* If you want strict security, you can provision an AWS NAT Gateway with a Static Elastic IP, and whitelist only that IP in MongoDB Atlas.
4. Copy your MongoDB Connection URI from Atlas (looks like `mongodb+srv://yashodipjadhav17:...`). You will use this in Phase 2.

---

## 🚀 Phase 2: Deploying the Node.js Backend

You have three main options to deploy the Express backend on AWS. We recommend **AWS App Runner** for simplicity, or **AWS Elastic Beanstalk** for classic autoscaling, or **Amazon EC2** for the lowest cost and full control.

### Option A: AWS App Runner (Recommended & Easiest)
AWS App Runner is a fully managed service that takes your Node.js code directly from GitHub, builds it, deploys it with SSL (HTTPS) automatically, and handles scaling.

1. **Prepare your GitHub Repository**:
   - Push your code to GitHub (ensure `node_modules` is ignored by `.gitignore`).
2. **Create service in App Runner**:
   - Open AWS App Runner console -> click **Create Service**.
   - Select **Source code repository** -> connect your GitHub account.
   - Choose your repository and the branch containing the backend.
3. **Configure Build Settings**:
   - **Runtime**: Select `Node.js 18` or `Node.js 20`.
   - **Build command**: `npm install`
   - **Start command**: `npm start`
   - **Port**: `4000` (matches the port in your [server.js](file:///d:/Doctor-Appointment/backend/server.js)).
4. **Configure Environment Variables**:
   Add the following variables inside the AWS App Runner configuration console:
   - `CURRENCY` = `INR`
   - `JWT_SECRET` = `[YourRandomSecretString]`
   - `ADMIN_EMAIL` = `[YourAdminEmail]`
   - `ADMIN_PASSWORD` = `[YourAdminPassword]`
   - `MONGODB_URI` = `[YourMongoDBConnectionURI]`
   - `CLOUDINARY_NAME` = `[YourCloudinaryName]`
   - `CLOUDINARY_API_KEY` = `[YourCloudinaryApiKey]`
   - `CLOUDINARY_SECRET_KEY` = `[YourCloudinarySecretKey]`
   - `RAZORPAY_KEY_ID` = `[YourRazorpayKey]`
   - `RAZORPAY_KEY_SECRET` = `[YourRazorpaySecret]`
   - `STRIPE_SECRET_KEY` = `[YourStripeSecret]`
5. **Deploy**:
   - Click **Create & Deploy**.
   - AWS will provision the service and provide a public URL (e.g., `https://xxxxxx.us-east-1.awsapprunner.com`). 
   - **Copy this URL**; this is your production `VITE_BACKEND_URL` for the frontend and admin panel.

---

### Option B: AWS Elastic Beanstalk (Traditional & Scalable)
Elastic Beanstalk is AWS's platform-as-a-service (PaaS). It automatically provisions EC2 instances, load balancers, and security groups.

1. **Prepare a Zip File**:
   - In your backend directory, create a zip file of all files *except* `node_modules`. Ensure `package.json` is at the root of the zip.
2. **Create application**:
   - Go to AWS Elastic Beanstalk -> **Create Application**.
   - Select **Web server environment**.
3. **Configure Environment**:
   - **Platform**: `Node.js` (choose the Node.js version matching your local version, e.g. Node 18 or 20 on AL2023).
   - **Application Code**: Choose **Upload your code** and select the zip file.
4. **Configure Environment Properties (Environment Variables)**:
   - Go to **Configuration** -> **Software** -> **Environment properties**.
   - Add all env variables (`MONGODB_URI`, `CLOUDINARY_*`, etc.).
   - Make sure to set `PORT = 4000` or leave it empty, as Elastic Beanstalk automatically proxies port `80` to port `8080` or uses standard Node.js routing. Elastic Beanstalk sets the `PORT` env variable automatically.
5. **Create & Launch**:
   - Launch the environment. Beanstalk will deploy it and give you an HTTP link (e.g. `http://doctor-api.env.elasticbeanstalk.com`).
   - *Note:* To use HTTPS (SSL), you will need to map a custom domain to it using AWS Route 53 and provision a Certificate using AWS Certificate Manager (ACM) on a Load Balancer.

---

### Option C: Amazon EC2 (Low Cost, Full Control)
If you want to host the backend on a single virtual machine (like a `t3.micro` which is eligible for Free Tier), you can set up Node.js manually.

1. **Launch an EC2 Instance**:
   - AMI: **Ubuntu Server 22.04 LTS** or **Amazon Linux 2023**.
   - Instance Type: `t2.micro` or `t3.micro`.
   - Security Group Rules: Allow **SSH (Port 22)**, **HTTP (Port 80)**, and **HTTPS (Port 443)**.
2. **SSH into the EC2 Instance**:
   ```bash
   ssh -i your-key.pem ubuntu@your-ec2-ip
   ```
3. **Install Node.js & Git**:
   ```bash
   sudo apt update
   sudo apt install -y curl git
   curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
   sudo apt-get install -y nodejs
   ```
4. **Clone & Setup the Project**:
   ```bash
   git clone <your-repo-url> doctor-appointment
   cd doctor-appointment/backend
   npm install
   ```
5. **Set Environment Variables**:
   Create a `.env` file in the EC2 instance inside the `backend` folder:
   ```bash
   nano .env
   ```
   Paste all production environment variables there (similar to your local [backend/.env](file:///d:/Doctor-Appointment/backend/.env)). Save the file.
6. **Install PM2 (Process Manager)**:
   PM2 runs your Node app in the background and restarts it if the server crashes or reboots.
   ```bash
   sudo npm install -y pm2 -g
   pm2 start server.js --name "doctor-backend"
   pm2 startup systemd
   # Run the command generated by the startup command to configure auto-boot
   pm2 save
   ```
7. **Configure Nginx as a Reverse Proxy**:
   Nginx receives web traffic on port 80 (HTTP) and forwards it to your Node.js app on port 4000.
   ```bash
   sudo apt install -y nginx
   sudo nano /etc/nginx/sites-available/default
   ```
   Replace the `location /` block inside `server` with:
   ```nginx
   location / {
       proxy_pass http://localhost:4000;
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection 'upgrade';
       proxy_set_header Host $host;
       proxy_cache_bypass $http_upgrade;
   }
   ```
   Test and reload Nginx:
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```
8. **Secure with Free SSL (Let's Encrypt)**:
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   sudo certbot --nginx -d your-backend-domain.com
   ```

---

## 🎨 Phase 3: Deploying Frontend & Admin Panels

Since the Frontend and Admin panels are client-side Vite/React apps, they compile into static HTML, CSS, and JS. There is **no Node.js server needed at runtime**.

### Option A: AWS Amplify (Recommended & Easiest)
AWS Amplify connects to your git repository, automatically builds your React project every time you commit, and deploys it to a secure, global CDN.

#### 1. Setup Admin Panel
1. Go to AWS Amplify Console -> click **New App** -> **Host web app**.
2. Select **GitHub** and authorize AWS Amplify.
3. Select your repository and the branch. Set the **monorepo subfolder** to `admin`.
4. Configure Build settings:
   - Amplify will auto-detect Vite. Ensure the configuration looks like this:
     ```yaml
     version: 1
     frontend:
       phases:
         preBuild:
           commands:
             - npm ci
         build:
           commands:
             - npm run build
       artifacts:
         baseDirectory: dist
         files:
           - '**/*'
       cache:
         paths:
           - node_modules/**/*
     ```
5. **Configure Environment Variables**:
   Under the App settings in Amplify, select **Environment Variables** and add:
   - `VITE_BACKEND_URL` = `https://[your-deployed-backend-url]` (obtained in Phase 2)
   - `VITE_CURRENCY` = `₹`
6. Click **Save and Deploy**. Once the build finishes, you will receive a URL like `https://main.xxxx.amplifyapp.com`.

#### 2. Setup Patient Frontend
1. Repeat the same steps as above, but set the **monorepo subfolder** to `frontend`.
2. Environment Variables:
   - `VITE_BACKEND_URL` = `https://[your-deployed-backend-url]`
   - `VITE_RAZORPAY_KEY_ID` = `[YourRazorpayKey]`
3. Deploy. You will receive a separate URL for your patients to access.

---

### Option B: Amazon S3 + CloudFront (Enterprise / Cost-Effective)
For massive traffic with the lowest cost, you can compile the React project locally and host it on an Amazon S3 Bucket behind a CloudFront CDN.

#### Step 1: Build the applications locally
Configure your local environment variables in `frontend/.env` and `admin/.env` with your **live production backend URL** and then compile:
```bash
# In frontend/ directory:
npm run build   # Creates 'dist' folder

# In admin/ directory:
npm run build   # Creates 'dist' folder
```

#### Step 2: Upload to Amazon S3
1. Create an S3 bucket for each project (e.g. `doctor-frontend-bucket` and `doctor-admin-bucket`).
2. Disable "Block all public access" (if hosting directly) or leave it blocked and use CloudFront OAI (Origin Access Control - *Recommended for security*).
3. Upload the contents of your `dist` folder directly to the root of the respective S3 bucket.

#### Step 3: Set up Amazon CloudFront CDN
1. Go to CloudFront console -> click **Create Distribution**.
2. **Origin Domain**: Select your S3 bucket.
3. **Origin access**: Select **Origin access control settings (OAC)** and create a control setting. This prevents users from bypassing CDN and accessing S3 directly.
4. **Viewer Protocol Policy**: Select **Redirect HTTP to HTTPS**.
5. **Default Root Object**: Set to `index.html`.
6. Under **Custom Error Response** (crucial for React Router SPA navigation):
   - Click **Create Error Response**.
   - HTTP Error Code: `404` (and `403`).
   - Customize Error Response: **Yes**.
   - Response Page Path: `/index.html`.
   - HTTP Response Code: `200`.
7. Click **Create Distribution**.

---

## 🔒 Phase 4: Production Security & Settings Checklist

Before launch, verify that all configurations are secure:

### 1. CORS Configuration (Cross-Origin Resource Sharing)
Currently, in your [server.js](file:///d:/Doctor-Appointment/backend/server.js), `app.use(cors())` allows *any* site to call the backend. This is fine during early deployment, but for security, edit your backend to restrict requests only to your frontend and admin URLs:
```javascript
const allowedOrigins = [
  'https://your-frontend-amplify-domain.amplifyapp.com',
  'https://your-admin-amplify-domain.amplifyapp.com',
  'http://localhost:5173' // keep for local development
];

app.use(cors({
  origin: function(origin, callback) {
    if (!origin || allowedOrigins.indexOf(origin) !== -1) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  }
}));
```

### 2. HTTPS (SSL/TLS) Requirement
Browsers prevent mixing secure and insecure content. If your frontend runs on HTTPS (which AWS Amplify and CloudFront do by default), your backend **must** also run on HTTPS (`https://...`).
- **AWS App Runner** handles SSL out of the box.
- **AWS Elastic Beanstalk** requires configuring an Application Load Balancer (ALB) with an ACM Certificate.
- **Amazon EC2** requires setting up `certbot` with Nginx.

### 3. Payment Gateway Webhook URLs
Update your webhooks in Razorpay and Stripe to point to your new live domain:
- **Stripe Webhook**: `https://your-backend-domain.com/api/user/verifyStripe`
- **Razorpay Callback**: Configure Razorpay in your frontend code with the live domain.

---

## 📝 Recommended Path for You

| Component | AWS Hosting Service | Reason |
| :--- | :--- | :--- |
| **Backend API** | **AWS App Runner** | Direct connection to Github, automatic deployment, zero configuration HTTPS. |
| **Frontend App** | **AWS Amplify** | Simple React build pipeline, CD/CD on git push, automatic routing fixes. |
| **Admin App** | **AWS Amplify** | Keeps Admin and Frontend deployment flows aligned and easy to monitor. |
| **Database** | **MongoDB Atlas (Existing)** | No database migration required, keep using current cloud instance. |
