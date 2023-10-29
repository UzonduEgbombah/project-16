# project-16
## AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM PART 1

#### Prerequisites before you begin writing Terraform code

- Create an IAM user, name it terraform (ensure that the user has only programatic access to your AWS account) and grant this user AdministratorAccess permissions.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/a15266df-ab02-4455-a9c1-4b22bf953740)

- Copy the secret access key and access key ID. Save them in a notepad temporarily.

- Configure programmatic access from your workstation to connect to AWS using the access keys copied above and a Python SDK (boto3). You must have Python 3.6 or higher on your workstation.
If you are on Windows, use gitbash, if you are on a Mac, you can simply open a terminal. Read here to configure the Python SDK properly.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/ecbb8c23-c5cd-4357-9335-702595f6239a)


- For easier authentication configuration – use AWS CLI with aws configure command.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/6d6dbf70-83d1-45d7-bdc5-0b81a34adb90)

Create an S3 bucket to store Terraform state file. You can name it something like <yourname>-dev-terraform-bucket

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/9967c70d-adf5-41eb-944b-2a6d13933ae2)

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/555c3fad-34be-483c-a77e-faacf879760c)

You shall see your previously created S3 bucket name – <yourname>-dev-terraform-bucket

## VPC | SUBNETS | SECURITY GROUPS
Let us create a directory structure
Open your Visual Studio Code and:

- Create a folder called PBL
- Create a file in the folder, name it main.tf
Your setup should look like this.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/156b00dd-d787-4c8f-ab7a-f242320679b5)

#### Provider and VPC resource section
Set up Terraform CLI as per this instruction.

- Add AWS as a provider, and a resource to create a VPC in the main.tf file.
- Provider block informs Terraform that we intend to build infrastructure within AWS.
- Resource block will create a VPC.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/f964d74d-04f0-477a-bd5c-097e11a12b7e)

#### Note: 
You can change the configuration above to create your VPC in other region that is closer to you. The same applies to all configuration snippets that will follow.

The next thing we need to do, is to download necessary plugins for Terraform to work. These plugins are used by providers and provisioners. At this stage, we only have provider in our main.tf file. So, Terraform will just download plugin for AWS provider.
Lets accomplish this with terraform init command as seen in the below demonstration.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/77595a69-dd42-49dc-a9b6-532208f3ec71)

#### Observations:

A new file is created terraform.tfstate This is how Terraform keeps itself up to date with the exact state of the infrastructure. It reads this file to know what already exists, what should be added, or destroyed based on the entire terraform code that is being developed.
If you also observed closely, you would realize that another file gets created during planning and apply. But this file gets deleted immediately. terraform.tfstate.lock.info This is what Terraform uses to track, who is running its code against the infrastructure at any point in time. This is very important for teams working on the same Terraform repository at the same time. The lock prevents a user from executing Terraform configuration against the same infrastructure when another user is doing the same – it allows to avoid duplicates and conflicts.

#### Subnets resource section
According to our architectural design, we require 6 subnets:

- 2 public
- 2 private for webservers
- 2 private for data layer
Let us create the first 2 public subnets.

Add below configuration to the main.tf file:

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/4a51a6dd-00ef-457e-b601-927c220bed5c)

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/13b6346c-dd45-451f-aeaf-162f4d1b6a6e)

We are creating 2 subnets, therefore declaring 2 resource blocks – one for each of the subnets.
We are using the vpc_id argument to interpolate the value of the VPC id by setting it to aws_vpc.main.id. This way, Terraform knows inside which VPC to create the subnet.
Run terraform plan and terraform apply

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/3984eaf7-ac78-424c-9f6d-dfbbb7033eaa)

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/34f944cd-6824-446a-b0e9-d59369d4356c)

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/c21f59cc-3f05-469b-b353-9c6c96892e2b)


Observations:

Hard coded values: Remember our best practice hint from the beginning? Both the availability_zone and cidr_block arguments are hard coded. We should always endeavour to make our work dynamic.
Multiple Resource Blocks: Notice that we have declared multiple resource blocks for each subnet in the code. This is bad coding practice. We need to create a single resource block that can dynamically create resources without specifying multiple blocks. Imagine if we wanted to create 10 subnets, our code would look very clumsy. So, we need to optimize this by introducing a count argument.
Now let us improve our code by refactoring it.

First, destroy the current infrastructure. Since we are still in development, this is totally fine. Otherwise, DO NOT DESTROY an infrastructure that has been deployed to production.

To destroy whatever has been created run terraform destroy command, and type yes after evaluating the plan.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/23ca5c38-29cd-412b-ae5f-57013cc1d570)

## FIXING THE PROBLEMS BY CODE REFACTORING

##### Fixing Hard Coded Values:
We will introduce variables, and remove hard coding.
Starting with the provider block, declare a variable named region, give it a default value, and update the provider section by referring to the declared variable.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/e69eeb5c-8b7c-4b05-b53c-04a134ec2fbd)

- Do the same to cidr value in the vpc block, and all the other arguments.


##### Fixing multiple resource blocks:
This is where things become a little tricky. It’s not complex, we are just going to introduce some interesting concepts. Loops & Data sources
Terraform has a functionality that allows us to pull data which exposes information to us. For example, every region has Availability Zones (AZ). Different regions have from 2 to 4 Availability Zones. With over 20 geographic regions and over 70 AZs served by AWS, it is impossible to keep up with the latest information by hard coding the names of AZs. Hence, we will explore the use of Terraform’s Data Sources to fetch information outside of Terraform. In this case, from AWS

Let us fetch Availability zones from AWS, and replace the hard coded value in the subnet’s availability_zone section.

- To make use of this new data resource, we will need to introduce a count argument in the subnet block: Something like this.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/6bcfc682-94b3-4040-bda4-a6f94f0b34b6)

Let us quickly understand what is going on here.

- The count tells us that we need 2 subnets. Therefore, Terraform will invoke a loop to create 2 subnets.
- The data resource will return a list object that contains a list of AZs. Internally, Terraform will receive the data like this

Each of them is an index, the first one is index 0, while the other is index 1. If the data returned had more than 2 records, then the index numbers would continue to increment.

Therefore, each time Terraform goes into a loop to create a subnet, it must be created in the retrieved AZ from the list. Each loop will need the index number to determine what AZ the subnet will be created. That is why we have data.aws_availability_zones.available.names[count.index] as the value for availability_zone. When the first loop runs, the first index will be 0, therefore the AZ will be eu-central-1a. The pattern will repeat for the second loop.

But we still have a problem. If we run Terraform with this configuration, it may succeed for the first time, but by the time it goes into the second loop, it will fail because we still have cidr_block hard coded. The same cidr_block cannot be created twice within the same VPC. So, we have a little more work to do.

#### Let’s make cidr_block dynamic.
We will introduce a function cidrsubnet() to make this happen. It accepts 3 parameters. Let us use it first by updating the configuration, then we will explore its internals.

A closer look at cidrsubnet – this function works like an algorithm to dynamically create a subnet CIDR per AZ. Regardless of the number of subnets created, it takes care of the cidr value per subnet.

Its parameters are cidrsubnet(prefix, newbits, netnum)

The prefix parameter must be given in CIDR notation, same as for VPC.
The newbits parameter is the number of additional bits with which to extend the prefix. For example, if given a prefix ending with /16 and a newbits value of 4, the resulting subnet address will have length /20
The netnum parameter is a whole number that can be represented as a binary integer with no more than newbits binary digits, which will be used to populate the additional bits added to the prefix
You can experiment how this works by entering the terraform console and keep changing the figures to see the output.

On the terminal, run terraform console
type cidrsubnet("172.16.0.0/16", 4, 0)
Hit enter
See the output
Keep change the numbers and see what happens.
To get out of the console, type exit

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/40a2059a-c447-4007-bc96-e17bd6a747ab)

#### The final problem to solve is removing hard coded count value.
If we cannot hard code a value we want, then we will need a way to dynamically provide the value based on some input. Since the data resource returns all the AZs within a region, it makes sense to count the number of AZs returned and pass that number to the count argument.

To do this, we can introuduce length() function, which basically determines the length of a given list, map, or string.

Since data.aws_availability_zones.available.names returns a list like ["eu-central-1a", "eu-central-1b", "eu-central-1c"] we can pass it into a lenght function and get number of the AZs.

length(["eu-central-1a", "eu-central-1b", "eu-central-1c"])

Open up terraform console and try it
![](https://github.com/UzonduEgbombah/project-16/assets/137091610/64bd746c-3f75-4474-a6fa-2ece04ab70bf)

#### Now we can simply update the public subnet block like this

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/c4132f3b-810d-45e2-854c-22ee140328f4)

#### Observations:

- What we have now, is sufficient to create the subnet resource required. But if you observe, it is not satisfying our business requirement of just 2 subnets. The length function will return number 3 to the count argument, but what we actually need is 2.
Now, let us fix this.

- Declare a variable to store the desired number of public subnets, and set the default value

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/97019be5-f089-42f8-a7ab-c0ab33993fb5)

- Next, update the count argument with a condition. Terraform needs to check first if there is a desired number of subnets. Otherwise, use the data returned by the lenght function. See how that is presented below.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/fb5f4433-4dad-4440-9e1b-1d92c5594c06)

#### Now lets break it down:

The first part var.preferred_number_of_public_subnets == null checks if the value of the variable is set to null or has some value defined.
The second part ? and length(data.aws_availability_zones.available.names) means, if the first part is true, then use this. In other words, if preferred number of public subnets is null (Or not known) then set the value to the data returned by lenght function.
The third part : and var.preferred_number_of_public_subnets means, if the first condition is false, i.e preferred number of public subnets is not null then set the value to whatever is definied in var.preferred_number_of_public_subnets

## INTRODUCING VARIABLES.TF & TERRAFORM.TFVARS

Instead of havng a long lisf of variables in main.tf file, we can actually make our code a lot more readable and better structured by moving out some parts of the configuration content to other files.

We will put all variable declarations in a separate file
And provide non default values to each of them
- Create a new file and name it variables.tf
- Copy all the variable declarations into the new file.
- Create another file, name it terraform.tfvars
- Set values for each of the variables.

## variables.tf

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/1e7d3e6d-73c7-43cd-b7ad-7da906a5c8f3)


## terraform.tfvars

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/839cf16b-8c7c-4e17-b5f7-9a448db97a3a)

You should also have this file structure in the PBL folder.

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/dd52dbf3-9d01-4c60-bd07-8957f283f0ef)

## Run terraform plan and ensure everything works

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/853c7b3b-00cc-44db-8ae6-c48af9143d41)

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/db34bc4e-ff2d-4e64-bf5a-7b6b59849e2d)

![](https://github.com/UzonduEgbombah/project-16/assets/137091610/8dd215e5-7336-403e-8a4b-112c955c34f9)




