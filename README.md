This is demo project to show some AWS Cloudformation capabilities:

1. Adding tags
    You can add tags to AWS resources when you create those resources. Tags can be added as part of Cloudformation template, or you can add tag during the resource creation.
    The benefits of adding tags during resource creation is to make sure that required tags are created according to company tagging policies, especially for cost and utilization tracking. Please check AWS cost allocation tag at https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html
    You can use external property file to define those required tags and include them when you run Cloudformation cli to create resources. It is recommended to include tagging in your CI/CD pipeline, so those tags can be controlled at organization level.

    In my first example, I have those tags defined in stack-tags.json which is located in config folder.

    When running Cloudformation cli, you need specify this file with --tags, you can use similar approach for parameters, like AMI id, region or instance type. Parameters are defined in stack-params.json

2. Template validation
    You can use cfn-lint to validate the format and dependency of your Cloudformation template

    If you have company rules for compliance purpose, please loock at AWS CloudFormation Guard project at https://github.com/aws-cloudformation/cloudformation-guard, this is particular useful if you want to avoid resource from being created if they are not compliant, for example:
    if you execute (need to install cfn-guard first):
    cfn-guard validate --rules cfn-guard/required-tag-rule.guard --data ./templates/demo-ec2.json

    This will validate if EBS has required “usage” tag


    Please check this blog for more information about Cloudformation Guard:
    https://aws.amazon.com/blogs/mt/introducing-aws-cloudformation-guard-2-0/

3. Resource Update behavior
    Some time when attribute of a resource is being modified, the resource need to be destroyed and recreated. If you'd like to monitor this behavior, you should create changeset first before apply your update.
    I will show you how to do it in my example

    Before going to the example, I created a script that can perform list of actions

    upload: upload template to AWS S3 bucket
    create: create resource using template
    update: create changeset when you make modification to your resource
    list: list changeset of your Stackset
    view: view changeset of your Stackset
    apply: apply changeset to your Stackset

    Script also execute cfn-lint to validate template before any other activities.

    Example 1

    Add tags:

    #first step is to upload your Cloudformation template to S3 
    ./cfnctl upload

    #Create resource
    ./cfnctl create

    You should get the response like following:
    {
        "StackId": "arn:aws:cloudformation:us-east-1:xxxxxxxxxxxxx:stack/ec2-demo-stack/9a686cf0-6c5c-11ee-97af-0ecf5ea19ac5"
    }
    You will need Stack Id for the following actions:

    After stack creation is completed, you then check all resources defined in the stack, they all should have tags defined in stack-tags.json. This the benefits of using external file, so you don't have to add tags to every individual resource in the template.

    Example 2

    Review Changeset
    Make some changes to your Cloudformation template, 
    1. add new rule in security group
    2. Change one of tags in stack-tags.json

    then
    ./cfnctl upload
    ./cfnctl update arn:aws:cloudformation:us-east-1:xxxxxxxxxxxx:stack/ec2-demo-stack/9a686cf0-6c5c-11ee-97af-0ecf5ea19ac5

    This will create Changset called DemoChangSet1. 

    ./cfnctl list arn:aws:cloudformation:us-east-1:xxxxxxxxxxxx:stack/ec2-demo-stack/9a686cf0-6c5c-11ee-97af-0ecf5ea19ac5
    Check is Changeset is created

    ./cfnctl view arn:aws:cloudformation:us-east-1:xxxxxxxxxxxx:stack/ec2-demo-stack/9a686cf0-6c5c-11ee-97af-0ecf5ea19ac5
    You can check the Changset and find if any changes are required to recreate resource

    In Changeset, you can see that all resource change has attribute like:

    "Replacement": "False",

    And this means that your resource will not need to be recreated in order to make change.

    You can either apply changeset if you like it by:
    ./cfnctl apply arn:aws:cloudformation:us-east-1:xxxxxxxxxxxx:stack/ec2-demo-stack/9a686cf0-6c5c-11ee-97af-0ecf5ea19ac5
    or you can delete it

    Example 3

    If you change SecurityGroup description which is immutatble, and execute
    ./cfnctl upload 
    ./cfnctl update arn:aws:cloudformation:us-east-1:xxxxxxxxxxxx:stack/ec2-demo-stack/9a686cf0-6c5c-11ee-97af-0ecf5ea19ac5

    ./cfnctl-view arn:aws:cloudformation:us-east-1:xxxxxxxxxxxx:stack/ec2-demo-stack/9a686cf0-6c5c-11ee-97af-0ecf5ea19ac5

    You will see something like following in the changeset:
    "type": "Resource",
        "resourceChange": {
        "action": "Modify",
        "logicalResourceId": "WebServerSecurityGroup",
        "physicalResourceId": "ec2-demo-stack-WebServerSecurityGroup-171P46959ONMV",
        "resourceType": "AWS::EC2::SecurityGroup",
        "replacement": "True",

        :
        :
        }

    The replacement is true, meaning your resource need to be recreated because of this change. Remember, when SG is recreated, your EC2 may also need to be recreated because of SG change.
    You can confirm by checking changeset on EC2 instance.

    You then either apply and delete this changeset.


4. Using CloudFormation Module 

    Example 4

    ModuleS3 folder contains a Cloudformation module that can reused in other Cloudformation template. This S3 module leverage best practices to create a private S3 bucket.

    To deploy this module, go to folder ModuleS3 execute
    cfn validate
    cfn submit

    To use this S3 Module, add follow to the resource section in demo-ec2.json

    "S3Bucket" : {
        "Type" : "AWS::S3::PrivateAccess::MODULE",
        "Properties" : {
            "KMSKeyAlias" : {
            "Fn::Sub" : "${AWS::StackName}" 
            },
            "ReadWriteArn" : {
            "Fn::Sub" : "arn:aws:iam::${AWS::AccountId}:role/Admin" 
            },
            "ReadOnlyArn" : {
            "Fn::Sub" : "arn:aws:iam::${AWS::AccountId}:role/AWSEC2ServiceRole-xxxx-demo"
            }
        }
        }

For more information, please check https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/modules.html
