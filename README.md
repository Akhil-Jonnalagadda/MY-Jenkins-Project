Install Jenkins.
Pre-Requisites:

Java (JDK)
Run the below commands to install Java and Jenkins
Install Java

sudo apt update
sudo apt install openjdk-17-jre
Verify Java is Installed

java -version
Now, you can proceed with installing Jenkins

curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
**Note: ** By default, Jenkins will not be accessible to the external world due to the inbound traffic restriction by AWS. Open port 8080 in the inbound traffic rules as show below.

EC2 > Instances > Click on
In the bottom tabs -> Click on Security
Security groups
Add inbound traffic rules as shown in the image (you can just allow TCP 8080 as well, in my case, I allowed All traffic).
Screenshot 2023-02-01 at 12 42 01 PM

Login to Jenkins using the below URL:
http://:8080 [You can get the Compute Engine instance-public-ip-address from your GCP console page]

After you login to Jenkins, - Run the command to copy the Jenkins Admin Password - sudo cat /var/lib/jenkins/secrets/initialAdminPassword - Enter the Administrator password


Here’s a ready-to-drop-in **README.md** for your repo. It documents the exact setup you just built: Jenkins (on 8080) builds a Docker image that serves your HTML via Nginx and publishes it on **8082**.

---

# Resume Site — Jenkins + Docker (Nginx) on GCP

This repo hosts a simple static site (HTML/CSS/JS) and a Jenkins pipeline that builds and runs it in Docker using **Nginx** on a **GCP VM**.

* **Jenkins UI:** `http://<VM_EXTERNAL_IP>:8080`
* **Site (container):** `http://<VM_EXTERNAL_IP>:8082`

> We keep Jenkins on 8080 and publish the site on **8082** to avoid port conflicts.

---

## Prerequisites

* **GCP VM** (Debian/Ubuntu) with an external IP.
* **Jenkins** installed and running (default port 8080).
* **Docker** installed on the VM.
* **Firewall rules** (VPC → Firewall):

  * Allow **TCP 8080** (Jenkins UI).
  * Allow **TCP 8082** (site). Target your VM (via tag or “all instances”).
* Let the Jenkins service user access Docker:

```bash
# run once on the VM
sudo usermod -aG docker jenkins
sudo systemctl restart docker
sudo systemctl restart jenkins
# verify (should NOT say "permission denied")
sudo -u jenkins -H bash -lc 'groups; docker ps'
```

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

If your main page is `Resume.html` (not `index.html`), the pipeline auto-copies it to `index.html` so Nginx serves it by default.

---

## Jenkins pipeline (Declarative)

> **Option A (recommended):** Put this file in your repo as `Jenkinsfile` and configure the job as “Pipeline script from SCM”.
> **Option B:** Paste the script directly into the job’s “Pipeline script” field.

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
          echo "== whoami & groups =="
          whoami; groups
          echo "== docker version =="
          docker version
        '''
      }
    }

    stage('Checkout Code') {
      steps {
        // If using "Pipeline from SCM", you can remove this step.
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
2. Make sure Jenkins can use Docker (see prerequisites).
3. In Jenkins:

   * New Item → **Pipeline** (e.g., `resume`)
   * Either:

     * **Pipeline script from SCM** (points to this repo with the `Jenkinsfile`), or
     * Paste the pipeline script above into “Pipeline script”
   * Save → **Build Now**
4. When the “Run Container” stage completes, open:
   `http://<VM_EXTERNAL_IP>:8082`

---

## Troubleshooting

* **Permission denied to Docker** in Preflight/Build/Run:

  ```bash
  sudo usermod -aG docker jenkins
  sudo systemctl restart docker
  sudo systemctl restart jenkins
  sudo -u jenkins -H bash -lc 'groups; docker ps'
  ```

* **Connection refused on 8082:**

  * Check something is listening:

    ```bash
    sudo ss -lntp | grep ':8082' || echo "NOT LISTENING"
    docker ps --format 'table {{.Names}}\t{{.Ports}}'
    ```
  * You should see `0.0.0.0:8082->80/tcp` for the container.
  * Verify the firewall rule targets your VM (tag or “all instances”).

* **Blank/403 at `/` but `/Resume.html` works:**
  The repo has no `index.html`. The pipeline copies `Resume.html` to `index.html` automatically. Confirm it with:

  ```bash
  docker exec -it resume-8082 ls -la /usr/share/nginx/html
  ```

* **Freeing the port / cleaning old containers:**

  ```bash
  docker rm -f resume-8082 2>/dev/null || true
  docker container prune -f
  docker image prune -f
  ```

---

## Security notes

* Don’t leave broad `0.0.0.0/0` rules forever. Prefer **network tags** and restrict source IPs where possible.
* Add TLS/HTTPS via a reverse proxy if exposing the site publicly.

---

## License

MIT (or your preferred license)

---

### Quick commands

Show external IP from the VM:

```bash
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip
```

Show listeners:

```bash
sudo ss -lntp | egrep ':8080|:8082' || true
```

---

Copy this into `README.md` and push. If you want, I can also generate a minimal `index.html` starter and a `.gitignore` for you.











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





