--------

The AWS SDK for Java team is hiring [software development engineers](https://github.com/aws/aws-sdk-java-v2/issues/3156) that are excited about open source software and the AWS developer experience\!

--------

# Tutorial: Amazon EC2 Spot Instances<a name="tutorial-spot-instances-java"></a>

## Overview<a name="tutor-spot-java-overview"></a>

Spot Instances enable you to bid on unused Amazon Elastic Compute Cloud \(Amazon EC2\) capacity t up to 90% versus the On\-Demand Instance price and run the acquired instances for as long as your bid exceeds the current *Spot Price*\. Amazon EC2 changes the Spot Price periodically based on supply and demand, and customers whose bids meet or exceed it gain access to the available Spot Instances\. Like On\-Demand Instances and Reserved Instances, Spot Instances provide you another option for obtaining more compute capacity\.

Spot Instances can significantly lower your Amazon EC2 costs for batch processing, scientific research, image processing, video encoding, data and web crawling, financial analysis, and testing\. Additionally, Spot Instances give you access to large amounts of additional capacity in situations where the need for that capacity is not urgent\.

To use Spot Instances, place a Spot Instance request specifying the maximum price you are willing to pay per instance hour; this is your bid\. If your bid exceeds the current Spot Price, your request is fulfilled and your instances will run until either you choose to terminate them or the Spot Price increases above your bid \(whichever is sooner\)\.

It’s important to note:
+ You will often pay less per hour than your bid\. Amazon EC2 adjusts the Spot Price periodically as requests come in and available supply changes\. Everyone pays the same Spot Price for that period regardless of whether their bid was higher\. Therefore, you might pay less than your bid, but you will never pay more than your bid\.
+ If you’re running Spot Instances and your bid no longer meets or exceeds the current Spot Price, your instances will be terminated\. This means that you will want to make sure that your workloads and applications are flexible enough to take advantage of this opportunistic capacity\.

Spot Instances perform exactly like other Amazon EC2 instances while running, and like other Amazon EC2 instances, Spot Instances can be terminated when you no longer need them\. If you terminate your instance, you pay for any partial hour used \(as you would for On\-Demand or Reserved Instances\)\. However, if the Spot Price goes above your bid and your instance is terminated by Amazon EC2, you will not be charged for any partial hour of usage\.

This tutorial shows how to use AWS SDK for Java to do the following\.
+ Submit a Spot Request
+ Determine when the Spot Request becomes fulfilled
+ Cancel the Spot Request
+ Terminate associated instances

## Prerequisites<a name="tutor-spot-java-prereq"></a>

To use this tutorial you must have the AWS SDK for Java installed, as well as having met its basic installation prerequisites\. See [Set up the AWS SDK for Java](setup-install.md) for more information\.

## Step 1: Setting Up Your Credentials<a name="tutor-spot-java-credentials"></a>

To begin using this code sample, you need to set up AWS credentials\. See [Set up AWS Credentials and Region for Development](setup-credentials.md) for instructions on how to do that\.

**Note**  
We recommend that you use the credentials of an IAM user to provide these values\. For more information, see [Sign Up for AWS and Create an IAM User](signup-create-iam-user.md)\.

Now that you have configured your settings, you can get started using the code in the example\.

## Step 2: Setting Up a Security Group<a name="tutor-spot-java-sg"></a>

A *security group* acts as a firewall that controls the traffic allowed in and out of a group of instances\. By default, an instance is started without any security group, which means that all incoming IP traffic, on any TCP port will be denied\. So, before submitting our Spot Request, we will set up a security group that allows the necessary network traffic\. For the purposes of this tutorial, we will create a new security group called "GettingStarted" that allows Secure Shell \(SSH\) traffic from the IP address where you are running your application from\. To set up a new security group, you need to include or run the following code sample that sets up the security group programmatically\.

After we create an `AmazonEC2` client object, we create a `CreateSecurityGroupRequest` object with the name, "GettingStarted" and a description for the security group\. Then we call the `ec2.createSecurityGroup` API to create the group\.

To enable access to the group, we create an `ipPermission` object with the IP address range set to the CIDR representation of the subnet for the local computer; the "/10" suffix on the IP address indicates the subnet for the specified IP address\. We also configure the `ipPermission` object with the TCP protocol and port 22 \(SSH\)\. The final step is to call `ec2.authorizeSecurityGroupIngress` with the name of our security group and the `ipPermission` object\.

```
// Create the AmazonEC2 client so we can call various APIs.
AmazonEC2 ec2 = AmazonEC2ClientBuilder.defaultClient();

// Create a new security group.
try {
    CreateSecurityGroupRequest securityGroupRequest = new CreateSecurityGroupRequest("GettingStartedGroup", "Getting Started Security Group");
    ec2.createSecurityGroup(securityGroupRequest);
} catch (AmazonServiceException ase) {
    // Likely this means that the group is already created, so ignore.
    System.out.println(ase.getMessage());
}

String ipAddr = "0.0.0.0/0";

// Get the IP of the current host, so that we can limit the Security
// Group by default to the ip range associated with your subnet.
try {
    InetAddress addr = InetAddress.getLocalHost();

    // Get IP Address
    ipAddr = addr.getHostAddress()+"/10";
} catch (UnknownHostException e) {
}

// Create a range that you would like to populate.
ArrayList<String> ipRanges = new ArrayList<String>();
ipRanges.add(ipAddr);

// Open up port 22 for TCP traffic to the associated IP
// from above (e.g. ssh traffic).
ArrayList<IpPermission> ipPermissions = new ArrayList<IpPermission> ();
IpPermission ipPermission = new IpPermission();
ipPermission.setIpProtocol("tcp");
ipPermission.setFromPort(new Integer(22));
ipPermission.setToPort(new Integer(22));
ipPermission.setIpRanges(ipRanges);
ipPermissions.add(ipPermission);

try {
    // Authorize the ports to the used.
    AuthorizeSecurityGroupIngressRequest ingressRequest =
        new AuthorizeSecurityGroupIngressRequest("GettingStartedGroup",ipPermissions);
    ec2.authorizeSecurityGroupIngress(ingressRequest);
} catch (AmazonServiceException ase) {
    // Ignore because this likely means the zone has
    // already been authorized.
    System.out.println(ase.getMessage());
}
```

Note you only need to run this application once to create a new security group\.

You can also create the security group using the AWS Toolkit for Eclipse\. See [Managing Security Groups from AWS Cost Explorer](https://docs.aws.amazon.com/toolkit-for-eclipse/v1/user-guide/tke-sg.html) for more information\.

## Step 3: Submitting Your Spot Request<a name="tutor-spot-java-submit"></a>

To submit a Spot request, you first need to determine the instance type, Amazon Machine Image \(AMI\), and maximum bid price you want to use\. You must also include the security group we configured previously, so that you can log into the instance if desired\.

There are several instance types to choose from; go to Amazon EC2 Instance Types for a complete list\. For this tutorial, we will use t1\.micro, the cheapest instance type available\. Next, we will determine the type of AMI we would like to use\. We’ll use ami\-a9d09ed1, the most up\-to\-date Amazon Linux AMI available when we wrote this tutorial\. The latest AMI may change over time, but you can always determine the latest version AMI by following these steps:

1. Open the [Amazon EC2 console](https://console.aws.amazon.com/ec2/home)\.

1. Choose the **Launch Instance** button\.

1. The first window displays the AMIs available\. The AMI ID is listed next to each AMI title\. Alternatively, you can use the `DescribeImages` API, but leveraging that command is outside the scope of this tutorial\.

There are many ways to approach bidding for Spot Instances; to get a broad overview of the various approaches you should view the [Bidding for Spot Instances](https://www.youtube.com/watch?v=WD9N73F3Fao&feature=player_embedded) video\. However, to get started, we’ll describe three common strategies: bid to ensure cost is less than on\-demand pricing; bid based on the value of the resulting computation; bid so as to acquire computing capacity as quickly as possible\.
+  *Reduce Cost below On\-Demand* You have a batch processing job that will take a number of hours or days to run\. However, you are flexible with respect to when it starts and when it completes\. You want to see if you can complete it for less cost than with On\-Demand Instances\. You examine the Spot Price history for instance types using either the AWS Management Console or the Amazon EC2 API\. For more information, go to [Viewing Spot Price History](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances-history.html)\. After you’ve analyzed the price history for your desired instance type in a given Availability Zone, you have two alternative approaches for your bid:
  + You could bid at the upper end of the range of Spot Prices \(which are still below the On\-Demand price\), anticipating that your one\-time Spot request would most likely be fulfilled and run for enough consecutive compute time to complete the job\.
  + Or, you could specify the amount you are willing to pay for Spot Instances as a % of the On\-Demand Instance price , and plan to combine many instances launched over time through a persistent request\. If the specified price is exceeded, then the Spot Instance will terminate\. \(We will explain how to automate this task later in this tutorial\.\)
+  *Pay No More than the Value of the Result* You have a data processing job to run\. You understand the value of the job’s results well enough to know how much they are worth in terms of computing costs\. After you’ve analyzed the Spot Price history for your instance type, you choose a bid price at which the cost of the computing time is no more than the value of the job’s results\. You create a persistent bid and allow it to run intermittently as the Spot Price fluctuates at or below your bid\.
+  *Acquire Computing Capacity Quickly* You have an unanticipated, short\-term need for additional capacity that is not available through On\-Demand Instances\. After you’ve analyzed the Spot Price history for your instance type, you bid above the highest historical price to provide a high likelihood that your request will be fulfilled quickly and continue computing until it completes\.

After you choose your bid price, you are ready to request a Spot Instance\. For the purposes of this tutorial, we will bid the On\-Demand price \($0\.03\) to maximize the chances that the bid will be fulfilled\. You can determine the types of available instances and the On\-Demand prices for instances by going to Amazon EC2 Pricing page\. While a Spot Instancee is running, you pay the Spot price that’s in effect for the time period your instances are running\. Spot Instance prices are set by Amazon EC2 and adjust gradually based on long\-term trends in supply and demand for Spot Instance capacity\. You can also specify the amount you are willing to pay for a Spot Instance as a % of the On\-Demand Instance price\.To request a Spot Instance, you simply need to build your request with the parameters you chose earlier\. We start by creating a `RequestSpotInstanceRequest` object\. The request object requires the number of instances you want to start and the bid price\. Additionally, you need to set the `LaunchSpecification` for the request, which includes the instance type, AMI ID, and security group you want to use\. Once the request is populated, you call the `requestSpotInstances` method on the `AmazonEC2Client` object\. The following example shows how to request a Spot Instance\.

```
// Create the AmazonEC2 client so we can call various APIs.
AmazonEC2 ec2 = AmazonEC2ClientBuilder.defaultClient();

// Initializes a Spot Instance Request
RequestSpotInstancesRequest requestRequest = new RequestSpotInstancesRequest();

// Request 1 x t1.micro instance with a bid price of $0.03.
requestRequest.setSpotPrice("0.03");
requestRequest.setInstanceCount(Integer.valueOf(1));

// Setup the specifications of the launch. This includes the
// instance type (e.g. t1.micro) and the latest Amazon Linux
// AMI id available. Note, you should always use the latest
// Amazon Linux AMI id or another of your choosing.
LaunchSpecification launchSpecification = new LaunchSpecification();
launchSpecification.setImageId("ami-a9d09ed1");
launchSpecification.setInstanceType(InstanceType.T1Micro);

// Add the security group to the request.
ArrayList<String> securityGroups = new ArrayList<String>();
securityGroups.add("GettingStartedGroup");
launchSpecification.setSecurityGroups(securityGroups);

// Add the launch specifications to the request.
requestRequest.setLaunchSpecification(launchSpecification);

// Call the RequestSpotInstance API.
RequestSpotInstancesResult requestResult = ec2.requestSpotInstances(requestRequest);
```

Running this code will launch a new Spot Instance Request\. There are other options you can use to configure your Spot Requests\. To learn more, please visit [Tutorial: Advanced Amazon EC2 Spot Request Management](tutorial-spot-adv-java.md) or the [RequestSpotInstances](https://docs.aws.amazon.com/sdk-for-java/v1/reference/com/amazonaws/services/ec2/model/RequestSpotInstancesRequest.html) class in the AWS SDK for Java API Reference\.

**Note**  
You will be charged for any Spot Instances that are actually launched, so make sure that you cancel any requests and terminate any instances you launch to reduce any associated fees\.

## Step 4: Determining the State of Your Spot Request<a name="tutor-spot-java-request-state"></a>

Next, we want to create code to wait until the Spot request reaches the "active" state before proceeding to the last step\. To determine the state of our Spot request, we poll the [describeSpotInstanceRequests](http://docs.aws.amazon.com/sdk-for-java/v1/reference/com/amazonaws/services/ec2/AmazonEC2Client.html#describeSpotInstanceRequests) method for the state of the Spot request ID we want to monitor\.

The request ID created in Step 2 is embedded in the response to our `requestSpotInstances` request\. The following example code shows how to gather request IDs from the `requestSpotInstances` response and use them to populate an `ArrayList`\.

```
// Call the RequestSpotInstance API.
RequestSpotInstancesResult requestResult = ec2.requestSpotInstances(requestRequest);
List<SpotInstanceRequest> requestResponses = requestResult.getSpotInstanceRequests();

// Setup an arraylist to collect all of the request ids we want to
// watch hit the running state.
ArrayList<String> spotInstanceRequestIds = new ArrayList<String>();

// Add all of the request ids to the hashset, so we can determine when they hit the
// active state.
for (SpotInstanceRequest requestResponse : requestResponses) {
    System.out.println("Created Spot Request: "+requestResponse.getSpotInstanceRequestId());
    spotInstanceRequestIds.add(requestResponse.getSpotInstanceRequestId());
}
```

To monitor your request ID, call the `describeSpotInstanceRequests` method to determine the state of the request\. Then loop until the request is not in the "open" state\. Note that we monitor for a state of not "open", rather a state of, say, "active", because the request can go straight to "closed" if there is a problem with your request arguments\. The following code example provides the details of how to accomplish this task\.

```
// Create a variable that will track whether there are any
// requests still in the open state.
boolean anyOpen;

do {
    // Create the describeRequest object with all of the request ids
    // to monitor (e.g. that we started).
    DescribeSpotInstanceRequestsRequest describeRequest = new DescribeSpotInstanceRequestsRequest();
    describeRequest.setSpotInstanceRequestIds(spotInstanceRequestIds);

    // Initialize the anyOpen variable to false - which assumes there
    // are no requests open unless we find one that is still open.
    anyOpen=false;

    try {
        // Retrieve all of the requests we want to monitor.
        DescribeSpotInstanceRequestsResult describeResult = ec2.describeSpotInstanceRequests(describeRequest);
        List<SpotInstanceRequest> describeResponses = describeResult.getSpotInstanceRequests();

        // Look through each request and determine if they are all in
        // the active state.
        for (SpotInstanceRequest describeResponse : describeResponses) {
            // If the state is open, it hasn't changed since we attempted
            // to request it. There is the potential for it to transition
            // almost immediately to closed or cancelled so we compare
            // against open instead of active.
        if (describeResponse.getState().equals("open")) {
            anyOpen = true;
            break;
        }
    }
} catch (AmazonServiceException e) {
      // If we have an exception, ensure we don't break out of
      // the loop. This prevents the scenario where there was
      // blip on the wire.
      anyOpen = true;
    }

    try {
        // Sleep for 60 seconds.
        Thread.sleep(60*1000);
    } catch (Exception e) {
        // Do nothing because it woke up early.
    }
} while (anyOpen);
```

After running this code, your Spot Instance Request will have completed or will have failed with an error that will be output to the screen\. In either case, we can proceed to the next step to clean up any active requests and terminate any running instances\.

## Step 5: Cleaning Up Your Spot Requests and Instances<a name="tutor-spot-java-cleaning-up"></a>

Lastly, we need to clean up our requests and instances\. It is important to both cancel any outstanding requests *and* terminate any instances\. Just canceling your requests will not terminate your instances, which means that you will continue to pay for them\. If you terminate your instances, your Spot requests may be canceled, but there are some scenarios such as if you use persistent bids where terminating your instances is not sufficient to stop your request from being re\-fulfilled\. Therefore, it is a best practice to both cancel any active bids and terminate any running instances\.

The following code demonstrates how to cancel your requests\.

```
try {
    // Cancel requests.
    CancelSpotInstanceRequestsRequest cancelRequest =
       new CancelSpotInstanceRequestsRequest(spotInstanceRequestIds);
    ec2.cancelSpotInstanceRequests(cancelRequest);
} catch (AmazonServiceException e) {
    // Write out any exceptions that may have occurred.
    System.out.println("Error cancelling instances");
    System.out.println("Caught Exception: " + e.getMessage());
    System.out.println("Reponse Status Code: " + e.getStatusCode());
    System.out.println("Error Code: " + e.getErrorCode());
    System.out.println("Request ID: " + e.getRequestId());
}
```

To terminate any outstanding instances, you will need the instance ID associated with the request that started them\. The following code example takes our original code for monitoring the instances and adds an `ArrayList` in which we store the instance ID associated with the `describeInstance` response\.

```
// Create a variable that will track whether there are any requests
// still in the open state.
boolean anyOpen;
// Initialize variables.
ArrayList<String> instanceIds = new ArrayList<String>();

do {
   // Create the describeRequest with all of the request ids to
   // monitor (e.g. that we started).
   DescribeSpotInstanceRequestsRequest describeRequest = new DescribeSpotInstanceRequestsRequest();
   describeRequest.setSpotInstanceRequestIds(spotInstanceRequestIds);

   // Initialize the anyOpen variable to false, which assumes there
   // are no requests open unless we find one that is still open.
   anyOpen = false;

   try {
         // Retrieve all of the requests we want to monitor.
         DescribeSpotInstanceRequestsResult describeResult =
            ec2.describeSpotInstanceRequests(describeRequest);

         List<SpotInstanceRequest> describeResponses =
            describeResult.getSpotInstanceRequests();

         // Look through each request and determine if they are all
         // in the active state.
         for (SpotInstanceRequest describeResponse : describeResponses) {
           // If the state is open, it hasn't changed since we
           // attempted to request it. There is the potential for
           // it to transition almost immediately to closed or
           // cancelled so we compare against open instead of active.
           if (describeResponse.getState().equals("open")) {
              anyOpen = true; break;
           }
           // Add the instance id to the list we will
           // eventually terminate.
           instanceIds.add(describeResponse.getInstanceId());
         }
   } catch (AmazonServiceException e) {
      // If we have an exception, ensure we don't break out
      // of the loop. This prevents the scenario where there
      // was blip on the wire.
      anyOpen = true;
   }

    try {
        // Sleep for 60 seconds.
        Thread.sleep(60*1000);
    } catch (Exception e) {
        // Do nothing because it woke up early.
    }
} while (anyOpen);
```

Using the instance IDs, stored in the `ArrayList`, terminate any running instances using the following code snippet\.

```
try {
    // Terminate instances.
    TerminateInstancesRequest terminateRequest = new TerminateInstancesRequest(instanceIds);
    ec2.terminateInstances(terminateRequest);
} catch (AmazonServiceException e) {
    // Write out any exceptions that may have occurred.
    System.out.println("Error terminating instances");
    System.out.println("Caught Exception: " + e.getMessage());
    System.out.println("Reponse Status Code: " + e.getStatusCode());
    System.out.println("Error Code: " + e.getErrorCode());
    System.out.println("Request ID: " + e.getRequestId());
}
```

## Bringing It All Together<a name="tutor-spot-java-bring-together"></a>

To bring this all together, we provide a more object\-oriented approach that combines the preceding steps we showed: initializing the EC2 Client, submitting the Spot Request, determining when the Spot Requests are no longer in the open state, and cleaning up any lingering Spot request and associated instances\. We create a class called `Requests` that performs these actions\.

We also create a `GettingStartedApp` class, which has a main method where we perform the high level function calls\. Specifically, we initialize the `Requests` object described previously\. We submit the Spot Instance request\. Then we wait for the Spot request to reach the "Active" state\. Finally, we clean up the requests and instances\.

The complete source code for this example can be viewed or downloaded at [GitHub](https://github.com/aws/aws-sdk-java/tree/master/src/samples/AmazonEC2SpotInstances-GettingStarted)\.

Congratulations\! You have just completed the getting started tutorial for developing Spot Instance software with the AWS SDK for Java\.

## Next Steps<a name="tutor-spot-java-next"></a>

Proceed with [Tutorial: Advanced Amazon EC2 Spot Request Management](tutorial-spot-adv-java.md)\.