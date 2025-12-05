Jenkins CI/CD pipeline for Python apps—automates cloning, dependency setup, testing, and deployment for reliable, repeatable releases.

## **About the Project**

This project demonstrates how to build a fully automated **CI/CD pipeline** using **Jenkins Pipeline (Declarative Pipeline + Jenkinsfile)** to deploy a Python application to a remote EC2 instance. The pipeline executes all stages sequentially—from cloning the repository to setting up dependencies, running tests, and starting the application—ensuring a fully automated workflow.

Every code change pushed to GitHub triggers this pipeline, enabling reliable, repeatable deployments without manual intervention.

## Process:

The CI/CD pipeline will follow the same sequential stages:

1. **Clone Repository** (Python app code from GitHub)
2. **Upload Files to EC2** (Transfer to the deployment server)
3. **Install Dependencies** (Run **`pip install`** on remote server)
4. **Start Application** (Launch/restart the app using **Gunicorn**)
5. **Continuous Integration** (Configure a **GitHub Webhook** to trigger the build on push)

## **Technologies Used**

- **Jenkins** → CI/CD automation system
- **Pipeline (Jenkinsfile)** → Defines the complete workflow
- **SSH Agent Plugin** → Secure authentication for remote EC2 access
- **GitHub** → Source code repository and webhook integration
- **Python 3.x** → Application runtime
- **Pip / Virtualenv** → Dependency management
- **Gunicorn / systemd** → Process manager to run the Python app
- **AWS EC2** → Deployment server (pythonapp-server)
- **Key Pair Authentication** → Secure remote command execution

## **Prerequisites**

- Jenkins installed on a dedicated EC2 instance
- GitHub repository containing a Python application and Jenkinsfile
- SSH private key copied and configured in Jenkins Credentials
- AWS EC2 (Ubuntu) instance prepared as a deployment target
- Python 3.x, Pip, and virtualenv installed on deployment server
- Proper security group rules (allow port 8000 or relevant app port)
- Jenkins Pipeline Plugins installed

## **Step 1: Start Jenkins Server**

1. Go to AWS console → EC2 Services
2. Start Jenkins server
3. Connect to the instance via SSH
4. Set hostname (optional) → `jenkins`

## Step 2: Launch instance (`pythonapp-server`)

The deployment server setup needs to install Python tools instead of Node.js tools.

1. Go to AWS console → EC2 Services.
2. Launch a new instance (`pythonapp-server`).
3. Connect instance via SSH.
4. Install all dependencies:

```bash
sudo apt update
# Install Python 3 and Pip
sudo apt install python3 -y
sudo apt install python3-pip -y
python3 --version
pip3 --version
# Install Gunicorn for process management
pip3 install gunicorn
gunicorn --version
```

![Project Screenshot](/images/instance.png).

## Step 3: Create Repo & Python Files

Create a new GitHub repository named **`python-app-cicd`** and add the following files.

### **`app.py`** (Flask Application)

```python
from flask import Flask
import os
app = Flask(__name__)

@app.route('/')
def hello_geek():
    return 'successfully deployed python application through jenkins!!!!!!!!!, webhook adding this time'
@app.route('/hi')
def hell():
    return '<h1>Hiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiii from Flask & Docker</h1>'

if __name__ == "__main__":
    port = int(os.environ.get('PORT', 5000))
    app.run(debug=True, host='0.0.0.0', port=port)

```

### **`requirements.txt`** (Python Dependencies)

```python
click==8.0.3
colorama==0.4.4
Flask==2.0.2
itsdangerous==2.0.1
Jinja2==3.0.3
MarkupSafe==2.0.1
Werkzeug==2.0.2
gunicorn==20.1.0
```

### **`jenkinsfile`** (Declarative Pipeline)

The new `jenkinsfile` uses the same stage structure but executes **`pip install`** and starts the app with **`gunicorn`**.

```bash
pipeline {
    agent any

    environment {
        SSH_CRED = 'node-app-key'
        SERVER_IP = '54.221.163.23'
        REMOTE_USER = 'ubuntu'
        APP_DIR = '/home/ubuntu/pythonapp'
    }

    stages {
        stage('Clone Repo') {
            steps {
                git url: 'https://github.com/rahullengare/python-app-cicd.git', branch: 'main'
            }
        }

        stage('Deploy to Server') {
            steps {
                sshagent(credentials: ["${SSH_CRED}"]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} "mkdir -p ${APP_DIR}"
                        scp -r Dockerfile README.md app.py requirements.txt test ${REMOTE_USER}@${SERVER_IP}:${APP_DIR}/
                    '''
                }
            }
        }

        stage('Install & Run App') {
            steps {
                sshagent(["${SSH_CRED}"]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${SERVER_IP} '
                            sudo apt update &&
                            sudo apt install -y python3-venv python3-pip &&
                            cd ${APP_DIR} &&
                            python3 -m venv venv &&
                            source venv/bin/activate &&
                            pip install --upgrade pip &&
                            pip install -r requirements.txt &&
                            nohup python3 app.py --host=0.0.0.0 > app.log 2>&1 &
                            exit 0
                        '
                    """
                }
            }
        }
    }
}

```

![Project Screenshot](/images/repo-done.png).
![Project Screenshot](/images/repo2.png).

## Step 4: Install Plugins, Create Job

### 1. Install Plugins & Create Job

1. Install the required plugins: **Pipeline, SSH Agent, GitHub**.
2. Create a new Jenkins Pipeline Job (`pythonapp-deployment`).
3. Configure the job exactly as before, pointing to your new **`python-app-cicd`** repository and **`jenkinsfile`**

![Project Screenshot](/images/job.png).
![Project Screenshot](/images/job2.png).

## Step 5: Check Python App

1. After a successful build, copy the deployment server's public IP.
2. Paste the URL into your browser with the Python port (**5000**):

```bash
http://98.89.20.143:5000/
```

![Project Screenshot](/images/run.png).
Your Python/Flask application should be running successfully!

## Step 6: Configure GitHub Webhook for Continuous Integration

you only upload or commit changes then automatically deploy.  

1. **Retrieve URL:** Get the Jenkins Webhook Listener URL:

```bash
http://<Jenkins_IP>:8080/github-webhook/
```

1. **Access GitHub:** Go to your repository **Settings** → **Webhooks**.
2. **Add Hook:** Click **"Add webhook"** and paste the URL into the **Payload URL** field.
3. **Set Trigger:** Select **"Just the push event"** as the trigger event.
4. **Test Push:** Commit a code change and push it to GitHub to automatically trigger the Jenkins build.
5. **Test Push:** Commit a code change and push it to GitHub to automatically trigger the Jenkins build.

![Project Screenshot](/images/webhook.png).
![Project Screenshot](/images/run2.png).

## Step 7: Now all Instances Terminated

1. Go to AWS console → EC2 Services
2. Select Your instance you want to deleted
3. Click on terminate
