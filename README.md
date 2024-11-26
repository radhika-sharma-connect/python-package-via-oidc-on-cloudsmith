# Python Package Deployment to Cloudsmith via GitHub Actions and OIDC

This repository demonstrates the process of building and deploying a Python package to Cloudsmith using GitHub Actions and OpenID Connect (OIDC) for secure authentication.

## Repository Overview
This repository contains:
- A Python project with the following structure
  
  ![image](https://github.com/user-attachments/assets/3a91fc22-03e1-4dca-9ec6-49af01a33ec8)
- A GitHub Actions workflow for building and pushing the Python package to Cloudsmith.


## Setup and Configuration

### 1. Cloudsmith Setup

#### 1. Create a Cloudsmith Account
- Sign up at [Cloudsmith](https://cloudsmith.com) and verify your email.
- Create an organization with the name `interview-radhika-sharma`

#### 2. Create a Repository
- Navigate to the organization and create a repository:
  - **Name:** `interview-assessment`
  - **Storage Region:** Default
  - **Repository Type:** Private

#### 3. Create a Service Account
- Go to **Accounts > Services** and create a service account:
  - **Name:** `technical-assessment-service-account`
  - Add the **owners** team to the service account for appropriate privileges.

#### 4. Set Access Control on Repository
- Navigate to **Repository > Access Control**.
- Add the service account and grant it **Write Permissions**.

#### 5. Configure OpenID Connect (OIDC)
- Go to **Settings > OpenID Connect** in Cloudsmith.
- Create an OIDC provider for GitHub:
  - **Name:** `Github-Provider`
  - Specify the necessary claims to validate, such as:
    ```json
    {
      "repository_owner": "radhika-sharma29"
    }
    ```
  - Add above created service account
    
    ![image](https://github.com/user-attachments/assets/2ef7d536-9f7c-47d3-b996-ff792cb03c59)

### 2. GitHub Repository Setup

#### 1. Create a GitHub Repository
- **Repository Name:** `python-package-via-oidc-on-cloudsmith`.
- Clone the repository.

### 3. Build and Deploy the Python Package

#### 1. Add Python Package Project File
- Add the required python packages file including metadata like `pyproject.toml`, main.py etc.

#### 2. GitHub Actions Workflow
- Add a `.github/workflows/deploy.yml` file with the  following content :
```yaml
name: Push a Python package to Cloudsmith with OIDC
on:
  push:
    branches:
      - main
      
permissions:
  id-token: write      # Necessary to GH Identity Provider to write the JWT token which Cloudsmith needs to read
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    name: Push a Python package to Cloudsmith using OIDC to authenticate
    env:
     CLOUDSMITH_SERVICE_ACCOUNT_SLUG: "technical-assessment-service-a"  # It would be the service account identifier
     CLOUDSMITH_ORGANIZATION: "interview-radhika-sharma"
     CLOUDSMITH_REPO: "interview-assessment"
      
    steps:
      # Step 1 : Checkout the repository
      - name: Check out code
        uses: actions/checkout@v4.2.2
        
      # Step 2: Set up Python environment
      - name: Set up Python
        uses: actions/setup-python@v5.3.0
        with:
          python-version: 3.9

      # Step 3: Install build tools
      - name: Install build tools
        run: |
          pip install build

      # Step 4: Build the package
      - name: Build the Python package
        run: |
          python -m build

      # Step 5: Get OIDC token
      - name: Get OIDC token
        run: |
          value=$(curl -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=api://AzureADTokenExchange" | jq '.value')
          token=$(curl -X POST -H "Content-Type: application/json" -d "{\"oidc_token\":$value, \"service_slug\":\"$CLOUDSMITH_SERVICE_ACCOUNT_SLUG\"}" https://api.cloudsmith.io/openid/$CLOUDSMITH_ORGANIZATION/ | jq -r '.token')
          curl --request GET --url https://api.cloudsmith.io/v1/user/self/ --header "X-Api-Key:Bearer $token" --header 'accept: application/json'          
          echo "CLOUDSMITH_API_KEY=$token" >> $GITHUB_ENV

      # Step 6: Push the Python Package to Cloudsmith
      - name: Push the Built Python Package to Cloudsmith
        uses: cloudsmith-io/action@v0.6.6
        with:
          command: "push"
          format: "python"
          owner: $CLOUDSMITH_ORGANIZATION
          repo: $CLOUDSMITH_REPO
          republish: "true"
          file: "dist/sample_python_package-*.tar.gz"
```
- **Workflow Steps:**

  - *Checkout Code*
     - The workflow retrieves the latest code from the GitHub repository to ensure that the build process works with the most up-to-date project files.

  - *Set up Python & Tools*
     - It installs Python 3.9 on the runner, along with necessary tools like `pip` (Python's package installer) and `build` (to package the Python project).

  - *Build Package* 
     - The workflow builds the Python package from the project files, creating distributable artifacts (e.g., `.tar.gz`) that are ready to be pushed to Cloudsmith.

  - *Authenticate & Publish* 
     - Using OpenID Connect (OIDC), the workflow securely authenticates with Cloudsmith by obtaining an API key and uses that key to publish the built package to your Cloudsmith repository.

### 3. Commit and Push Changes
#### 1. Add, commit, and push your changes to the GitHub repository:
```
git add .
git commit -m "Initial add of python package & workflow pipeline"
git push origin main
```
### 4. Trigger the Workflow
- when changes are pushed to the main branch, The workflow will automatically:
  - Build the Python package.
  - Push it to Cloudsmith

## References

- [Cloudsmith OIDC Set Up with Github Actions Documentation](https://help.cloudsmith.io/docs/setup-cloudsmith-to-authenticate-with-oidc-in-github-actions)
- [Packaging Python Projects](https://packaging.python.org/en/latest/tutorials/packaging-projects/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions)

