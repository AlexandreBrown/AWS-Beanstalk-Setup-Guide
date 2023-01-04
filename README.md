# AWS-Beanstalk-Setup-Guide  

## Prerequisites
1. Create an IAM user with programmatic access.
2. Download AWS CLI https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
3. Setup your AWS account : ```aws configure```
4. Download & install python https://www.python.org/downloads/ (I recommend using a virtual enviornment via micromamba/conda or venv)
5. Download aws Beanstalk CLI : `pip3 install awsebcli`

## Setup
### Subdomain Delegation
Let's say we have a root domain (eg: `mlprojects.ai`) hosted on some domain name provider (eg: https://www.namecheap.com/ , https://www.godaddy.com/ etc).  
Our goal is to create a subdomain on AWS Route53 without having to migrate the root domain `mlprojects.ai` to Route53.  
To do so, we need to do the following :  
1. Create a Route53 Hosted zone for the subdomain (eg: `staging.mlprojects.ai`)  
![image](https://user-images.githubusercontent.com/26939775/210656785-854c47a4-8598-4ce4-ba46-5a1ddc2a0840.png)
1. Copy/Take note of the value of the NS record that was created automatically by Route53, it contains the DNS Server Names for the subdomain.
1. Navigate to the domain name provider of your root domain (eg: https://www.namecheap.com/ or any other). In this example, the root domain is `mlprojects.ai`.  
1. Navigate to the DNS settings and create a new record of type `NS` for every DNS Server Name from step 2.  
This essentially enables the subdomain delegation from the root domain name provider to Route53.  
This is the only change required for the root domain name provider configuration, everything from now on will be done on Route53 and we won't have to edit the root domain name provider configuration.  

### HTTPS Certificate
1. Navigate to AWS Certificate Manager (ACM)
1. Click to "Request" an HTTPS certificate using AWS Certificate Manager (ACM)
1. Select public certificate
1. Enter the fully qualified domain name as your subdomain : `staging.mlprojects.ai`
1. Click "Request"
#### Confirm Ownership Of The Domain
1. From the ACM click "Create records in Route53" in the certificate details page.  
This will allow Route53 to validate ownership.  
After a few minutes, AWS will have issued your certificate upon confirming ownership.  
From the ACM certificates page you should see that your certificate is issued and not in use :  
![image](https://user-images.githubusercontent.com/26939775/210650886-9cfade31-9945-4510-bfe5-5b0a176ccdd7.png)

### Create Your Application
1. For the purpose of this example, a basic python application was created in `application.py`.

### Create Beanstalk Application
Here we create an application called `myapp` that uses the platform (AWS predefined env) `python-3.8` under the region `us-east-1` :  
1. `eb init myapp --platform python-3.8 --region us-east-1`

### Create Beanstalk Environment With HTTPS + HTTP to HTTPS redirect
These steps follow the [AWS documentation](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/configuring-https-elb.html) and the config file is taken from the official [AWS repository](https://github.com/awsdocs/elastic-beanstalk-samples/blob/master/configuration-files/aws-provided/resource-configuration/alb-http-to-https-redirection-full.config).  
1. Create a folder for the beanstalk configuration, it has to be named `.ebextensions` at the root of your project.
You can learn more about this configuration folder here : https://docs.amazonaws.cn/en_us/elasticbeanstalk/latest/dg/environment-configuration-methods-before.html#configuration-options-before-ebextensions  
1. Create a `.config` file under the `.ebextensions` folder that will contain the environment resources options (see the file `staging-resources-options.config` from this repo).
1. Edit the `CertificateArn` from the `.ebextensions/staging-resources-options.config` file by entering the ARN of the ACM Certificate created earlier.
1. Create the environment : `eb create staging-env --region us-east-1 --elb-type application`  
Here we created an environment called `staging-env` in the region `us-east-1` that has a load balancer of type `application`.  
For more information on what is an Appliation Load Balancer see the [AWS documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)  
Note that the configuration in the `.config` file presented earlier expects a load balancer of type `application`.  

### Use Custom Domain For Beanstalk Environment
These steps follow the [AWS documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-beanstalk-environment.html)
1. Navigate to the subdomain Route53 Hosted Zone (eg: `staging.mlprojects.ai`)
1. Click "Create a record"
1. Leave the `Record name` blank  
![image](https://user-images.githubusercontent.com/26939775/210655021-416b385b-61f7-465b-a5d6-e5001f9ba5b4.png)
1. Leave the `Record type` to type `A`  
![image](https://user-images.githubusercontent.com/26939775/210655116-2416d601-fc02-46fa-927d-330ec3cb657e.png)  
1. Toggle the "Alias" toggle  
![image](https://user-images.githubusercontent.com/26939775/210655224-f689bc3f-e349-45e0-9d04-fe70955d3795.png)  
1. Under `Route traffic to` select `Alias to Elastic Beanstalk environment`.
1. Select the region you deployed your Beanstalk environment (eg: `us-east-1`).  
1. Select your Beanstalk environment from the dropdown :  
![image](https://user-images.githubusercontent.com/26939775/210655489-82c19600-7205-44df-b104-297400c8dacc.png)  
1. Click "Create records"  

Visit your application using the subdomain url you used when you created the Route53 Hosted Zone (eg: `https://staging.mlprojects.ai/`)  
![image](https://user-images.githubusercontent.com/26939775/210655802-de150efe-841f-404b-a963-8e8cd479841d.png)

## Deploy New Changes To An Environment
1. Edit your application source code.
1. Deploy your application to your Beanstalk environment : `eb deploy staging-env --region us-east-1`  
Here we deploy our application to the environment `staging-env` that we created earlier.  
1. Navigate to your page and notice the changes were applied :  
![image](https://user-images.githubusercontent.com/26939775/210657779-2d6e701d-4a0f-4a72-a4de-05e8f3638906.png)
