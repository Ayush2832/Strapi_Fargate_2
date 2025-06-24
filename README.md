# Task10 Host/Publish the Project
After logging in:
- Create Content Types (Collections, Singles, etc.).
- Use Roles & Permissions to allow public access if needed.
- Go to Settings > Users & Permissions Plugin and configure access.
- Publish your content.
- Use the public URL (e.g., http://<alb-dns-name>/api/[your-endpoint]) to test APIs or connect a frontend

## 1. Changes in strapi code
We have to make a new file called `vite.config.ts` under `src > admin` folder and the changes are described below
```ts
import { server } from '@strapi/strapi/admin/test';
import { mergeConfig, type UserConfig } from 'vite';

export default (config: UserConfig) => {
  // Important: always return the modified config
  return mergeConfig(config, {
    resolve: {
      alias: {
        '@': '/src',
      },
    },

    // To allow loadbalancer dns
    server: {
        allowedHosts: true
    }
  });
};
```

## 2. Github Actions
- We deploy our image to the docker hub by building it and then making the infrastructure using terraform.
- The structure of `deploy.yml` is

```yml
name: Strapi_Fargate

on:
  push:
    branches: master

jobs:
    Strapi_task:
        runs-on: ubuntu-latest

        env:
            IMAGE_NAME: ${{ secrets.DOCKER_USERNAME }}/strapi3

        steps:
        - name: Checkout-code
          uses: actions/checkout@v4

        - name: docker login
          uses: docker/login-action@v3
          with:
            username: ${{secrets.DOCKER_USERNAME}}
            password: ${{secrets.DOCKER_PASSWORD}}

        - name: Creating env file
          run: |
            cat <<EOF > ./strapi10/.env
            ${{secrets.ENV_FILE}}
            EOF
        
        - name: Build image
          run: |
            docker build -t ${{env.IMAGE_NAME}}:v5 ./strapi10/

        - name: Push the image to docker hub
          run: |
            docker push ${{env.IMAGE_NAME}}:v5 

        - name: terraform setup
          uses: hashicorp/setup-terraform@v3

        - name: Terraform init
          run: terraform init
          working-directory: ./terraform

        - name: Terraform Apply
          run: terraform apply -auto-approve
          working-directory: ./terraform
          env:
            AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
            AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

        - name: Import terrfrom.tfstate file as artifact
          uses: actions/upload-artifact@v4
          with:
            name: location of terraform file
            path: ./terraform/terraform.tfstate
```
- Image will build the image and push the image to the docker hub and then terraform init and plan will work. Once all things are configured we will take the `terraform.tfstate` file as a artifact.

- All the terraform files are under terraform directory
    - `alb.tf` is used to configure the AWS load balancer
    - `ecs.tf` is used to configure the ECS cluster+service there
    - `vpc.tf` to create the VPC service which attach to the load balancer.

- Once all these things are configured we will run the command.

> git push origin master

## 3. Content creation in strapi
- Login to the strapi using load balancer dns
> http://strapi-alb-xxxxxxxxx.us-east-2.elb.amazonaws.com/amdin

- Then go the content type builder and make the content of your type.

<image src="images/1.png" width="500">

- Then in the content manager with give the enteries and then go to `settings > user and permissions > roles > public > and then give permission` to access the api

<image src="images/2.png" width="500">

- Finally access the api by copy the url and give the destination
> http://strapi-alb-xxxxxxx.us-east-2.elb.amazonaws.com/api/authors

<image src="images/3.png" width="500">

- We can also use thunder client for that

<image src="images/4.png" width="500">