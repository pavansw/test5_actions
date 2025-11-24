Step-by-step guide to building and deploying a Docker container using ubuntu:latest as the base image for an Apache server.
This workflow uses two separate jobs on GitHub-hosted runners:
Build Job: Builds the image from ubuntu:latest and pushes it to Docker Hub.
Deploy Job: Pulls that image on a fresh runner and starts the container.

Step 1: Create the Project Structure

On your local machine or in your GitHub repository, create the following files:



/my-project
├── index.html
├── Dockerfile
└── .github
    └── workflows
        └── main.yml



Step 2: Create the Website Content

Create a file named index.html. This is the page Apache will serve.

HTML


<!DOCTYPE html>
<html>
<head>
    <title>Pavan's Apache Server</title>
</head>
<body>
    <h1>Hello Pavan!</h1>
    <p>This Apache server was built from an Ubuntu base image via GitHub Actions.</p>
</body>
</html>



Step 3: Create the Dockerfile

Create a file named Dockerfile (no extension).
Since you requested ubuntu:latest as the base, we must manually install Apache and configure it to run in the foreground so the container doesn't exit immediately.

Dockerfile


# 1. Use the official Ubuntu latest image
FROM ubuntu:latest

# 2. Avoid prompts from apt (like timezone selection)
ENV DEBIAN_FRONTEND=noninteractive

# 3. Update Ubuntu and install Apache
RUN apt-get update && \
    apt-get install -y apache2 && \
    apt-get clean

# 4. Copy our custom index.html to the Apache web folder
COPY index.html /var/www/html/index.html

# 5. Expose Port 80
EXPOSE 80

# 6. Start Apache in the foreground
CMD ["apache2ctl", "-D", "FOREGROUND"]



Step 4: Configure GitHub Secrets

For GitHub Actions to push to your Docker Hub account (pavansw), you must provide credentials.
Go to your GitHub Repository.
Click Settings (Top tab) → Secrets and variables (Left sidebar) → Actions.
Click New repository secret.
Add these two secrets:
Name
Value
DOCKER_USERNAME
Pavansw (You have to use yours)
DOCKER_PASSWORD
Your Docker Hub Access Token (Recommended) or Password.


Step 5: Create the GitHub Actions Workflow

Create the file .github/workflows/main.yml.
This file defines the two runners (jobs).

YAML


name: Build and Deploy Apache on Ubuntu

on:
  push:
    branches: [ "main" ]

jobs:
  # ======================================================
  # JOB 1: BUILD & PUSH
  # Run on GitHub-hosted ubuntu-latest runner
  # ======================================================
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      # Log in to Docker Hub using your secrets
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      # Set up Docker Buildx (required for modern builds)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Build the image using the Dockerfile and push to pavansw/apache-ubuntu-demo
      - name: Build and Push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: pavansw/apache-ubuntu-demo:latest

  # ======================================================
  # JOB 2: DEPLOY (Simulated Deployment)
  # Run on a NEW, FRESH GitHub-hosted ubuntu-latest runner
  # ======================================================
  deploy-container:
    needs: build-and-push  # This ensures Job 2 only starts if Job 1 succeeds
    runs-on: ubuntu-latest

    steps:
      # Log in again (because this is a new runner)
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Pull and Run Container
        run: |
          echo "Step 1: Pulling the image we just built..."
          docker pull pavansw/apache-ubuntu-demo:latest
          
          echo "Step 2: Starting the container..."
          # Run in detached mode (-d), map port 80 to 80
          docker run -d --name my-apache-server -p 80:80 pavansw/apache-ubuntu-demo:latest

          echo "Step 3: Verifying the container is running..."
          docker ps

      # Since this runner is ephemeral (it dies after the job), we verify it works locally
      - name: Verify Application (Integration Test)
        run: |
          echo "Waiting 5 seconds for Apache to initialize..."
          sleep 5
          
          echo "Making a request to the server..."
          curl -v http://localhost:80



Summary of Execution

Commit and Push: When you push these files to GitHub, the Action triggers.
Job 1 (Build): * Downloads ubuntu:latest.
Installs Apache inside it.
Puts your index.html inside it.
Uploads the final image to Docker Hub as pavansw/apache-ubuntu-demo:latest.
Job 2 (Deploy):
Starts on a completely new machine.
Downloads your image from Docker Hub.
Runs it.
The curl command prints your HTML code to the Action logs to prove it worked.
Important Note on GitHub Runners:
Because runs-on: ubuntu-latest is a hosted runner provided by GitHub, the container created in Job 2 will disappear as soon as the job finishes. This setup is perfect for CI/CD Testing (verifying the deployment works).
If you want the website to remain online permanently, you would simply install a "Self-Hosted Runner" agent on your own server (e.g., AWS EC2) and change runs-on: ubuntu-latest to runs-on: self-hosted.
