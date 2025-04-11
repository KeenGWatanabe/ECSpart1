[For Debian OS](DockerIssues.md)

# Setting Up GitHub Workflow with ECR and Unique Tags

To set up a GitHub workflow that builds and pushes your Docker image to AWS ECR with both `latest` and the Git commit SHA as tags, follow these steps:

## 1. Configure AWS Credentials in GitHub Secrets

First, store your AWS credentials in GitHub Secrets:

1. Go to your GitHub repo
https://github.com/KeenGWatanabe/ECSpart1
 → Settings → Secrets → Actions
2. Add these secrets:
   - `AWS_ACCESS_KEY_ID`: Your AWS access key
   - `AWS_SECRET_ACCESS_KEY`: Your AWS secret key
   - `AWS_REGION`: Your AWS region (e.g., `us-east-1`)
   - `ECR_REPOSITORY`: Your ECR repository URI (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com/your-repo`)

## 2. Create the GitHub Workflow File

Create a new file at `.github/workflows/docker-build-push.yml` with this content:

```yaml
name: Build and Push to ECR

on:
  push:
    branches: [ "main" ]

env:
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
  AWS_REGION: ${{ secrets.AWS_REGION }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build the Docker image
          docker build -t $ECR_REPOSITORY:$IMAGE_TAG .
          
          # Tag with both SHA and latest
          docker tag $ECR_REPOSITORY:$IMAGE_TAG $ECR_REPOSITORY:latest
          
          # Push both tags
          docker push $ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REPOSITORY:latest
          
          echo "Image pushed to ECR with tags: $IMAGE_TAG and latest"
```

## 3. Update Your Dockerfile

Make sure your Dockerfile is properly configured in your repository root. For example:

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
```

## 4. Push to Main Branch to Trigger Workflow
![image1](https://github.com/user-attachments/assets/b0f873cb-9fbb-46fa-9294-4b46605395c1)

After committing these changes and pushing to the main branch, the workflow will automatically:

1. Check out your code
2. Authenticate with AWS
3. Log in to ECR
4. Build your Docker image
5. Tag it with both the Git commit SHA and `latest`
6. Push both tagged images to ECR

## Verification

After the workflow runs successfully:
1. Check the Actions tab in your GitHub repo to see the workflow run
2. Go to your AWS ECR repository to see both tagged images:
   - One with the Git SHA (e.g., `sha-123abc...`)
   - One with the `latest` tag

This approach gives you traceability (via the SHA tags) while maintaining the convenience of a `latest` tag.
![image0](https://github.com/user-attachments/assets/ebb50ca4-6d9d-4525-ab8b-28ba4a8bbca4)
