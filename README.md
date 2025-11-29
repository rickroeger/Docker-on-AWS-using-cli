# Docker-on-AWS-using-cli
How to create an AWS instance running docker using AWSCLI

Prerequisites: aws-cli/2.31.33

## 1. Configure the AWSCLI Profile

Configure the awscli profile. You need to create accesskey and secretkey on AWS WEB Console and provide on this command
```
aws configure --profille [profile_name]
```
```
aws configure --profille mvp
```
## 2. Get the Ubuntu AMI-ID

Run the command below to get the AMI-ID for ubuntu images on AWS.
command:
```
aws ssm get-parameter \
    --name "/aws/service/canonical/ubuntu/server/[ubuntu_version]/stable/current/amd64/hvm/ebs-gp2/ami-id" \
    --query "Parameter.Value" \
    --output text \
    --profile [profile]
```
example:
```
aws ssm get-parameter \
    --name "/aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id" \
    --query "Parameter.Value" \
    --output text \
    --profile mvp
```
## 3. Prepare the Cloud Init file
You must create the file below. It's the cloud init file. The script run on machine bootstrap
example:
```
cat << EOF > cloud-init.txt
#cloud-config
runcmd:
  - apt-get update
  - apt-get install -y docker.io
  - systemctl start docker
  - systemctl enable docker
EOF
```
## 4. Create EC2 key-pair
commands:
```
aws ec2 create-key-pair --key-name [key_pair_name] --query 'KeyMaterial' --output text --profile mvp > [key_pair_name]
#change permissions is required for ssh security
chmod 400 [key_pair_name]
```
example:
```
aws ec2 create-key-pair --key-name MVPDOCKER001 --query 'KeyMaterial' --output text --profile mvp > MVPDOCKER001.pem
chmod 400 MVPDOCKER001.pem
```

## 5. Run The Instace
commands:
```
aws ec2 run-instances \
    --image-id [ami_id] \
    --count 1 \
    --instance-type [instance_time] \
    --key-name [key_name] \
    --associate-public-ip-address \
    --profile [profile] \
    --user-data file://cloud-init.txt 
```
example:
```
aws ec2 run-instances \
    --image-id ami-00b13f11600160c10 \
    --count 1 \
    --instance-type t2.micro \
    --key-name MVPDOCKER001 \
    --associate-public-ip-address \
    --profile mvp \
    --user-data file://cloud-init.txt 

```
## 6. Get the Public DNS
command:
```
aws ec2 describe-instances  --profile [profile] --query Reservations[].Instances[].NetworkInterfaces[].Association[].PublicDnsName
```
example:
```
aws ec2 describe-instances  --profile mvp --query Reservations[].Instances[].NetworkInterfaces[].Association[].PublicDnsName
[
    "ec2-54-92-000-000.compute-1.amazonaws.com"
]
```

## 7. Connect on Public Instance
command:
```
ssh -i [key_name].pem ubuntu@[external_dns]
```
example:
```
ssh -i MVPDOCKER001.pem ubuntu@ec2-54-234-000-000.compute-1.amazonaws.com
```

Now you may test the docker installed on the servers running the a httpd server
example:
Running the http
```
ubuntu@ip-172-31-27-133:~$ sudo docker run httpd &
[1] 2412
ubuntu@ip-172-31-27-133:~$ Unable to find image 'httpd:latest' locally
latest: Pulling from library/httpd
0e4bc2bd6656: Pull complete
4742a9e996d1: Pull complete
4f4fb700ef54: Pull complete
87a14f083967: Pull complete
9cd0271fa751: Pull complete
5b4d5959fc75: Pull complete
Digest: sha256:f9b88f3f093d925525ec272bbe28e72967ffe1a40da813fe84df9fcb2fad3f30
Status: Downloaded newer image for httpd:latest
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.17.0.2. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.17.0.2. Set the 'ServerName' directive globally to suppress this message
[Sat Nov 29 15:04:51.122921 2025] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.65 (Unix) configured -- resuming normal operations
[Sat Nov 29 15:04:51.127781 2025] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'

```
Example:
Checking the docker status
```
ubuntu@ip-172-31-27-133:~$ sudo docker ps
CONTAINER ID   IMAGE     COMMAND              CREATED          STATUS          PORTS     NAMES
54599a448c21   httpd     "httpd-foreground"   11 seconds ago   Up 11 seconds   80/tcp    trusting_spence
```

## 8. Terminate the instace
command:
```
aws ec2 terminate-instances --instance-ids [instance_id] --profile [profile]
```
example:
```
aws ec2 terminate-instances --instance-ids i-0be6be1dc050adb04 --profile mvp
```
if you don't have the instance id, you may get this information using the comand below:
```
aws ec2 describe-instances  --profile mvp --query Reservations[].Instances[].InstanceId
```
