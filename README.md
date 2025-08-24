Here’s a clean, copy-paste **README.md** you can drop into your repo ✅

---

# Resume Site — Jenkins + Docker (Nginx) on GCP

This repo hosts a simple static site (HTML/CSS/JS) and a Jenkins pipeline that builds and runs it in Docker using **Nginx** on a **GCP VM**.

* **Jenkins UI:** `http://<VM_EXTERNAL_IP>:8080`
* **Site (container):** `http://<VM_EXTERNAL_IP>:8082`

We keep Jenkins on **8080** and publish the site on **8082** to avoid port conflicts.

---

## Table of Contents

1. [Pre-requisites](#pre-requisites)
2. [Install Jenkins (on GCP VM)](#install-jenkins-on-gcp-vm)
3. [Open firewall ports (GCP)](#open-firewall-ports-gcp)
4. [Initial Jenkins login](#initial-jenkins-login)
5. [Repo layout](#repo-layout)
6. [Jenkins Pipeline](#jenkins-pipeline)
7. [How to run](#how-to-run)
8. [Troubleshooting](#troubleshooting)
9. [Useful commands](#useful-commands)
10. [Security notes](#security-notes)
11. [License](#license)

---

## Pre-requisites

* **GCP VM** (Debian/Ubuntu) with an external IP.
* **Jenkins** installed and running (default port 8080).
* **Docker** installed on the VM.
* **GCP firewall rules** allowing:

  * **TCP 8080** (Jenkins UI)
  * **TCP 8082** (site container)
* Let the Jenkins service user access Docker:

  ```bash
  sudo usermod -aG docker jenkins
  sudo systemctl restart docker
  sudo systemctl restart jenkins
  # verify (should NOT show "permission denied")
  sudo -u jenkins -H bash -lc 'groups; docker ps'
  ```

---

## Install Jenkins (on GCP VM)

### 1) Install Java (JDK/JRE 17)

```bash
sudo apt update
sudo apt install -y openjdk-17-jre
java -version
```

### 2) Install Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install -y jenkins
```

> **Note:** On GCP, inbound traffic is blocked by default at the VPC firewall.
> You must explicitly allow **TCP 8080** (for Jenkins) and **TCP 8082** (for the site).

---

## Open firewall ports (GCP)

* In the Console: **VPC network → Firewall → Create firewall rule**

  * **Direction:** Ingress, **Action:** Allow
  * **Targets:** Your VM (via network tag or “All instances in the network”)
  * **Source IPv4 ranges:** `0.0.0.0/0` *(or restrict to your IP)*
  * **Protocols/ports:** `tcp:8080` and another rule for `tcp:8082`

---

## Initial Jenkins login

1. Open `http://<VM_EXTERNAL_IP>:8080`
2. Get the initial admin password:

   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Paste it into the Jenkins setup page, install recommended plugins, create your admin user.

---

## Repo layout

Place your static files at the repo root:

```
.
├── Resume.html          # or index.html
├── styles.css
├── script.js
└── README.md
```

If your main page is **`Resume.html`** (not `index.html`), the pipeline automatically copies it to `index.html` so Nginx serves it by default.

---

## Jenkins Pipeline

> **Option A (recommended):** Save the following as `Jenkinsfile` in your repo and configure the job as **“Pipeline script from SCM.”**
> **Option B:** Paste the script directly into the job’s **Pipeline script** field.

```groovy
pipeline {
  agent any
  options { timeout(time: 15, unit: 'MINUTES') }

  environment {
    // repo
    GIT_REPO       = 'https://github.com/Akhil-Jonnalagadda/MY-Jenkins-Project'
    // docker
    IMAGE_NAME     = 'resume-site'
    CONTAINER_NAME = 'resume-8082'
    HOST_PORT      = '8082'   // Jenkins uses 8080; serve site on 8082
  }

  stages {
    stage('Preflight') {
      steps {
        sh '''
          set -e
          echo "== whoami & groups =="; whoami; groups
          echo "== docker version =="; docker version
        '''
      }
    }

    stage('Checkout Code') {
      steps {
        // If using "Pipeline from SCM", you can remove this line.
        git branch: 'main', url: "${GIT_REPO}"
        sh 'ls -la'
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          set -e
          rm -rf docker_resume && mkdir docker_resume

          # Copy site assets from repo root
          cp -r *.html *.css *.js docker_resume/ 2>/dev/null || true

          # Ensure nginx default page exists
          if [ -f docker_resume/Resume.html ] && [ ! -f docker_resume/index.html ]; then
            cp docker_resume/Resume.html docker_resume/index.html
          fi

          # Minimal Dockerfile
          cat > docker_resume/Dockerfile <<'EOF'
FROM nginx:alpine
COPY ./ /usr/share/nginx/html/
EXPOSE 80
EOF

          cd docker_resume
          docker build -t ${IMAGE_NAME}:latest .
        '''
      }
    }

    stage('Run Container on 8082') {
      steps {
        sh '''
          set -e
          docker rm -f ${CONTAINER_NAME} || true
          docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:80 ${IMAGE_NAME}:latest
          docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

          # quick local health check
          for i in $(seq 1 15); do
            curl -sSf http://127.0.0.1:${HOST_PORT} >/dev/null && break || sleep 1
          done
        '''
      }
    }

    stage('Site URL') {
      steps {
        sh '''
          EXT_IP=$(curl -s -H "Metadata-Flavor: Google" \
            http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip || true)
          echo "Your site is available at: http://${EXT_IP:-<YOUR_VM_EXTERNAL_IP>}:${HOST_PORT}"
        '''
      }
    }
  }

  post {
    always { echo 'Pipeline finished.' }
  }
}
```

---

## How to run

1. Confirm firewall allows **tcp:8082** to your VM.
2. Ensure Jenkins can use Docker (see Pre-requisites).
3. In Jenkins:

   * **New Item → Pipeline** (e.g., `resume`)
   * Use **Pipeline script from SCM** (with `Jenkinsfile` in this repo) **or** paste the script above.
   * **Save → Build Now**
4. After the **Run Container** stage completes, open:
   `http://<VM_EXTERNAL_IP>:8082`

---

## Troubleshooting

**Docker permission denied** in Preflight/Build/Run:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart docker
sudo systemctl restart jenkins
sudo -u jenkins -H bash -lc 'groups; docker ps'
```

**Connection refused on 8082**:

```bash
# is anything listening?
sudo ss -lntp | grep ':8082' || echo "NOT LISTENING"
docker ps --format 'table {{.Names}}\t{{.Ports}}'   # expect 0.0.0.0:8082->80/tcp
```

If it works locally but not from internet, ensure the **firewall rule targets this VM** (tag or “All instances”).

**Blank/403 at `/` but `/Resume.html` works**:
No `index.html` in the image. The pipeline copies `Resume.html` → `index.html` automatically—rebuild and re-run.

**Free the port / clean old containers**:

```bash
docker rm -f resume-8082 2>/dev/null || true
docker container prune -f
docker image prune -f
```

---

## Useful commands

Show VM external IP (from inside the VM):

```bash
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip
```

Show listeners:

```bash
sudo ss -lntp | egrep ':8080|:8082' || true
```

---

## Security notes

* Avoid broad `0.0.0.0/0` where possible—prefer **network tags** and restrict source IPs.
* Add TLS/HTTPS (e.g., via a reverse proxy) for public exposure.

---

## License

MIT (or your preferred license)

---

Below are my Inages how the application is Running.
<img width="1914" height="845" alt="image" src="https://github.com/user-attachments/assets/e2190559-dfc3-4dd8-a058-f548d35c2e81" />
<img width="1157" height="876" alt="image" src="https://github.com/user-attachments/assets/0cb87363-ce62-4c9e-ae8c-efa4e70e7419" />
<img width="1916" height="851" alt="image" src="https://github.com/user-attachments/assets/feb5ed80-2ddc-482b-bc2c-d0a2b8f2f97e" />
<img width="1917" height="903" alt="image" src="https://github.com/user-attachments/assets/407b1b07-b02a-4d63-8d2e-2d2f3e42b9c6" />
<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/c823ad50-6d58-47c7-b258-69136a9db72b" />
<img width="1913" height="908" alt="image" src="https://github.com/user-attachments/assets/80521036-87e7-4577-ad0f-ed6ca6708735" />
<img width="1919" height="847" alt="image" src="https://github.com/user-attachments/assets/165a9cee-8161-462c-9791-74f896f7d73d" />
<img width="1911" height="842" alt="image" src="https://github.com/user-attachments/assets/521e32c9-6a38-44cc-ba68-c800547dca17" />
<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/16a5e22f-280a-4bc0-859c-ebe519ad391c" />

<img width="1916" height="843" alt="image" src="https://github.com/user-attachments/assets/e01d44b0-d649-4cea-a3e7-1bfb16c19bce" />





