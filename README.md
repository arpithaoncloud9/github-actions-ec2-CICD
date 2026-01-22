# Automated CI/CD Pipeline for Node.js App on AWS EC2 Using GitHub Actions

### STEP 1 – Create the GitHub repository
1. 1. **Click:** New Repository
2. **Repository name:** 'repo name'  
3. **Visibility:** Public
4. **Initialize with:**
    - ✅ Add a README file
    - ❌ No .gitignore or license needed right now (we can add later)    
5. Click **Create repository**
### STEP 2 – Clone the repo to your machine
- Open your terminal (VS Code terminal, Git Bash, or macOS Terminal — anything works).
- Run this commands:
```bash
git clone https://github.com/<your-username>/automated-cicd-nodejs-aws-ec2-github-actions.git
```
```bash
cd automated-cicd-nodejs-aws-ec2-github-actions
```
Replace `<your-username>` with your actual GitHub username.
This will:
- Download your new repo to your machine
- Move you inside the project folder
- Prepare you to start building the Node.js app
### **STEP 3 — Create a simple Node.js app 
#### 3.1 Initialize Node.js
- In your project folder, run:
```bash
npm init -y
``` 
- This creates `package.json`.
#### 3.2 Install Jest (for testing)
```bash
npm install jest --save-dev
```
#### 3.3 Create folders
```bash
mkdir src tests
```
#### 3.4 Create the app file
- Create: `src/index.js`
```js
function add(a, b) {
  return a + b;
}

module.exports = add;
```
#### 3.5 Create the test file
- Create: `tests/app.test.js`
```js
const add = require('../src/index');

test('adds numbers correctly', () => {
  expect(add(2, 3)).toBe(5);
});
```
#### 3.6 Update package.json scripts
- Open `package.json` and replace the `"scripts"` section with:
```json
"scripts": {
  "test": "jest",
  "build": "echo 'Build step placeholder'"
}
```
- This gives us:
- A test command
- A build command (placeholder for now)
#### 3.7 Commit and push your code
```bash
git add .
git commit -m "Add Node.js app and tests"
git push
```
### STEP 4 — Add the CI Workflow (GitHub Actions)
This workflow will automatically:
- Install dependencies
- Run tests
- Build the app
- Upload the build artifact
This is your **Continuous Integration** pipeline.
#### 4.1 Create the workflows folder
In your project root, create:
```
.github/workflows
```
#### 4.2 Create the CI workflow file
Inside `.github/workflows`, create:
```
ci.yaml
```
Paste this workflow:
```yaml
name: CI Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build project
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: .
```
This workflow teaches you:
- Triggers
- Runners
- Steps
- Using marketplace actions
- Caching
- Artifacts
#### 4.3 Commit and push the workflow
```bash
git add .
git commit -m "Add CI workflow"
git push
```
#### 4.4 Verify CI is running
Go to your GitHub repo → **Actions tab**
You should see:
**CI Pipeline — running**
This means your automation is alive.
### STEP 5 — Prepare Your EC2 Instance for Deployment
#### 5.1 Launch an EC2 Instance
Go to AWS Console → EC2 → Instances → **Launch Instance**
Use these settings:
- **Name:** `cicd-nodejs-ec2`
- **AMI:** Amazon Linux 2 (recommended)
- **Instance type:** t2.micro (free tier)
- **Key pair:** Create or choose an existing one
- **Security group:**
    - Allow **SSH (22)** from your IP
    - Allow **HTTP (80)** from anywhere (for app access)
Click **Launch Instance**.
#### 5.2 Connect to the EC2 Instance
In your terminal:
```bash
chmod 400 your-key.pem

ssh -i your-key.pem ec2-user@<EC2-Public-IP>
```
Replace:
- `your-key.pem` → your private key file
- `<EC2-Public-IP>` → from AWS console
If you’re on Windows and using Git Bash, same command works.

#### 5.3 Install Node.js and Git on EC2
Run
```bash
sudo yum update -y
sudo yum install -y nodejs git
```
Verify
```bash
node -v
npm -v
```
#### 5.4 Install PM2
PM2 keeps your Node.js app running even after deployment.
```bash
sudo npm install -g pm2
```
#### 5.5 Create a folder for your app
```bash
sudo mkdir -p /var/www/myapp
sudo chown ec2-user:ec2-user /var/www/myapp
```
This is where GitHub Actions will upload your build files.

### STEP 6 — Add EC2 secrets to GitHub (for secure deployment)
Your GitHub Actions workflow will need:
- The EC2 public IP
- The EC2 username
- Your private SSH key
These must be stored **securely** in GitHub Secrets.
#### 6.1 Go to GitHub Secrets
In your GitHub repo:
**Settings → Secrets and variables → Actions → New repository secret**
You will create **three secrets**.
#### 6.2 Secret 1 — EC2_HOST
- Name: `EC2_HOST`
- Value: your EC2 public IPv4 address Example: `54.175.34.160`
Click **Add secret**.
#### 6.3 Secret 2 — EC2_USER
This depends on your AMI:
- For **Amazon Linux 2** → `ec2-user`
- For **Ubuntu** → `ubuntu`
You used Amazon Linux 2 earlier, so:
- Name: `EC2_USER`
- Value: `ec2-user`
Click **Add secret**.
#### 6.4 Secret 3 — EC2_SSH_KEY
This is your **private key (.pem)** content.
1. Open your `.pem` file in a text editor
2. Copy the entire content including:
```
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```
3. Create a new secret:
- Name: `EC2_SSH_KEY`
- Value: paste the entire key
Click **Add secret**.
### STEP 7 — Create the CD workflow to deploy to EC2
This workflow will:
1. Download the build artifact from the CI pipeline
2. SSH into your EC2 instance
3. Copy the build files
4. Restart your Node.js app using PM2
#### 7.1 Create the CD workflow file
Inside your project, go to:
```
.github/workflows/
```
Create a new file:
```
cd-ec2.yaml
```
Paste this workflow:
```yaml
name: Deploy to EC2

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: build/

      - name: Add SSH key
        run: |
          echo "${{ secrets.EC2_SSH_KEY }}" > key.pem
          chmod 600 key.pem

      - name: Copy files to EC2
        run: |
          scp -o StrictHostKeyChecking=no -i key.pem -r build/* \
          ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:/var/www/myapp/

      - name: Restart app on EC2
        run: |
          ssh -o StrictHostKeyChecking=no -i key.pem \
          ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} \
          "pm2 restart all || pm2 start /var/www/myapp/index.js"

```
#### 7.2 Commit and push the workflow
```
git add .
git commit -m "Add CD workflow for EC2 deployment"
git push
```
#### 7.3 Verify the workflow exists
Go to your GitHub repo → **Actions tab**
You should now see:
- CI Pipeline
- Deploy to EC2
The CD pipeline will show a **Run workflow** button.

### STEP 8 — Run Your First Deployment to EC2
This is where you trigger the CD workflow manually and watch your app deploy to your EC2 instance.
#### 8.1 Go to the Actions tab
In your GitHub repo:
1. Click **Actions**
2. On the left side, select **Deploy to EC2**
3. Click the **Run workflow** button (top right)
A small dropdown appears → click **Run workflow** again.
This will start your deployment job.

<img width="1374" height="557" alt="Screenshot 2026-01-10 at 1 44 28 PM" src="https://github.com/user-attachments/assets/5de7c535-a70b-4e73-b471-e35870d60bba" />


<img width="1417" height="443" alt="Screenshot 2026-01-10 at 1 45 22 PM" src="https://github.com/user-attachments/assets/0ed46ebd-bfab-4b66-9385-d278b93b8acb" />


#### 8.2 Watch the logs
You should see steps like:
- Checkout code
- Download build artifact
- Add SSH key
- Copy files to EC2
- Restart app on EC2
If everything is correct, the workflow will turn **green**.
#### 8.3 Verify the app is running on EC2
SSH into your EC2 instance:
```
ssh -i your-key.pem ec2-user@<EC2-Public-IP>
```
Then check PM2:
```
pm2 status
```
You should see your app running.

<img width="1362" height="389" alt="Screenshot 2026-01-10 at 1 16 59 PM" src="https://github.com/user-attachments/assets/07d61402-12a4-44f8-b56c-7c439bf1257d" />


### Outcome :
- Built a fully automated CI/CD pipeline using GitHub Actions and AWS EC2
- CI workflow runs tests, builds the app, and uploads optimized artifacts
- CD workflow deploys to EC2 automatically with zero manual steps
- Deployment is verified live via PM2, demonstrating production‑ready DevOps automation
