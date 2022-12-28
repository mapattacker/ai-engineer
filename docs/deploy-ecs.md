# Docker Compose & ECS

```bash
# create ECS context
docker context create ecs <name>
docker context ls
docker context use <name>

# login to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com

# create ECS
docker compose up --project-name <name> # create cloudformation stack
docker compose down # delete cluster

docker context rm -f flask-test
docker context use default

# create cloudformation template
# can use yml file to view visually in AWS designer
docker compose convert > cloudformation.yml
```

From the documentation, "By default, the Docker Compose CLI creates an ECS cluster for your Compose application, a Security Group per network in your Compose file on your AWS account’s default VPC, and a LoadBalancer to route traffic to your services."