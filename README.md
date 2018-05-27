# AWS CLI Docker container for dev and production 

To use the container you will have to pull it from the public dockerhub container registry 

```bash 
docker pull eranchetz/awscli
```

this container is a built automatically, see more details on the [dockerhub](https://hub.docker.com/r/eranchetz/awscli/)

## Ways of authentication with AWS 

### 1. Using Environment Variables

Set your access keys: 
```bash
export MY_AWS_ACCESS_KEY=<AWS_ACCESS_KEY>
export MY_AWS_SECRET_KEY=<AWS_SECRET_KEY>

```

and run:
```bash
docker run -it --rm \
-e AWS_ACCESS_KEY_ID=${MY_AWS_ACCESS_KEY} \
-e AWS_SECRET_ACCESS_KEY=${MY_AWS_SECRET_KEY} \
-e AWS_DEFAULT_REGION=eu-central-1 \
-v $PWD:/data \
-w /data \
eranchetz/awscli \
s3 ls --recursive s3://tech-test-zooz/
```

### 2. Using ~/.aws/credentials file

awscli will look for credentials in the 
`~/.aws/credentials` file

create the the file below with the name `credentials`:

```toml
[default]
aws_access_key_id=<AWS_ACCESS_KEY>
aws_secret_access_key=<AWS_SECRET_KEY>
```

```bash
docker run -it --rm \
-e AWS_DEFAULT_REGION=eu-central-1 \
-v $PWD:/data \
-w /data \
-v $PWD/credentials:/home/aws/.aws/credentials \
eranchetz/awscli \
s3 ls --recursive s3://tech-test-zooz/
```

### 3. Using IAM role (ec2 metadata)

If running on AWS Instance you can use IAM Roles and attach it to the instance.
aws cli will automatically fetch the credentials using the instance metadata, 
this options is *best* for running in production environments on ec2.



a.1. create aws role json file 
create this file : `ec2-role-trust-policy.json`
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
```

a.2. create IAM role 

```bash
 $ aws iam create-role --role-name s3access --assume-role-policy-document file://ec2-role-trust-policy.json
```

a.3. create access policy file` ec2-role-access-policy.json`
```json
    {
        "Effect": "Allow",
        "Action": [
            "s3:GetObject",
            "s3:ListBucket"
        ],
        "Resource": [
            "arn:aws:s3:::tech-test-zooz",
            "arn:aws:s3:::tech-test-zooz/*"
        ]
    }
```

a.4. 

```bash
 
# Attach the access policy to the role. 

$ aws iam put-role-policy --role-name s3access --policy-name S3-Permissions --policy-document file://ec2-role-access-policy.json

# Create an instance profile named s3access-profile. 

$ aws iam create-instance-profile --instance-profile-name s3access-profile

# Add the s3access role to the s3access-profile instance profile. 

$ aws iam add-role-to-instance-profile --instance-profile-name s3access-profile --role-name s3access
```

b. launch an instance and attach role

```bash
 $ aws ec2 run-instances --image-id <ami-11aa22bb> --iam-instance-profile Name="s3access-profile" --key-name <my-key-pair> --security-groups <my-security-group> --subnet-id <subnet-1a2b3c4d>
```

c. ssh to machine and run (you might need to install docker)

```bash
docker run -it --rm \
eranchetz/awscli \
s3 ls --recursive s3://tech-test-zooz/
```

> Notice! if you are not using default docker network, you must make sure the container has access to the host 169.254.169.254 (ec2 metadata endpoint)


### 4. Using Hashicorp Vault AWS Secrets Engine 

This generates AWS access credentials dynamically based on IAM policies.

for more information please see vault [documentation](https://www.vaultproject.io/docs/secrets/aws/index.html)  


## Ranking based on what is best to deploy/use on a production system: 

>Notice! All auth methods have their pros and cons, it depends on your usage/use case. 

### (1) IAM Role:  

 - This approach is best for production environments, the secrets are generate by AWS, The metadata information is automatically supplied to all EC2 instances (or other services, for example: AWS Lambda), and IAM credentials are automatically supplied to AWS instances in IAM instance profiles. 

 - Limitations: Only works with AWS Instances, working with CI/CD pipelines on 3rd party providers will need to use a different approach.

 ### (2) Environment Varaibles: 

 - This conforms to the [12-Factor methodology](https://12factor.net/), which states that [config](https://12factor.net/) and secrets should be part of the environment, this approach makes it easy to change between deploys, unlike config files, there is little chance of them being checked into the code repo accidentally.  
 That said, this method is probably the best to use in CI/CD pipeline while encrypting the env vars, [GitLab Example](https://docs.gitlab.com/ee/ci/variables/#secret-variables).

- Limitations: You will have to provide the Environment Variables on each instance, or create a mechanism to populate the instances with these environment variables. 

### (3) Using ~/.aws/credentials file

 - The least relevant approach, except for the chance of the file being checked into the code repo accidentally, there is also the need to pass the file to the instance, copying it to the container, either via COPY or mounting a Volume.






