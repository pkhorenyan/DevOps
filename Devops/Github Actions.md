
```yaml
#---------------------------------------------------------------------
# GitHub Action Workflow to Deploy Flask App to AWS ElasticBeanstalk
#
# Version      Date        Info
# 1.0          2019        Initial Version
#
# Made by Denis Astahov ADV-IT Copyleft (c) 2019
#---------------------------------------------------------------------
name: CI-CD-Pipeline-to-AWS-ElasticBeanstalk
env:
  EB_PACKAGE_S3_BUCKET_NAME : "zzz.flask-app"
  EB_APPLICATION_NAME       : "MyFlask"
  EB_ENVIRONMENT_NAME       : "MyFlask-env"
  DEPLOY_PACKAGE_NAME       : "flask-app-${{ github.sha }}.zip"
  AWS_REGION_NAME           : "us-west-2"

on: 
  push:
    branches: 
      - master

jobs:
    my_ci_pipeline:
       runs-on: ubuntu-latest
       
       steps:
         - name: Git clone our repository
           uses: actions/checkout@v1
            
         - name: Create ZIP deployment package
           run : zip -r ${{ env.DEPLOY_PACKAGE_NAME }} ./  -x  *.git*
           
         - name: Configure my AWS Credentils
           uses: aws-actions/configure-aws-credentials@v1
           with:
             aws-access-key-id    : ${{ secrets.MY_AWS_ACCESS_KEY }}
             aws-secret-access-key: ${{ secrets.MY_AWS_SECRET_KEY }}
             aws-region           : ${{ env.AWS_REGION_NAME }}

         - name: Copy our Deployment package to S3 bucket
           run : aws s3 cp ${{ env.DEPLOY_PACKAGE_NAME }} s3://${{ env.EB_PACKAGE_S3_BUCKET_NAME}}/
         
         - name: Print nice message on completion of CI Pipeline
           run : echo "CI Pipeline part finished successfully"
           
    my_cd_pipeline:
       runs-on: ubuntu-latest
       needs: [my_ci_pipeline]
       
       steps:
         - name: Configure my AWS Credentils
           uses: aws-actions/configure-aws-credentials@v1
           with:
             aws-access-key-id    : ${{ secrets.MY_AWS_ACCESS_KEY }}
             aws-secret-access-key: ${{ secrets.MY_AWS_SECRET_KEY }}
             aws-region           : ${{ env.AWS_REGION_NAME }}
         
         - name: Create new ElasticBeanstalk Applicaiton Version
           run : |
            aws elasticbeanstalk create-application-version \
            --application-name ${{ env.EB_APPLICATION_NAME }} \
            --source-bundle S3Bucket="${{ env.EB_PACKAGE_S3_BUCKET_NAME }}",S3Key="${{ env.DEPLOY_PACKAGE_NAME }}" \
            --version-label "Ver-${{ github.sha }}" \
            --description "CommitSHA-${{ github.sha }}"

         - name: Deploy our new Application Version
           run : aws elasticbeanstalk update-environment --environment-name ${{ env.EB_ENVIRONMENT_NAME }} --version-label "Ver-${{ github.sha }}"
           
         - name: Print nice message on completion of CD Pipeline
           run : echo "CD Pipeline part finished successfully"  
        
```



Деплой простого flask приложения.
Создаем SSH ключи и загружаем публичный ключ в .ssh/authorized_keys
Частный ключ добавляем в секреты GitHub репозитория.

```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - master
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.EC2_HOST }} >> ~/.ssh/known_hosts

      - name: Copy project to EC2
        run: |
          rsync -avz -e "ssh -i ~/.ssh/id_ed25519" ./ ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:/home/${{ secrets.EC2_USER }}/app

      - name: Deploy with Docker Compose
        run: |
          ssh -i ~/.ssh/id_ed25519 ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} << 'EOF'
            cd ~/app
            docker-compose down
            docker-compose up -d --build
          EOF
```