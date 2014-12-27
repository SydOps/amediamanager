amediamanager
=============
http://blogs.aws.amazon.com/application-management/post/Tx1NV26L8WNB0QS/Part-2-Develop-Deploy-and-Manage-for-Scale-with-Elastic-Beanstalk-and-CloudForma#

Part 2: Develop, Deploy, and Manage for Scale with Elastic Beanstalk and CloudFormation

April 14, 2014 Evan Brown 
Tags: Best practices How-to CloudFormation DDMSeries Elastic Beanstalk S3 VPC
Part 2: Elastic Beanstalk in VPC and Patterns for Authoring Multi-Region CloudFormation Templates

Welcome to the 2nd part of this 5-part series where we’ll cover best-practices and practical tips & tricks for developing, deploying, and managing a web application with an eye for application performance and operational efficiency using AWS CloudFormation and Elastic Beanstalk.

We’ll introduce a new part of the series as a blog post each Monday, and discuss the post as well as take questions during an Office Hours-style Hangout that Thursday. All application source and accompanying CloudFormation templates are available on GitHub at http://github.com/awslabs/amediamanager

Last week (blog post and Office Hours video) we introduced the aMediaManager application, all of its components, and then used CloudFormation and Elastic Beanstalk to deploy it into an EC2 Classic environment.

This week in the second part of the series we’re going to do a few things:

Use CloudFormation to design a VPC and update our Elastic Beanstalk application to deploy into that VPC
Explore how to make CloudFormation templates that lend themselves to multi-region deployments
The Hangout

We’ll be discussing this blog post - including your Q&A - during a live Office Hours Hangout at 9a Pacific on Thursday, April 17, 2014. Sign up at http://bit.ly/1hDcKaU.

Deploy the Application

The complete application source code and CloudFormation templates are Apache licensed and available on GitHub at http://github.com/awslabs/amediamanager.

An Important Note About Cost

All defaults in the CloudFormation template fall within the AWS Free Usage Tier. If your account is not eligible for the free usage tier, or if you use non-default values, you will be charged for the resources provisioned by this template. To delete the application, navigate to the AWS CloudFormation Managemenet Console, choose the amediamanager stack and click the Delete Stack button.

Deploy From the Console

If you’d like to follow along with a running version of the application, you can deploy it now using CloudFormation by clicking this link.

Deploy From the Command Line

To deploy from your command line, you’ll need the AWS CLI installed and configured, as well as a git client:

Download the source from GitHub:

$> git clone https://github.com/awslabs/amediamanager
Go into the source folder and copy the launch params template file:

$> cd amediamanager
$> cp cfn/vpc/launch-params.json.example cfn/vpc/launch-params.json
Edit cfn/vpc/launch-params.json, replacing YOUR_EC2_KEY_PAIR with the name of a real EC2 Key Pair in your account.

Also in cfn/vpc/launch-params.json, modify VPCAvailabilityZone1 and VPCAvailabilityZone2 depending on the region you’re deploying to.

Run a stack with the aws CLI:

$> aws cloudformation create-stack \
     --stack-name amediamanager \
     --template-body file://cfn/vpc/amm-master.cfn.json \
     --parameters file://cfn/vpc/launch-params.json \
     --capabilities CAPABILITY_IAM
About the Templates

Here’s a diagram illustrating all of the templates we’ll use to run the aMediaManager application in a VPC, including the order in which the stacks are created:



Let’s talk in a bit more detail about each of the templates (numbers in this list correspond to the number annotations in the above diagram):

amm-master.cfn.json: This is the parent template and it only defines three Resources: the embedded CloudFormation stacks amm-vpc.cfn.json, amm-elasticbeanstalk.cfn.json, and amm-resources.cfn.json. To run the entire application, run this stack.

amm-vpc.cfn.json: This template defines the VPC network topology, including an EC2 Instance for both Bastion and NAT services. The template also defines a Security Group. More on that in a bit. This stack outputs the ID of the VPC, Subnets, and Security Group it creates.

amm-resources.cfn.json: This template defines all of the dependencies for our application, including RDS databse, ElastiCache cluster, DynamoDB table, S3 bucket, IAM roles, etc. This stack takes as input parameters the VPC and Subnet IDs required to launch RDS and ElastiCache clusters into the VPC.

amm-elasticbeanstalk.cfn.json: This template defines the Elastic Beanstalk Application and Environment that runs our app code. It takes as inputs the IDs of its resource dependencies (DB hostname, S3 bucket name, etc), as well as the VPC ID, Subnets, and Security Group ID created by the VPC stack.

VPC in CloudFormation

We’re going to design a VPC with the following characteristics:

Multi-AZ
Public and Private Subnets
Both NAT and Bastion hosts
RDS, ElastiCache, and Elastic Beanstalk Instances in private subnet
ELB, NAT, and Bastion in public subnet
We’ll model the VPC in the amm-vpc.cfn.json template. That template defines a VPC with public and private subnets in 2 Availability Zones. Here’s a simple architecture diagram:



Parameterizing Availability Zones

When you create a VPC subnet you have to specify the Availability Zone that it’s associated with. CloudFormation does make it possible to dynamically select a subnet using the Fn::GetAZs and Fn::Select intrinsic functions:

"PublicSubnet1": {
  "Type": "AWS::EC2::Subnet",
  "Properties": {
    ...
    "AvailabilityZone": {
      "Fn::Select": ["0", { "Fn::GetAZs: "" }]
    },
    ...
  }
}
This approach has a downside. Fn::GetAZs returns every Availability Zone for your account, but it is possible that - for accounts that existed before VPC was introduced - an Availability Zone may exist but is not VPC enabled. In my case (where my account is ~5 years old), I have 5 Availability Zones but 1 of them isn’t available for VPC.

Rather than dynamically selecting from a list, amm-vpc.cfn.json requires you to explicitly provide the 2 Availability Zones that should be used by defining them as Parameters:

"Parameters": {
  ...
  "VPCAvailabilityZone1": {
    "Description": "One of two Availability Zones that will be used to create subnets.",
    "Type": "String",
    "MinLength": "1",
    "MaxLength": "255"
  },
  "VPCAvailabilityZone2": {
    "Description": "Two of two Availability Zones that will be used to create subnets. Must be different than VPCAvailabilityZone2.",
    "Type": "String",
    "MinLength": "1",
    "MaxLength": "255"
  }
  ...
}
And when I launch the stack using the CLI, the parameter values in my cfn/vpc/launch-params.json file look like:

[
  {
    "ParameterKey": "KeyName",
    "ParameterValue": "evbrown-amazon"
  },
  {
    "ParameterKey": "VPCAvailabilityZone1",
    "ParameterValue": "us-east-1a"
  },
  {
    "ParameterKey": "VPCAvailabilityZone2",
    "ParameterValue": "us-east-1c"
  }
]
More Info on VPC and CloudFormation

We covered more examples of building VPCs with CloudFormation a few months ago in this post: https://github.com/evandbrown/aws-hangouts/tree/master/20140213_cfn. In addition to code samples, the post includes a video discussion with Q&A. Hopefully you find it helpful!

Elastic Beanstalk in VPC

When amm-master.cfn.json creates the amm-elasticbeanstalk.cfn.json stack, it has already created the VPC and Resources stack and has access to their outputs. Notably, the VPC stack outputs the IDs of both of the public and private subnets it created.

amm-master.cfn.json joins these values with a comma (using Fn::Join) and passes them to the relevant amm-elasticbeanstalk.cfn.json input parameters:

"App1" : {
  "Type" : "AWS::CloudFormation::Stack",
  "Properties" : {
    "Parameters" : {
      ...
      "VPCId" : { "Fn::GetAtt" : ["VPCScaffold", "Outputs.VPCId"] },
      "PrivateSubnets" : {
        "Fn::Join": [",", [{ "Fn::GetAtt" : ["VPCScaffold", "Outputs.PrivateSubnet1"] }, { "Fn::GetAtt" : ["VPCScaffold", "Outputs.PrivateSubnet2"] }]]
      },
      "PublicSubnets" : {
        "Fn::Join": [",", [{ "Fn::GetAtt" : ["VPCScaffold", "Outputs.PublicSubnet1"] }, { "Fn::GetAtt" : ["VPCScaffold", "Outputs.PublicSubnet2"] }]]
      }
    }
  }
}
You can see this code highlighted on GitHub: https://github.com/awslabs/amediamanager/blob/master/cfn/vpc/amm-master.cfn.json#L165-L171

The amm-elasticbeanstalk.cfn.json template then provides these values - along with the VPC id - to the Elastic Beanstalk Application’s Default Configuration via Option Settings:

"Application": {
  "Type": "AWS::ElasticBeanstalk::Application",
  "Properties": {
    ...
    "ConfigurationTemplates": [{
      ...
      "OptionSettings": [
      ...
      {
        "Namespace": "aws:ec2:vpc",
        "OptionName": "VPCId",
        "Value": { "Ref": "VPCId"
        }
      },
      {
        "Namespace": "aws:ec2:vpc",
        "OptionName": "Subnets",
        "Value": { "Ref" : "PrivateSubnets" }
      },
      {
        "Namespace": "aws:ec2:vpc",
        "OptionName": "ELBSubnets",
        "Value": { "Ref" : "PublicSubnets" }
      },
      ...]
    }],
    ...
  }
},
...
From this snippet we can see that Elastic Beanstalk cares about 3 things when it comes to VPC:

What is the VPC the application will run in?
What are the (public) subnets the ELB should be deployed to?
What are the (private) subnets the EC2 Instances should be deployed to?
You can see this code highlighted on GitHub: https://github.com/awslabs/amediamanager/blob/master/cfn/vpc/amm-elasticbeanstalk.cfn.json#L117-L132

About That Security Group

We saw earlier that amm-vpc.cfn.json created a Security Group and output its ID (in addition to the expected VPC resources):



But why? Well, if you fast-forward to when the Elastic Beanstalk application is created (and remember, that’s the last step/stack in the amm-master.cfn.json parent stack), the EC2 Instances created by Elastic Beanstalk need a few things from a network security perspective:

They need to be able to access the NAT Instance for outbound Internet access
The Bastion Instance needs to be able to access them for management purposes
They also need to be able to access the RDS Database
To address these 3 concerns, we create a Security Group along with the VPC stack. In that same stack we define authorizations that allow that SG to communicate with the NAT and Bastion Instances. Then in the amm-resources.cfn.json stack where we declare the RDS database, we input the ID of this SG and authorize it to the RDS SG. Finally, we provide the SG - which is now authorized to communicate with the NAT, Bastion, and RDS hosts - to the Elastic Beanstalk stack, and tell Elastic Beanstalk to use that SG for the Instances is launches:

"Application": {
  "Type": "AWS::ElasticBeanstalk::Application",
  "Properties": {
    ...
    "ConfigurationTemplates": [{
      ...
      "OptionSettings": [
      ...
      {
        "Namespace": "aws:autoscaling:launchconfiguration",
        "OptionName": "SecurityGroups",
        "Value": { "Ref": "InstanceSecurityGroup" }
      },
      ...
      ]
    }],
    ...
  }
},
Patterns for Authoring Multi-Region CloudFormation Templates

Now let’s explore how to use Mappings and S3 bucket-naming schemes to build templates we can easily run in different AWS Regions.

Mapping AZs: An Alternative to Parameters

Parameterizing AZs as described earlier is a good solution for a sample app like this that is being shared broadly and publicly: I don’t know the details of each of your accounts and which AZ may not be VPC enabled, leaving that to parameters makes the template maximally flexible.

But let’s imagine we’re really building this aMediaManager application on a team. We’d likely be sharing an AWS Account and know the Availability Zones we want to use in each Region. In that case, defining a Mapping of Region->[AvailabilityZone1, AvailabilityZone2] in the template would allow us to enforce some consistency and not require params.

I could use a Mapping named RegionToVpc mapping like the following to identify the correct AZs in the us-east-1, us-west-1, and us-west-2 regions:

"Mappings": {
  ...
  "RegionToVpc" : {
    "us-east-1": {
      "AvailabilityZone1": "us-east-1a",
      "AvailabilityZone2": "us-east-1c"
    },
    "us-west-1": {
      "AvailabilityZone1": "us-west-1b",
      "AvailabilityZone2": "us-west-1c"
    },
    "us-west-2": {
      "AvailabilityZone1": "us-west-2a",
      "AvailabilityZone2": "us-west-2b"
    }
  }
  ...
}
I can then dynamically look up the correct Availability Zone for my subnet using the Fn::FindInMap function and { "Ref": "AWS::Region"} (which returns the region name the stack is running in):

"PublicSubnet1": {
  "Type": "AWS::EC2::Subnet",
  "Properties": {
    ...
    "AvailabilityZone": {
      "Fn::FindInMap": ["RegionToVpc", { "Ref" : "AWS::Region" }, "AvailabilityZone1"]
    },
    ...
  }
}
We talked in detail about other uses for Mappings - especially as they relate to VPC and multi-region templates - in this post a few months ago.

Extra Credit If you’re looking to sharpen your CloudFormation skills, modify the amm-vpc.cfn.json template to use Mappings as described, rather than Parameters.

S3 Buckets, Regions, and Naming Conventions

S3 Buckets play a really important role in nearly every aspect our application. Let’s talk about where we use S3 Buckets and how we approach naming them:

CloudFormation Template Storage

Even though we can upload and run the amm-master.cfn.json file from our local machine to kick off our stack, remember that template defines other nested CloudFormation stacks, and you have to provide a TemplateURL for each stack.

Let’s look at a snippet of amm-master.cfn.json for the VPC version of aMediamanager, see how we define the VPC nested stack:

"Resources": {
  "VPCScaffold" : {
    "Type" : "AWS::CloudFormation::Stack",
    "Properties" : {
      "TemplateURL" : { "Fn::Join" : ["", [ "http://", { "Ref" : "AssetsBucketPrefix" }, { "Ref" : "AWS::Region" }, ".s3.amazonaws.com/", { "Ref" : "VPCTemplateKey" }]]},
      "Parameters" : {
        ...
      }
    }
  },
  ...
You can see right away that we didn’t hard-code the TemplateURL value (e.g., https://my-bucket.s3.amazonaws.com/path/to/template/file.cfn.json). Rather we used the Fn::Join intrinsic function, several parameters, and AWS::Region to construct the URL by convention.

And that convention is really straightforward in this example. Let’s dissect TemplateURL and see what we’re Fn::Joining together:

{ "Ref" : "AssetsBucketPrefix" }: This is a Parameter in the amm-master.cfn.json template. I’ve given it a default value of amm-, which is an abbreviation of the aMediManager application name. So right away we see the S3 bucket we’re using has an association with a particular application. Our TemplateURL at this point is http://amm-
{ "Ref" : "AWS::Region" }: Next we concatenate the current AWS Region name. This means we’ll have an S3 Bucket in each Region we plan to run this application. If we’re running our application in us-east–1, our TemplateURL would now be http://amm-us-east-1
.s3.amazonaws.com/: This is the univeral S3 endpoint DNS. Our TemplateURL becomes http://amm-us-east-1.s3.amazonaws.com/
{ "Ref" : "VPCTemplateKey" }: For the grand finale, we reference a Parameter which contains the path (i.e., S3 Object Key) to the CloudFormation template that declares our VPC. I chose to store that template with the public/vpc/amm-vpc.cfn.json key/file name and set it as the default value for that Parameter (which, of course, you can override). That means the final value for TemplateURL is http://amm-us-east-1.s3.amazonaws.com/public/vpc/amm-vpc.cfn.json
Coming Up: Part 3

First, don’t forget to join us for the live Office Hours Hangout later this week (or view the recording if it’s past April 17 2014 and you don’t have a time machine).

In Part 3 of this series (blog post and Office Hours links forthcoming at http://blogs.aws.amazon.com/application-management) we’ll dive into best practices for writing application code that will run easily in any AWS Region, and discuss how (and why) we used Elastic Beanstalk and S3 to store and manage our application’s configuration.

