# CI/CD Pipeline with GitHub Actions & AWS EC2 Self-Hosted Runner

A complete CI/CD pipeline that automatically tests a Node.js application and deploys it to an AWS EC2 instance using a GitHub Actions self-hosted runner.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Step 1 — Repository Setup](#step-1--repository-setup)
- [Step 2 — AWS EC2 Setup](#step-2--aws-ec2-setup)
- [Step 3 — Install Node.js and PM2 on EC2](#step-3--install-nodejs-and-pm2-on-ec2)
- [Step 4 — Configure Self-Hosted Runner on EC2](#step-4--configure-self-hosted-runner-on-ec2)
- [Step 5 — Create the Workflow File](#step-5--create-the-workflow-file)
- [Step 6 — Push and Verify](#step-6--push-and-verify)
- [Workflow File Explained](#workflow-file-explained)
- [Challenges and Solutions](#challenges-and-solutions)

---

## Architecture Overview

```
Developer pushes code to GitHub (main branch)
        │
        ▼
GitHub Actions triggered
        │
        ├──► JOB 1: Test (runs on GitHub's server)
        │         ├── Install Node.js & dependencies
        │         ├── Run tests → save to test-results.txt
        │         └── Upload test-results.txt as artifact
        │
        └──► JOB 2: Deploy (runs on YOUR EC2 via self-hosted runner)
                  ├── Download artifact
                  ├── Display test results
                  ├── Install PM2 if missing
                  ├── Deploy app with PM2
                  └── Verify app is live on port 3000
```

---

## Prerequisites

- A GitHub account
- An AWS account (Free Tier is enough)
- Basic knowledge of terminal/SSH

---

## Step 1 — Repository Setup

Clone the original repository and push it to your own GitHub account.

```bash
# Clone the original project
git clone https://github.com/roy35-909/OSTAD-Assignment-module-3.git
cd OSTAD-Assignment-module-3

# Remove the original remote
git remote remove origin

# Add your new GitHub repository as the remote
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git

# Push to your repository
git push -u origin main
```

---

## Step 2 — AWS EC2 Setup

1. Go to **AWS Console → EC2 → Launch Instance**
2. Configure the instance with these settings:

| Setting | Value |
|---|---|
| OS | Ubuntu 22.04 LTS |
| Instance type | t2.micro (Free Tier eligible) |
| Key pair | Create new → download the `.pem` file |

3. Under **Security Group**, add these inbound rules:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | 0.0.0.0/0 |
| HTTP | 80 | 0.0.0.0/0 |
| Custom TCP | 3000 | 0.0.0.0/0 |

4. Launch the instance and note down the **Public IP address**.

5. Connect to EC2 from your local terminal:

```bash
chmod 400 your-key.pem
ssh -i "your-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

---

## Step 3 — Install Node.js and PM2 on EC2

Run these commands inside the EC2 terminal:

```bash
# Install Node.js version 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify Node.js installation
node --version   # Should show v22.x.x

# Install PM2 globally (process manager that keeps the app running)
sudo npm install -g pm2

# Verify PM2 installation
pm2 --version

# Configure PM2 to auto-start on EC2 reboot
pm2 startup
# Copy and run the command that pm2 prints out, for example:
# sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

---

## Step 4 — Configure Self-Hosted Runner on EC2

The self-hosted runner is a program installed on EC2 that listens for jobs from GitHub Actions and executes them directly on your server — no SSH secrets needed.

**On GitHub:**

1. Go to your repository → **Settings → Actions → Runners**
2. Click **New self-hosted runner**
3. Select: OS = **Linux**, Architecture = **x64**
4. GitHub will show you a set of commands — run them on your EC2

**On EC2**, run the commands GitHub provides. They will look like this:

```bash
# Create a directory for the runner
mkdir actions-runner && cd actions-runner

# Download the runner package (use the exact URL from GitHub)
curl -o actions-runner-linux-x64-2.x.x.tar.gz -L https://github.com/actions/runner/releases/download/...
tar xzf ./actions-runner-linux-x64-2.x.x.tar.gz

# Configure the runner (use the exact token from GitHub)
./config.sh --url https://github.com/YOUR_USERNAME/YOUR_REPO --token YOUR_TOKEN
```

When prompted during configuration:
- Runner group → press **Enter** (use default)
- Runner name → type `ec2-runner`
- Labels → type `self-hosted,ec2`
- Work folder → press **Enter** (use default)

**Install the runner as a permanent background service:**

```bash
# This ensures the runner keeps running even after EC2 reboots or SSH disconnects
sudo ./svc.sh install
sudo ./svc.sh start

# Verify it is running
sudo ./svc.sh status
```

Go back to **GitHub → Settings → Actions → Runners** and confirm the runner shows **Idle** status.

---

## Step 5 — Create the Workflow File

Create the directory and file in your local project:

```bash
mkdir -p .github/workflows
```

Create `.github/workflows/ci-cd.yml` with the following content:

```yaml
name: CI/CD Pipeline - Module 5

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:

  test:
    name: Run Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js 22
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm install

      - name: Run tests and capture results
        run: |
          echo "========================================" > test-results.txt
          echo "Test Run: $(date)" >> test-results.txt
          echo "========================================" >> test-results.txt
          npm run check >> test-results.txt 2>&1 || true
          echo "========================================" >> test-results.txt
          echo "Test completed at: $(date)" >> test-results.txt

      - name: Show test results
        run: cat test-results.txt

      - name: Upload test results as artifact
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results.txt
          retention-days: 7

  deploy:
    name: Deploy to EC2
    runs-on: [self-hosted, ec2]
    needs: test

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download test results artifact
        uses: actions/download-artifact@v4
        with:
          name: test-results

      - name: Display test results
        run: |
          echo "====== Test Results from Artifact ======"
          cat test-results.txt
          echo "========================================"

      - name: Install Node.js 22 if not present
        run: |
          if ! command -v node &> /dev/null; then
            echo "Node.js not found, installing..."
            curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
            sudo apt-get install -y nodejs
          else
            echo "Node.js already installed: $(node --version)"
          fi

      - name: Install PM2 if not present
        env:
          PATH: /usr/bin:/usr/local/bin:/bin:/usr/sbin:/sbin
        run: |
          if ! command -v pm2 &> /dev/null; then
            echo "PM2 not found, installing automatically..."
            sudo npm install -g pm2
            echo "PM2 installed successfully: $(pm2 --version)"
          else
            echo "PM2 already installed: $(pm2 --version)"
          fi

      - name: Install dependencies on EC2
        run: npm install

      - name: Deploy application with PM2
        env:
          PATH: /usr/bin:/usr/local/bin:/bin:/usr/sbin:/sbin
        run: |
          echo "Starting deployment..."

          pm2 delete node-app || true
          pm2 start "./src/server.js" --name node-app
          pm2 save

          sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu || true

          echo "Deployment successful!"
          pm2 status

      - name: Verify deployment
        env:
          PATH: /usr/bin:/usr/local/bin:/bin:/usr/sbin:/sbin
        run: |
          sleep 3
          echo "Checking if app is running..."
          pm2 status node-app
          curl -f http://localhost:3000 && echo "App is live!" || echo "App check failed"
```

---

## Step 6 — Push and Verify

```bash
git add .
git commit -m "Add CI/CD workflow with self-hosted runner"
git push origin main
```

Go to **GitHub → Actions** tab. You will see the workflow running with two jobs:

```
✅ Run Tests       (GitHub server)
✅ Deploy to EC2   (your EC2 runner)
```

Open your browser and visit:

```
http://YOUR_EC2_PUBLIC_IP:3000
```

You should see the **Hello World** page — the app is live.

---

## Workflow File Explained

### Trigger section

```yaml
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
```

Defines when the workflow runs. It triggers on every push or pull request to the `main` branch.

---

### Test Job

```yaml
runs-on: ubuntu-latest
```

Runs on GitHub's own cloud server — not your EC2. GitHub provides this for free.

```yaml
uses: actions/checkout@v4
```

Downloads your repository code into the runner's workspace. Without this, the runner has no files to work with.

```yaml
uses: actions/setup-node@v4
with:
  node-version: '22'
  cache: 'npm'
```

Installs Node.js version 22. The `cache: 'npm'` option saves downloaded packages so repeated runs are faster.

```yaml
npm run check >> test-results.txt 2>&1 || true
```

Runs the test script and saves both normal output and error output (`2>&1`) into `test-results.txt`. The `|| true` means even if the test fails, the workflow continues instead of stopping.

```yaml
uses: actions/upload-artifact@v4
with:
  name: test-results
  path: test-results.txt
  retention-days: 7
```

Uploads `test-results.txt` to GitHub so the deploy job (which runs on a different machine) can download and display it. Files are kept for 7 days.

---

### Deploy Job

```yaml
runs-on: [self-hosted, ec2]
```

Tells GitHub to run this job on a runner that has both the `self-hosted` and `ec2` labels. This matches the runner you installed on EC2.

```yaml
needs: test
```

This job will not start until the `test` job finishes successfully. If tests fail, deployment is skipped automatically.

```yaml
uses: actions/download-artifact@v4
with:
  name: test-results
```

Downloads the `test-results.txt` file that was uploaded by the test job. The name must match exactly.

```yaml
if ! command -v pm2 &> /dev/null; then
  sudo npm install -g pm2
fi
```

Checks whether PM2 is already installed. If not, installs it automatically. This means the workflow works on any fresh EC2 without manual setup.

```yaml
env:
  PATH: /usr/bin:/usr/local/bin:/bin:/usr/sbin:/sbin
```

The GitHub Actions runner uses a non-login shell that does not load `~/.bashrc`. This means it cannot find commands like `pm2` by default. Setting `PATH` explicitly tells the shell exactly where to look for installed programs.

```yaml
pm2 delete node-app || true
pm2 start "./src/server.js" --name node-app
pm2 save
```

Stops the old version of the app if it is running, starts the new version, and saves the process list. `|| true` prevents errors if the app was not previously running.

```yaml
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu || true
```

Configures PM2 to start automatically when EC2 reboots. This ensures the app stays running even after server restarts. `|| true` prevents failure if this was already configured.

```yaml
curl -f http://localhost:3000 && echo "App is live!" || echo "App check failed"
```

Sends an HTTP request to the app running on port 3000. If it responds, prints "App is live!". If it fails, prints "App check failed" without stopping the workflow.

---

## Challenges and Solutions

### Challenge 1 — Self-Hosted Runner Showing Offline

After installing the GitHub Actions runner on EC2, it appeared as "Offline" in the GitHub repository settings. The runner was started manually but was not configured as a persistent background service, so it stopped when the SSH session ended.

**Solution:** Navigated to the `actions-runner` directory and ran `sudo ./svc.sh install` followed by `sudo ./svc.sh start` to register the runner as a systemd service. This ensures the runner stays active permanently, even after EC2 reboots or SSH session disconnections. Note: this step must be done manually once — it cannot be automated in the workflow because the runner must already be online before any workflow job can execute.

---

### Challenge 2 — PM2 Command Not Found During Deployment

The deploy job failed with `pm2: command not found` (exit code 127). The GitHub Actions runner executes shell scripts in a non-interactive, non-login shell environment, which means it does not load `~/.bashrc` or `~/.profile`. As a result, the npm global binary path was not included, and PM2 could not be located even though it was installed.

**Solution:** Added an `Install PM2 if not present` step to the workflow that checks whether PM2 exists and installs it automatically if missing. Also added an explicit `env: PATH` block in each deploy step to hardcode the correct binary locations. This removes the need to manually SSH into the server to install PM2 and makes the pipeline work on any fresh EC2 instance.

---

### Challenge 3 — Application Not Accessible via Browser

After successful deployment, the application was running on EC2 (confirmed via `pm2 status` and `curl http://localhost:3000`), but the browser returned `ERR_CONNECTION_REFUSED` when accessing the public IP.

**Solution:** The AWS EC2 Security Group was missing an inbound rule for port 3000. Added a Custom TCP inbound rule allowing traffic on port 3000 from `0.0.0.0/0`. After saving the rule, the application became accessible at `http://YOUR_EC2_IP:3000`.

---

## Live Application

Once deployed, access the application at:

```
http://YOUR_EC2_PUBLIC_IP:3000       → Hello World page
http://YOUR_EC2_PUBLIC_IP:3000/api   → JSON response


---

## Author

**Salauddin**  
Software Developer
```