# Terraform AWS Elastic Beanstalk PHP

Terraform script to setup AWS Elastic Beanstalk with a load-balanced or Single Instance PHP app.

## What this script does

* Create an Elastic Beanstalk Application and environment.
* Setup the EB environment with PHP, an Elastic Loadbalancer and forward port from HTTP / HTTPS to the specified instance port.
* (Optionnal) Create a Route53 Alias to link your domain to the EB domain name
* (Optionnal) Create a Cloudfront distribution on top of your Elastic Beanstalk environment


## Usage

Create a `main.tf` file with the following configuration:

### First: create an EB Application

```hcl
##################################################
## Your variables
##################################################
variable "aws_region" {
  type    = string
  default = "eu-west-1"
}
variable "env" {
  type = string
  default = "dev"
}
variable "service_name" {
  type    = string
  default = "php-app-test"
}


##################################################
## AWS config
##################################################
provider "aws" {
  region = var.aws_region
}


##################################################
## Elastic Beanstalk config
##################################################
resource "aws_elastic_beanstalk_application" "eb_app" {
  name        = var.service_name
  description = "My awesome nodeJs App"
}
```

### Then, create an EB environment using the module

```hcl
##################################################
## Elastic Beanstalk config
##################################################
module "eb_env" {
  source = "github.com/digitregroup/terraform-elastic-beanstalk-php//eb-env"
  aws_region = var.aws_region

  # Application settings
  env = var.env
  service_name = var.service_name
  service_description = "My awesome php App"
  
  # PHP settings
  php_version = "7.0"
  document_root = "/public"
  memory_limit = "512M"
  zlib_php_compression = "Off"
  allow_url_fopen = "On"
  display_errors = "On"
  max_execution_time = "60"
  composer_options = ""
  
  # Instance settings
  instance_type = "t2.micro"
  min_instance = "2"
  max_instance = "4"

  # ELB
  enable_https = "false" # If set to true, you will need to add an ssl_certificate_id (see L70 in app/variables.tf)

  # Security
  vpc_id = "vpc-xxxxxxx"
  vpc_subnets = "subnet-xxxxxxx"
  elb_subnets = "subnet-xxxxxxx"
  security_groups = "sg-xxxxxxx"
}
```

### Link your domain using Route53

Add to the previous script the following lines:

```hcl
##################################################
## Route53 config
##################################################
module "app_dns" {
  source = "github.com/digitregroup/terraform-elastic-beanstalk-php//r53-alias"
  aws_region = var.aws_region

  domain = "example.io"
  domain_name = "my-app.example.io"
  eb_cname = module.eb_env.eb_cname
}
``` 

### Add an Elastic File System to your EB

* First, create the EFS before the Elastic Beanstalk Environment:
```hcl
##################################################
## AWS Elastic File System
##################################################
resource "aws_efs_file_system" "file_storage" {
  creation_token = "${var.service_name}-${var.env}"

  tags {
    Name = "${var.service_name}-${var.env}"
  }
}

resource "aws_efs_mount_target" "file_storage_target" {
  file_system_id  = aws_efs_file_system.file_storage.id
  subnet_id       = var.vpc_subnets
  security_groups = [var.security_groups]
}
```

* Then set `efs_id` & `efs_mount_directory` to the Elastic Beanstalk Environment module:
```hcl
##################################################
## Elastic Beanstalk config
##################################################
module "eb_env" {
  source = "github.com/digitregroup/terraform-elastic-beanstalk-php//eb-env"
  aws_region = var.aws_region

  # Application settings
  env = var.env
  service_name = var.service_name
  service_description = "My awesome php App"
  
  # PHP settings
  php_version = "7.0"
  document_root = "/public"
  memory_limit = "512M"
  zlib_php_compression = "Off"
  allow_url_fopen = "On"
  display_errors = "On"
  max_execution_time = "60"
  composer_options = ""
  
  # Instance settings
  instance_type = "t2.micro"
  min_instance = "2"
  max_instance = "4"

  # ELB
  enable_https = "false" # If set to true, you will need to add an ssl_certificate_id (see L70 in app/variables.tf)

  # Security
  vpc_id = "vpc-xxxxxxx"
  vpc_subnets = "subnet-xxxxxxx"
  elb_subnets = "subnet-xxxxxxx"
  security_groups = "sg-xxxxxxx"
  
  # EFS
  efs_id=aws_efs_file_system.file_storage.id
  efs_mount_directory="/var/app/efs"
}
```

* Finally, copy `.ebextensions/efs_mount.config` to your `.ebextensions` project directory and deploy using AWS EB CLI

### Example

* Take a look at [example.tf](./example.tf) for an example with Elastic Beanstalk and Route53.
* Take a look at [example_with_cloudfront.tf](./example_with_cloudfront.tf) for an example with Elastic Beanstalk, Cloudfront and Route53.

## Customize

Many options are available through variables. Feel free to look into `variables.tf` inside each module to see all parameters you can setup.

# Tips

Elastic Beanstalk PHP Tips:

* Install PDO DBLIB (PDO MSSQL Driver) for PHP 7.0: add the following lines to `.ebextensions/dblib.config`

```yaml
packages:
  yum:
    freetds: []
    freetds-devel: []
    php70-pdo-dblib: []
```

* Install PDO DBLIB (PDO MSSQL Driver) for PHP 5.6: add the following lines to `.ebextensions/dblib.config`

```yaml
packages:
  yum:
    freetds: []
    freetds-devel: []
    php56-mssql: []
```

## Terraform related documentation

* Elastic Beanstalk Application: https://www.terraform.io/docs/providers/aws/r/elastic_beanstalk_application.html
* Elastic Beanstalk Environment: https://www.terraform.io/docs/providers/aws/r/elastic_beanstalk_environment.html
* CloudFront: https://www.terraform.io/docs/providers/aws/r/cloudfront_distribution.html
* Route53: https://www.terraform.io/docs/providers/aws/d/route53_zone.html
