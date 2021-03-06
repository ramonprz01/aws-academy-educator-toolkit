# CloudFormation Demo

In this lab you incrementally create a cloudformation template to launch an EC2 instance in a public subnet. You will then make a number of additional improvements to the template. The goal is you will be more confident with writing your own templates from scratch.

## Prep

- Start Lab

If using Cloud9 it will be easier to update the cloudformation stack using the command line and pass the template in as a file. 

Alternatively you can copy your template file with each update to an S3 bucket and update from there using the console.

```bash
aws s3 cp template.yaml s3://<BUCKET>
```

## Goal

TODO

## Step 1

CloudFormation templates start with

- a format version
- optional description
- at least one resource
- comments start with a `#`

The obvious resource to create first is a VPC.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc.html>
- For the VPC we need to specify the type `AWS::EC2::VPC`
- It has a number of parameters. If we look at the parameter descriptions, only the `CidrBlock` is mandatory.

template.yaml:

```yaml
---
AWSTemplateFormatVersion: "2010-09-09"

# Step 1 - Create VPC

Description: Writing your first cloudformation template

# Metadata:

# Parameters:

# Mappings:

# Conditions:

# Transform:

Resources:

# VPC

  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

# Outputs:
```

Create the stack in the console or with

```bash
aws cloudformation create-stack \
    --stack-name cfn-demo \
    --template-body file://template.yaml
```

_Q: How did cloudformation know which region to put the VPC?_

## Step 2

If you look at the VPC you just created you can see it is missing a name. By tagging resources properly it helps us understand what they are for, who owns them, etc. We can also automatic tasks based on a resource's tag.

Go back to the VPC documentation and you can see it describes how to add tags, and using the stack name is good way to keep the name unique.

To refer to the parameter in the resource definition, we are will use the `!Ref` intrinsic function

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html>

The stack name is a psuedo parameter that we can always access in our stack

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html>
    - `!Ref "AWS::StackName"`

Update your template to this:

```yaml
---
AWSTemplateFormatVersion: "2010-09-09"

# Step 2 - Add VPC Name

Description: Writing your first cloudformation template

# Metadata:

# Parameters:

# Mappings:

# Conditions:

# Transform:

Resources:

# VPC

  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Ref "AWS::StackName"

# Outputs:
```

_Q: If we update the stack with a new tag, what will happen to the existing VPC?_

Update your stack with the new template:

```bash
aws cloudformation update-stack \
    --stack-name cfn-demo \
    --template-body file://template.yaml
```

Confirm the VPC name has been updated.

## Step 3

If we plan to reuse this template whenever we want to create a network and an instance, we will need to be able to configure settings. To do this we use parameters.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html>
- Add a parameter to your template for the VPC CidrBlock and make the default `10.0.0.0/16`. 
- Optional: To help with input validation you can also specify `AllowedPattern: ^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])(\/(1[6-9]|2[0-8]))$`

```yaml
Parameters:

  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
    AllowedPattern: ^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])(\/(1[6-9]|2[0-8]))$
```

Use `!Ref` again to refer to your parameter in the VPC resource.

[Solution](./3-template.yaml)

Update the stack with your new template, you should see no change. In fact if you do it in the console you might get an error because there is nothing to update.

Regardless of how you update the stack, look at your stack in the console, pressing the `Update` button and `Use current template`.

## Step 4

Lets start building other resources in our VPC, starting with the internet gateway.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-internetgateway.html>
  - Give the internet gateway a name
- Strangely you can't specify the VPC for the Internet Gateway, you then also have to attach it
  - <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-vpc-gateway-attachment.html>

[Solution](./4-template.yaml)

Update your stack with your change.

_Q: Why keep updating the stack with each change to the template?_

## Step 5

The next thing to build for our network is a public route table and route to the internet.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route-table.html>
  - Because we will also have a private route table, we need to add more to the name than just the stack name
  - The `!Sub` function can be used to form strings and substitute parameters
    - `!Sub "${AWS::StackName}-public"`
- Also remember to add a route to the internet
  - <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-route.html>

[Solution](./5-template.yaml)

This time when updating the stack, lets watch the progress:

```bash
aws cloudformation update-stack \
    --stack-name cfn-demo \
    --template-body file://template.yaml

watch -n 5 -d \
    aws cloudformation describe-stack-resources \
    --stack-name cfn-demo \
    --query 'StackResources[*].[ResourceType,ResourceStatus]' \
    --output table
```

This will update a table showing the state of the resources in the stack. Press `ctrl-c` when done.

## Step 6

Now we have a public route table we can create a public subnet. Looking at the documentation for a subnet you will see that `AvailabilityZone` is a mandatory field, but we didn't even specify the region with the VPC.

We will create our own mappings when we create the instance, but we can use the `!GetAZs` intrinsic function to get the list of availability zones in this region, and the `!Select` function to extract the first in the list.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getavailabilityzones.html>
- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-select.html>
- `!Select [ 0, !GetAZs ]`

We also need to specify the CIDR block for the VPC. Since we made the CIDR range for the VPC a parameter we can't hardcode the subnet CIDR block. The simplest approach might be to add another parameter for the public subnet CIDR, but we can use another intrinsic function to calculate the CIDR range for us.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-cidr.html>
  - `!Select [ 0, !Cidr [ !GetAtt VPC.CidrBlock, 20, 8 ]]`

Note that we fetch the CIDR block from the VPC that was created rather than use the user parameter, so this code should not be affected by changes to how we specify the VPC CIDR. To get the CIDR block from the VPC we can't use `!Ref` as that returns the VPC ID. Instead we use `!GetAtt` to get another attribute from the VPC. The full list of attributes for each resource is included in the cloudformation documentation.

Another way to build strings is to use `!Join`. Here is another way we could generate the subnet name:

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-join.html>
- `!Join ['-', ["public", !Ref "AWS::StackName", !Select [ 0, !GetAZs ]]]`

[Solution](./6-template.yaml)

Update your stack and check your subnet in the console.

## Step 7

What route table is your subnet associated with? Associate the public route table we created with the public subnet.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ec2-subnet-route-table-assoc.html>

[Solution](./07-template.yaml)

Update your stack.

_Q: Could we have created the public subnet prior to the public route table?_

## Step 8

Before we can create an instance in our subnet, we will need a security group with at least port 80 open.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group.html>
  - SecurityGroup type in cloudformation includes properties for its name and description. You can also tag the security group.
- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-security-group-ingress.html>

[Solution](./08-template.yaml)

Update the stack and check the security group in the console.

## Step 9

Have a look at the documentation for creating an EC2 instance.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html>

There are a lot more properties here, but most are optional. The properties we need to worry about are:

- InstanceType
- KeyName
- SecurityGroupIds
- SubnetId
- UserData
- Tags

`SubnetId` and `Tags` should be straightforward. `SecurityGroupIds` is a little more complicated since it is expecting a list even though we only have one:

```yaml
      SecurityGroupIds:
        - !Ref SecurityGroup
```

For the `InstanceType` we can specify `t2.micro`, but it would be best if the AMI and Access Key were not hardcoded but specified as parameters. With the CPC CIDR we used a string, but for the AMI and Access key we can use different parameter types that help pre-fill the values.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html#aws-specific-parameter-types>

For the image ID we want to fetch the latest AMI ID for a particular image, so specify this path as the default:

```yaml
  AMI:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
    Description: Parameter store path to AMI ID
```

Last but not least is the user data. You can provide a script using the follow syntax, in this case to install and run a web server:

```yaml
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          # Install Apache Web Server and PHP
          yum install -y httpd
          # Add a message to the web server
          echo "What are you doing Dave?" > /var/www/html/index.html
          # Turn on web server
          chkconfig httpd on
          service httpd start
```

Try to update your stack. In the console you should be prompted to select an access key that you already have in the account. If updating using the CLI you will get an error for this parameter has no value. Adjust the command line arguments so you pass in the access key name, if you are running in Vocaruem it will be `vockey`:

```bash
aws cloudformation update-stack \
    --stack-name cfn-demo \
    --template-body file://template.yaml \
    --parameters ParameterKey=KeyName,ParameterValue=vockey
```

[Solution](./09-template.yaml)

Update your stack and watch the EC2 instance start in console.

_Q: The cloudformation template completes before the instance has finished starting. What if the next resource we create depends on the instance running?_

## Step 10

At this point in the lab we often instruct students to look in the Vocareum lab details for the IP address of the instance, or look in the console. For resource attributes the template can explicitly output values that we need from the resources in the stack.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html>

Output the instance IP address. For bonus points also output the URL for the web server running on the instance.

[Solution](./10-template.yaml)

```bash
$ aws cloudformation describe-stacks \
    --stack cfn-demo \
    --query 'Stacks[*].Outputs[*]' \
    --output table

---------------------------------------------------------------------
|                          DescribeStacks                           |
+-----------------------------+------------+------------------------+
|         Description         | OutputKey  |      OutputValue       |
+-----------------------------+------------+------------------------+
|  Instance Public IP Address |  PublicIP  |  34.205.28.192         |
|  Instance Web Site          |  WebSite   |  http://34.205.28.192  |
+-----------------------------+------------+------------------------+
```

## Step 11

In Step 9 the instane type was hardcoded, but if we wanted to use different instance sizes depending on a parameter we can specify? A common way to do this is to add a map.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/mappings-section-structure.html>

A map allows us to hardcode a set of values but select the appropriate value based on the region we are in or a parameter the user sets.

- Add a parameter that allows the user to select from `Micro`, `Small` and `Large`, with `Micro` as the default
- Add a mapping for those values to different instance sizes

```yaml
Mappings:

  InstanceDetails:
    Micro:
      Type: t2.micro
    Small:
      Type: t3.small
    Large:
      Type: m5.large
```

Now instead of hard coding t2.micro, use `!FindinMap` to look up the type of instance based on the size parameter.

[Solution](./11-template.yaml)

If you get an error that there is nothing to update that means your change has worked, the map lookup resulted in the same size as the running instance, a `t2.micro`.

## Step 12

We have web access to the instance, but what about SSH? An access key was installed but we never added SSH to the security group. How can we specify the CIDR range that we want to permit for the SSH client, ideally this should not `0.0.0.0/0`, and how can we make SSH access optional?

- Create a new parameter that takes the permitted CIDR range.

```yaml
  SSHCidr:
    Type: String
    Default: ""
    Description: Leave empty for no SSH access, otherwise specify trusted CIDR
```

- Create a condition so that if the CIDR range is empty, do not add SSH access to the instance
  - <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html>

[Solution](./12-template.yaml)

Update your template passing in your external IPv4 address and confirm you can ssh to the instance.

```bash
aws cloudformation update-stack \
    --stack-name cfn-demo \
    --template-body file://template.yaml \
    --parameters ParameterKey=KeyName,ParameterValue=vockey ParameterKey=SSHCidr,ParameterValue=<IP Address>
```

## Step 13

There are now 5 parameters users can specify. As the list of parameters grow they can be harder for the user to understand. You can provide a `Description` with the parameters, but you can provide additional meta data to customer the layout of the parameters.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudformation-interface.html>

Create two `ParameterGroups` and `ParameterLabels` for the parameters in your template.

[Solution](./13-template.yaml)

Because this step creates no new resources, complete the next step before testing these changes.

# Step 14

We create a simple VPC with just one public subnet, what about a private subnet? First think about the components you will need and what order you need to create these resources:

- Private Route Table
- Private Route
- Private Subnet
- NAT Gateway
- Elastic IP for the NAT Gateway

The order of resources in the template does not influence the order in which they are created. CloudFormation builds a dependency graph and determines the order resources can be created based on how the resources reference each other. This allows it to create multiple resources in parallel if they do no depend on each other.

In defining these resources you will find you will need more resources, and you will also need to tell CloudFormation about a dependency because a resource (NAT Gateway) depends on the Internet Gateway being attached to the VPC, but there is no reference to the Internet Gateway Attachment in its properties. You will not encounter this race because your Internet Gateway is already in place, but if you were running this template from scratch, you would risk a random error on some runs.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-dependson.html>

For the private subnet, use `!Cidr` again but this time select a different range.

[Solution](./14-template.yaml)

## Step 15

It is scarily easy to destroy all the resources you just built, but **before you run this** how can we protect important resources from deletion?

```bash
# Don't do this just yet
aws cloudformation delete-stack --stack-name cfn-demo
```

Lets add an EBS volume to the instance, but configure the volume to create a snapshot when the template deletes.

- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html>
- <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-ebs-volume.html>

For the size of the volume, extend the map so that different size instances get a different size volume attached. Can you attach the volume without rebuilding the instance?

[Solution](./15-template.yaml)

Update your stack to add the volume to your instance.

See <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html> for other ways you can protect resources from accidental deletion.

## Step 16

In the console open up CloudFormation designer and view your template graphically.

Note: At time of writing, the design does not support !Cidr so you may need to replace the two times this function is used and hardcode CIDR ranges for the subnets to get the designer to work.

## Additional Steps

If you want to keep going, you can try...

- Use the help scripts to wait for the EC2 instance to finish startup
  - <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-helper-scripts-reference.html>
- Create public and private subnets across 2 or more availability zones
- Split the template in a Master, Network and Application templates.
  - <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html#nested>
- Use exports to reference values across stacks
- Create an application load balancer, target group, autoscaling group and launch config to run multiple web servers
- Use stacksets to deploy to multiple regions

## Cleanup

```bash
aws cloudformation delete-stack --stack-name cfn-demo
```

Navigate to EC2->Snapshots and confirm a snapshot was generated for the volume.

## License Summary

This sample code is made available under the MIT-0 license. See the LICENSE file.