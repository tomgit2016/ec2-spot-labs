# Testing with EC2 Spot Instances and Spot Fleet

## Overview

This Builder Session will demonstrate how to leverage the Spot Fleet API to perform cost-effective automated testing with EC2 Spot Instances. We'll learn how to diversify the Spot Fleet across instance pools to minimize the impact of interruptions, how to dynamically scale up the Fleet size using AutoScaling Policies, and some strategies for handling Spot Instance interruptions. This is a fairly simple example - the goal is to get you familiar enough with Spot Fleets and Spot Instance interruption handling so that you can start experimenting with your own workloads. For more realistic load testing, consider integrating a framework like [Apache JMeter](https://jmeter.apache.org/) with your EC2 Instances.

## Requirements, notes, and legal
1. Please read the [Amazon EC2 Testing Policy](https://aws.amazon.com/ec2/testing/) before using any of the testing strategies presented in this session on your own services or workloads.

1. To complete this session, have access to an AWS account with administrative permissions. An IAM user with administrator access (**arn:aws:iam::aws:policy/AdministratorAccess**) will do nicely.

1. Make sure you have the latest version of the AWS CLI installed. https://docs.aws.amazon.com/cli/latest/userguide/installing.html

1. This session is configured to run in us-east-1. To run in a different region, make sure you pass equivalent AMIs as parameters when creating the CloudFormation stack.

1. While the session provides step by step instructions, please do take a moment to look around and understand what is happening at each step. The session is meant as a getting started guide, but you will learn the most by digesting each of the steps and thinking about how they would apply in your own environment. You might even consider experimenting with the steps to challenge yourself.

1. During this session, you will install software (and dependencies) on the Amazon EC2 instances launched in your account. The software packages and/or sources you will install will be from the [Amazon Linux 2](https://aws.amazon.com/amazon-linux-2/) distribution as well as from third party repositories and sites. Please review and decide your comfort with installing these before continuing.

1. Remember to complete the **Clean up all resources** section after you are done with this builder session, even if you haven't completed all the steps.


## Architecture

You will first upload the following to an S3 bucket:
* A [script](simple_server.py) for running the service to be tested. This service is a simple, single-threaded HTTP server that throttles incoming requests at 5 TPS.
* A [script](test_simple_server.py) for testing this service. This script spins up one thread per vCPU on the instance where it's running. Each thread continuously calls the target service at 1 TPS and publishes success and error metrics to CloudWatch.

Next, you will create the [CloudFormation stack](demo_setup.yaml) which performs most of the overhead setup for this builder session. This stack creates the following:
* a publicly accessible web server running on an On-Demand Instance
* prerequisites for launching a Spot Fleet to test this web server. These prerequisites are:
  * a VPC
  * two subnets
  * a Launch Template, which includes the EC2 UserData that starts the test script
  * the Spot Fleet IAM Role
* a CloudWatch event rule that will publish to an SNS topic every time a Spot Instance is interrupted

Finally, you will deploy a [Fleet of Spot Instances](request_spot_fleet_template.json) using the Launch Template and subnets generated by the previous CloudFormation stack. This Fleet will test the target web server and publish results to CloudWatch.

Here is a diagram of the resulting architecture:

![](images/testing_with_spot_fleet_arch.png)

## Steps

### 1. Checkout this repo

```
git clone https://github.com/awslabs/ec2-spot-labs
```

```
cd ec2-spot-labs/builder-sessions/test-dev-on-spot
```

### 2. Push the web server script and test script to S3

Create an S3 bucket.

```
username=<your_github_username>
```

```
bucket=$username-reinvent-ec2spottestdev-bucket
```

```
aws s3api create-bucket --bucket $bucket
```

Upload scripts to the bucket.

```
aws s3 cp simple_server.py s3://$bucket/
```

```
aws s3 cp test_simple_server.py s3://$bucket/
```

### 3. Create an ssh key pair

Create an SSH key pair. This can be used to SSH to any of the EC2 instances launched in this builder session.

```
aws ec2 create-key-pair \
  --key-name ReInvent-EC2SpotTestDev-KeyPair \
  --query 'KeyMaterial' \
  --output text \
  > ReInvent-EC2SpotTestDev-KeyPair.pem
```

```
chmod 400 ReInvent-EC2SpotTestDev-KeyPair.pem
```

### 4. Create the CloudFormation stack

Create the CloudFormation stack which performs most of the overhead setup for this builder session. This stack creates the following:
* an HTTP web server running on an On-Demand EC2 instance and publicly accessible at the "webServerPublicDns" field output from the stack
* prerequisites for launching a Fleet of Spot Instances to test this web server. These prerequisites are:
  * a VPC
  * two subnets
  * a Launch Template, which includes the EC2 UserData that starts the test script
  * the Spot Fleet IAM Role
* a CloudWatch event rule that will publish to an SNS topic every time a Spot Instance is interrupted

```
aws cloudformation create-stack \
  --template-body file://demo_setup.yaml \
  --parameters ParameterKey=s3Bucket,ParameterValue=$bucket \
  --capabilities CAPABILITY_IAM \
  --stack-name ReInvent-EC2SpotTestDev-Stack
```

Wait for the stack creation to complete (StackStatus == CREATE_COMPLETE) - this will take a few minutes. While you wait, take a look at the [CloudFormation template](demo_setup.yaml) and [web server script](simple_server.py) to get an understanding of what's happening.

```
aws cloudformation describe-stacks \
  --stack-name ReInvent-EC2SpotTestDev-Stack \
  --query 'Stacks[0].StackStatus' \
  --output text
```

After the stack creation is complete, validate that the public web server we will test is running by navigating to it's public DNS in your brower. You should see a "Hello World!" message. Note that it may take another minute or two after the stack creation is complete before the web server boots up and can start serving traffic.

Command to parse the public DNS of the web server:

```
aws cloudformation describe-stacks \
  --stack-name ReInvent-EC2SpotTestDev-Stack \
  --query 'Stacks[0].Outputs[?OutputKey==`webServerPublicDns`].OutputValue' \
  --output text
```

An SNS topic named ReInvent-EC2SpotTestDev-SpotInterruptionTopic was created in this stack. This topic will publish for every Spot Instance that is interrupted.

Feel free to subcribe to this topic if you'd like to test this functionality (e.g. email or SMS). This can be done in the AWS Console at Services > SimpleNotificationService > Topics > ReInvent-EC2SpotTestDev-SpotInterruptionTopic > Actions > Subscribe to topic. You'll get an opt-in confirmation message that you'll also need to acknowledge to enable the notifications.


### 5. Launch a Fleet of Spot Instances to test our service

In step 4, we created two subnets and a Launch Template that we'll use for launching the testing Fleet. Retrieve these values.

```
launch_template_id=\
$(aws cloudformation describe-stacks \
  --stack-name ReInvent-EC2SpotTestDev-Stack \
  --query 'Stacks[0].Outputs[?OutputKey==`testFleetLaunchTemplate`].OutputValue' \
  --output text)
```

```
subnet_id_1=\
$(aws cloudformation describe-stacks \
  --stack-name ReInvent-EC2SpotTestDev-Stack \
  --query 'Stacks[0].Outputs[?OutputKey==`testFleetSubnet1`].OutputValue' \
  --output text)
```

```
subnet_id_2=\
$(aws cloudformation describe-stacks \
  --stack-name ReInvent-EC2SpotTestDev-Stack \
  --query 'Stacks[0].Outputs[?OutputKey==`testFleetSubnet2`].OutputValue' \
  --output text)
```

```
spot_fleet_role_arn=\
$(aws cloudformation describe-stacks \
  --stack-name ReInvent-EC2SpotTestDev-Stack \
  --query 'Stacks[0].Outputs[?OutputKey==`spotFleetRoleArn`].OutputValue' \
  --output text)
```

Now, build the input JSON for the RequestSpotFleet call.

```
sed s~%LAUNCH_TEMPLATE_ID%~$launch_template_id~ request_spot_fleet_template.json \
  | sed s~%SUBNET_ID_1%~$subnet_id_1~ \
  | sed s~%SUBNET_ID_2%~$subnet_id_2~ \
  | sed s~%IAM_ROLE_ARN%~$spot_fleet_role_arn~ \
  > request_spot_fleet.json
```

Take note of some important values in the request_spot_fleet.json file that we just created:
* Type: maintain
  * Fleet maintains your target capacity by replenishing interrupted Spot Instances.
  * Other options are:
    * instant - Fleet submits a one-time request for your desired capacity.
    * request - Fleet submits ongoing requests until your desired capacity is fulfilled, but does not attempt to submit requests in alternative capacity pools if capacity is unavailable.
* WeightedCapacity
  * We've weighted each instance type by the number of vCPUs.
* TargetCapacity: 4, OnDemandTargetCapacity: 0
  * This is the target capacity for the Fleet, taking into account weighting. So, in our example, we're requesting a Fleet of Spot Instances that combine for 4 total vCPUs.
  * In this case, we've requested all Spot Instances, but it is possible to mix On-Demand and Spot Instances in the same Fleet.
* AllocationStrategy: lowestPrice, InstancePoolsToUseCount: 2
  * This says to diversify the Fleet across the 2 cheapest Spot pools where capacity is available.
  * In this context, a "pool" is the combination of instance type (e.g. t2.small) and Availability Zone.

Create the Fleet and record the resulting request id.

```
sfr_id=\
$(aws ec2 request-spot-fleet \
  --spot-fleet-request-config file://request_spot_fleet.json \
  --query 'SpotFleetRequestId' \
  --output text)
```

```
echo $sfr_id
```

Wait for the Fleet to launch (RequestState == 'active', FulfilledCapacity == 4).

```
aws ec2 describe-spot-fleet-requests \
  --spot-fleet-request-ids $sfr_id \
  --query 'SpotFleetRequestConfigs[0].{RequestState:SpotFleetRequestState,FulfilledCapacity:SpotFleetRequestConfig.FulfilledCapacity}'
```

You should also be able to describe the EC2 instances that have been launched in this Fleet. They should be distributed across two Spot pools (Instance Type + Availability Zone).

```
aws ec2 describe-instances \
  --filters Name=tag:aws:ec2spot:fleet-request-id,Values=$sfr_id \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,Placement.AvailabilityZone,CpuOptions.CoreCount]'
```

If your Fleet isn't getting fulfilled, you can debug by retrieving the Fleet history.

```
aws ec2 describe-spot-fleet-request-history \
  --spot-fleet-request-id $sfr_id \
  --start-time 2018-11-01
```

We've now launched a Fleet of Spot Instances totaling 4 vCPUs and diversified across 2 Spot pools. If Spot capacity decreases in one of the pools where we've launched, one or more of our instances may have to be interrupted. In this case, the Fleet will automatically try to replenish the missing capacity using another pool. This is why it's important to diversify the number of pools in your request as much as possible.


### 6. Verify the CloudWatch metrics

The test Fleet should now be running and publishing test results as CloudWatch metrics. Verify this in the AWS Console at Services -> CloudWatch -> Metrics -> ReInvent-EC2SpotTestDev-Metrics. View the Sum of the "Error" and "Success" metrics with a 1 Minute Period. At the current Fleet size (4 vCPUs), tests should be succeeding because our test Fleet calls the target web server at 1 TPS per vCPU (so, 4 TPS total) and the web server can handle 5 TPS.


### 7. Scale up the Fleet size to test the limits of the target service using an AutoScaling Policy

Register the Fleet as a scalable target.

```
aws application-autoscaling register-scalable-target \
  --service-namespace ec2 \
  --resource-id spot-fleet-request/$sfr_id \
  --scalable-dimension ec2:spot-fleet-request:TargetCapacity \
  --min-capacity 1 \
  --max-capacity 10
```

Create an Autoscaling Policy for the Fleet.

```
aws application-autoscaling put-scaling-policy \
  --policy-name ReInvent-EC2SpotTestDev-ScalingPolicy \
  --service-namespace ec2 \
  --resource-id spot-fleet-request/$sfr_id \
  --scalable-dimension ec2:spot-fleet-request:TargetCapacity \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration file://scaling_policy_config.json
```

We've created an AutoScaling Policy here such that our Fleet will scale up until the testing script success rate is [reduced to 85%](scaling_policy_config.json) or the Fleet reaches a maximum size of 10 vCPUs. Remember, we weighted each instance type by number of vCPUs, so Fleet capacity is equal to vCPUs in this case.


### 8. Verify the Fleet scales as expected

In the AWS Console, navigate to Services > CloudWatch -> Alarms. An alarm should have been created as part of the AutoScaling Policy called TargetTracking-spot-fleet-request/\<fleet_request-id\>-AlarmHigh-\<UUID\>. This may be marked as INSUFFICIENT_DATA, but should soon move to ALARM. The Fleet should then continue to scale up until our testing Fleet is seeing a sufficient number of errors when calling the target web server.

You can verify this by keeping an eye on the alarm - the metric should drop until the alarm changes to OK. Once the alarm moves to OK, check the new Fleet size at which our target service failing under the load.

```
aws ec2 describe-spot-fleet-requests \
  --spot-fleet-request-ids $sfr_id \
  --query 'SpotFleetRequestConfigs[0].{RequestState:SpotFleetRequestState,FulfilledCapacity:SpotFleetRequestConfig.FulfilledCapacity}'
```


### 9. Delete the AutoScaling Policy

Delete the AutoScaling Policy and deregister the Fleet as a scaling target.

```
aws application-autoscaling delete-scaling-policy \
  --policy-name ReInvent-EC2SpotTestDev-ScalingPolicy \
  --service-namespace ec2 \
  --resource-id spot-fleet-request/$sfr_id \
  --scalable-dimension ec2:spot-fleet-request:TargetCapacity
```

```
aws application-autoscaling deregister-scalable-target \
  --service-namespace ec2 \
  --resource-id spot-fleet-request/$sfr_id \
  --scalable-dimension ec2:spot-fleet-request:TargetCapacity
```

This is a good stopping point in the Builder Session as we've demonstrated how to leverage the Spot Fleet API to create a Fleet of Spot Instances for testing a service and how to dynamically scale up the size of this Fleet. If you'd like to stop now, remember to execute the cleanup steps at end of this doc.

If you'd like to continue and learn some strategies for Spot Instance interruption handling, leave your Fleet running and continue on to the next section.


## Interruption handling strategies

The Spot Instances that we launched in the previous steps are stateless. Each instance is simply calling the target web server and posting results back to CloudWatch. If an instance is interrupted, the Spot Fleet will automatically replace it with a new one.

However, in many use cases, some additional logic is necessary to handle Spot Instance interruptions. Some examples:
* Saving the application state.
* Uploading final log files.
* Removing instance from Elastic Load Balancer.
* Pushing SNS notifications.
* Draining containers.

Spot Instances get a 2 minute notice before they'll be shut down, via both CloudWatch events and the instance metadata.

Spot instances also support 3 different interruption behaviors:
* terminate
* stop - EBS volume is saved and restored when the instance starts again
* hibernate - EBS volume and RAM are saved and restored when the instance starts again

See the documentation for detailed information on Spot Instance interruptions: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html

You can test interruption handling by decreasing the Fleet size because the default ExcessCapacityTerminationPolicy of a Fleet results in excess capacity being interrupted.


### 10. Interruption handling via CloudWatch events

In step 4 (creating the CloudFormation stack), we created a CloudWatch event rule that will publish to the SNS topic named ReInvent-EC2SpotTestDev-SpotInterruptionTopic for every Spot Instance that is interrupted. If you didn't already as part of step 4, you can subscribe to this topic if you'd like to test the functionality (e.g. email or SMS). This can be done in the AWS Console at Services > SimpleNotificationService > Topics > ReInvent-EC2SpotTestDev-SpotInterruptionTopic > Actions > Subscribe to topic. You'll get an opt-in confirmation message that you'll also need to acknowledge to enable the notifications.

Now reduce the fleet size, which should trigger Spot interruptions.

```
aws ec2 modify-spot-fleet-request \
  --spot-fleet-request-id $sfr_id \
  --target-capacity 2
```

Wait for the Spot instances to terminate (RequestState == 'active', FulfilledCapacity == 2).

```
aws ec2 describe-spot-fleet-requests \
  --spot-fleet-request-ids $sfr_id \
  --query 'SpotFleetRequestConfigs[0].{RequestState:SpotFleetRequestState,FulfilledCapacity:SpotFleetRequestConfig.FulfilledCapacity}'
```

You should have received a notification on the SNS Topic for every Spot instance that was interrupted.

Note that in this example we've created an SNS topic, but CloudWatch event rules are flexible and can even be configured to trigger Lambda functions.


### 11. Interruption notice on the instance metadata

Try to modify the [test script](test_simple_server.py) from the previous example to respond to an interruption notice. Maybe persist some state to S3 before shutting down.

Interruption notices are available on the instance metadata at the following url. It won't be reachable (`curl` returns a 404) until an interruption notice is delivered. 

```
http://169.254.169.254/latest/meta-data/spot/instance-action
```

To deploy the new version of the test script, first tear down the running Fleet.

```
aws ec2 cancel-spot-fleet-requests \
  --spot-fleet-request-ids $sfr_id \
  --terminate-instances
```

Upload the modified test script to S3.

```
aws s3 cp test_simple_server.py s3://$bucket/
```

Launch a new Fleet and wait for it to be fulfilled (as in Step 5).

```
sfr_id=\
$(aws ec2 request-spot-fleet \
  --spot-fleet-request-config file://request_spot_fleet.json \
  --query 'SpotFleetRequestId' \
  --output text)
```

Now reduce the Fleet size, which should trigger Spot interruptions.

```
aws ec2 modify-spot-fleet-request \
  --spot-fleet-request-id $sfr_id \
  --target-capacity 2
```

Validate that your new test script performed its expected interruption handling.


## Clean up all resources

Delete your subscription to the ReInvent-EC2SpotTestDev-SpotInterruptionTopic if you created one. This can be done via the AWS Console at Services > Simple Notification Service > Subscriptions > Actions > Delete Subscriptions

Delete the AutoScaling Policy and degister the Fleet if you didn't already do this as part of Step 9.

```
aws application-autoscaling delete-scaling-policy \
  --policy-name ReInvent-EC2SpotTestDev-ScalingPolicy \
  --service-namespace ec2 \
  --resource-id spot-fleet-request/$sfr_id \
  --scalable-dimension ec2:spot-fleet-request:TargetCapacity
```

```
aws application-autoscaling deregister-scalable-target \
  --service-namespace ec2 \
  --resource-id spot-fleet-request/$sfr_id \
  --scalable-dimension ec2:spot-fleet-request:TargetCapacity
```

Cancel the Spot Fleet.

```
aws ec2 cancel-spot-fleet-requests \
  --spot-fleet-request-ids $sfr_id \
  --terminate-instances
```

Delete the CloudFormation Stack.

```
aws cloudformation delete-stack \
  --stack-name ReInvent-EC2SpotTestDev-Stack
```

Empty and delete the S3 bucket.

```
aws s3 rm s3://$bucket --recursive
```

```
aws s3api delete-bucket \
  --bucket $bucket
```

Delete the SSH key pair.

```
aws ec2 delete-key-pair \
  --key-name ReInvent-EC2SpotTestDev-KeyPair
```


## Summary

In this session, we've demonstrated how to create a Fleet of Spot Instances to test a service. This pattern can be useful in a variety of scenarios:
* Unit and integration testing - leverage the Spot Fleet API to find the cheapest Spot pool that fits your needs when you need to test a new change.
* Performance testing - incrementally scale up a Fleet to test the limits of your service.
* Continuous black box testing - maintain a Fleet that continuously verifies the functionality of your service.

We also introduced some strategies for handling interruptions.

If you're interested in integrating Spot Instances into a CI/CD workflow, you should check out our Jenkins and Atlassian Bamboo plugins:
* Jenkins:
  * [GitHub](https://github.com/awslabs/ec2-spot-jenkins-plugin)
  * [Jenkins wiki](https://wiki.jenkins.io/display/JENKINS/Amazon+EC2+Fleet+Plugin)
  * [YouTube tutorial](https://www.youtube.com/watch?v=8gGItacZjps)
* Atlassian Bamboo
  * [GitHub](https://github.com/awslabs/ec2-spot-fleet-bamboo-plugin)
  * [Atlassian Marketplace](https://marketplace.atlassian.com/apps/1218167/awsspotfleetbambooplugin?hosting=server&tab=overview)
