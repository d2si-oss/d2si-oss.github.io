---
author: Pauline Blanc
layout: post
title: "Windows Performance Counters and Event logs monitoring using Systems Manager and CloudWatch"
twitter: pbl_tw
keywords: windows automation,aws,cloudwatch,event logs,perforance counters
---

## Introduction

[AWS Systems Manager](http://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html)
helps you automate management tasks. It can be used to collect system inventory
such as Performance Counters or event logs.  To achieve this, you have to
configure the SSM Agent on your Windows instance to send metrics to CloudWatch.

[SSM Agent](http://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html)
is used since November 2016 on 2003-2012R2 AMI and on 2016 AMI. In this article,
we will configure an EC2 IAM role to allow the SSM Agent on a Windows instance
to send event logs and Memory Usage performance counter into CloudWatch.

## Create an IAM role

You have to create an IAM role to allow your instances to send metrics and logs to CloudWatch.

### Create a policy

<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/01.png" alt="XXX" width="800" style="margin: 0px auto;display:block;" />

Create a policy in IAM using the JSON below:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeAssociation",
                "ssm:GetDeployablePatchSnapshotForInstance",
                "ssm:GetDocument",
                "ssm:GetParameters",
                "ssm:ListAssociations",
                "ssm:ListInstanceAssociations",
                "ssm:PutInventory",
                "ssm:UpdateAssociationStatus",
                "ssm:UpdateInstanceAssociationStatus",
                "ssm:UpdateInstanceInformation"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2messages:AcknowledgeMessage",
                "ec2messages:DeleteMessage",
                "ec2messages:FailMessage",
                "ec2messages:GetEndpoint",
                "ec2messages:GetMessages",
                "ec2messages:SendReply"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricData"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstanceStatus"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ds:CreateComputer",
                "ds:DescribeDirectories"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts",
                "s3:ListBucketMultipartUploads"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::amazon-ssm-packages-*"
        }
    ]
}
```

### Create the Role

Create an EC2 Role and attach the previously created policy.
  
<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/02.png" alt="XXX" width="601" style="margin: 0px auto;display:block;" />

<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/03.png" alt="XXX" width="601" style="margin: 0px auto;display:block;" />

<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/04.png" alt="XXX" width="800" style="margin: 0px auto;display:block;" />

You can now
[attach the IAM Role](https://aws.amazon.com/fr/blogs/security/new-attach-an-aws-iam-role-to-an-existing-amazon-ec2-instance-by-using-the-aws-cli/)
to your instances.

The SSM Agent will use the IAM Role to send metrics to CloudWatch.

## Prepare your configuration file

You have to configure the SSM Agent to send specific metrics and logs.
In the example below, we send Windows event logs and Memory Usage to CloudWatch.

```json
{
    "IsEnabled":true,
      "EngineConfiguration": {
        "PollInterval": "00:00:02",
        "Components": [
            {
                "Id": "ApplicationEventLog",
                "FullName": "AWS.EC2.Windows.CloudWatch.EventLog.EventLogInputComponent,AWS.EC2.Windows.CloudWatch",
                "Parameters": {
                    "LogName": "Application",
                    "Levels": "7"
                }
            },
            {
                "Id": "SystemEventLog",
                "FullName": "AWS.EC2.Windows.CloudWatch.EventLog.EventLogInputComponent,AWS.EC2.Windows.CloudWatch",
                "Parameters": {
                    "LogName": "System",
                    "Levels": "7"
                }
            },
            {
                "Id": "SecurityEventLog",
                "FullName": "AWS.EC2.Windows.CloudWatch.EventLog.EventLogInputComponent,AWS.EC2.Windows.CloudWatch",
                "Parameters": {
                "LogName": "Security",
                "Levels": "7"
                }
            },
            {
                "Id": "PerformanceCounter",
                "FullName": "AWS.EC2.Windows.CloudWatch.PerformanceCounterComponent.PerformanceCounterInputComponent,AWS.EC2.Windows.CloudWatch",
                "Parameters": {
                    "CategoryName": "Memory",
                    "CounterName": "Available MBytes",
                    "InstanceName": "",
                    "MetricName": "AvailableMemory",
                    "Unit": "Megabytes",
                    "DimensionName": "InstanceID",
                    "DimensionValue": "{instance_id}"
                }
            },
            {
                "Id": "CloudWatchLogs",
                "FullName": "AWS.EC2.Windows.CloudWatch.CloudWatchLogsOutput,AWS.EC2.Windows.CloudWatch",
                "Parameters": {
                    "AccessKey": "",
                    "SecretKey": "",
                    "Region": "eu-west-1",
                    "LogGroup": "ssmagent",
                    "LogStream": "{instance_id}"
                }
            },
            {
                "Id": "CloudWatch",
                "FullName": "AWS.EC2.Windows.CloudWatch.CloudWatch.CloudWatchOutputComponent,AWS.EC2.Windows.CloudWatch",
                "Parameters": 
                {
                    "AccessKey": "",
                    "SecretKey": "",
                    "Region": "eu-west-1",
                    "NameSpace": "Windows/Default"
                }
            }
        ],
        "Flows": {
            "Flows": 
            [
                "(SystemEventLog,ApplicationEventLog,SecurityEventLog),CloudWatchLogs",
                "PerformanceCounter,CloudWatch"
            ]
        }
    } 
}
```

Fist thing is to enable the collect.
`"IsEnabled":true,`

And configure the Poll interval.
`"PollInterval": "00:00:02",`

Then you have to
[configure what you want to collect](http://docs.aws.amazon.com/systems-manager/latest/APIReference/aws-cloudWatch.html).
In this example we have configured Windows Event logs.


```json
{
    "Id": "ApplicationEventLog",
    "FullName": "AWS.EC2.Windows.CloudWatch.EventLog.EventLogInputComponent,AWS.EC2.Windows.CloudWatch",
    "Parameters": {
        "LogName": "Application",
        "Levels": "7"
    }
}
```

The level describes what type of log you want to collect:
* 1 - Only error messages uploaded.
* 2 - Only warning messages uploaded.
* 4 - Only information messages uploaded.
* 7 - All of above.

We also configured it to send memory usage.

```json
{
    "Id": "PerformanceCounter",
        "FullName": "AWS.EC2.Windows.CloudWatch.PerformanceCounterComponent.PerformanceCounterInputComponent,AWS.EC2.Windows.CloudWatch",
        "Parameters": {
            "CategoryName": "Memory",
            "CounterName": "Available MBytes",
            "InstanceName": "",
            "MetricName": "AvailableMemory",
            "Unit": "Megabytes",
            "DimensionName": "InstanceID",
            "DimensionValue": "{instance_id}"
        }
}
```

This section needs to refer to an existing Windows Performance Counter available
in Performance Monitor.
  
<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/05.png" alt="XXX" width="627" style="margin: 0px auto;display:block;" />

The CategoryName refers to the Category, and the CounterName must be the same as
a specific counter (case sensitive).

<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/06.png" alt="XXX" width="722" style="margin: 0px auto;display:block;" />

The DimensionName refers to the name of the Metrics in CloudWatch.

<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/07.png" alt="XXX" width="465" style="margin: 0px auto;display:block;" />

The DimensionValue refers to the name used to tag the metric in CloudWatch. In
our example it refers to the InstanceID.
  
<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/08.png" alt="XXX" width="800" style="margin: 0px auto;display:block;" />

If you want to add more than one performance counter, you just have to add a
performance counter section in your json.

```json
{
    "Id": "PerformanceCounter2",
        "FullName": "AWS.EC2.Windows.CloudWatch.PerformanceCounterComponent.PerformanceCounterInputComponent,AWS.EC2.Windows.CloudWatch",
            "Parameters": {
            "CategoryName": "System",
            "CounterName": "Context Switches/sec",
            "InstanceName": "",
            "MetricName": "ContextSwitches",
            "Unit": "Count/Second",
            "DimensionName": "InstanceID",
            "DimensionValue": "{instance_id}"
        }
}
```

The next step is to configure where you want to send metrics in CloudWatch.

```json
{
    "Id": "CloudWatchLogs",
    "FullName": "AWS.EC2.Windows.CloudWatch.CloudWatchLogsOutput,AWS.EC2.Windows.CloudWatch",
    "Parameters": {
        "AccessKey": "",
        "SecretKey": "",
        "Region": "eu-west-1",
        "LogGroup": "ssmagent",
        "LogStream": "{instance_id}"
    }
}
```

This section specifies in which stream you want to send your logs into. In our
example, logs will be stored in a LogGroup named `ssmagent`, and the stream will
be named after the instance ID.
  
<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/09.png" alt="XXX" width="800" style="margin: 0px auto;display:block;" />

<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/10.png" alt="XXX" width="542" style="margin: 0px auto;display:block;" />

```json
{
    "Id": "CloudWatch",
    "FullName": "AWS.EC2.Windows.CloudWatch.CloudWatch.CloudWatchOutputComponent,AWS.EC2.Windows.CloudWatch",
        "Parameters": {
            "AccessKey": "",
            "SecretKey": "",
            "Region": "eu-west-1",
            "NameSpace": "Windows/Default"
        }
}
````
       
This section specifies the NameSpace where metrics will be stored.

<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/11.png" alt="XXX" width="786" style="margin: 0px auto;display:block;" />

Then, when you have defined all your metrics and configured CloudWatch, you have
to specify where you want to send each of these metrics in the `flows` section.

```json
{
    "Flows": {
        "Flows": 
        [
            "(SystemEventLog,ApplicationEventLog,SecurityEventLog),CloudWatchLogs",
            "PerformanceCounter,CloudWatch"
        ]
    }
}
```

In our example, we send the event logs in CloudWatchLogs, and the Performance
Counter in CloudWatch.

## Configure the SSM Agent with Systems Manager

You can configure the
[SSM Agent](http://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/send_logs_to_cwl_instances.html)
via the AWS Management Console.

In EC2/Systems Manager Service/Run Command, click on `Run a Command`.
  
<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/12.png" alt="XXX" width="595" style="margin: 0px auto;display:block;" />

Select `AWS-ConfigureCloudWatch` and your target instances.
  
<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/13.png" alt="XXX" width="800" style="margin: 0px auto;display:block;" />

<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/14.png" alt="XXX" width="800" style="margin: 0px auto;display:block;" />

In properties, paste your configured JSON (AWS.EC2.Windows.CloudWatch.json).
  
<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/15.png" alt="XXX" width="800" style="margin: 0px auto;display:block;" />

And click `Run`.
  
<img src="/assets/2017-05-19-windows-performance-counters-and-event-logs-monitoring/16.png" alt="XXX" width="800" style="margin: 0px auto;display:block;" />

Congrats! Your SSM Agent is now fully configured. 

## Configure the SSM Agent manually

Systems Manager Service is not yet available in all AWS region but you can use
SSM Agent even if Systems Manager is not available in your region. To push the
configuration on your instances you will have to add the
`AWS.EC2.Windows.CloudWatch.json` file in
`%programfiles%\Amazon\SSM\Plugins\awsCloudWatch` and restart the Amazon SSM
Agent service. Logs are available in `%PROGRAMDATA%\Amazon\SSM\Logs`. You may
see the following error messages:

*[MessageProcessor] error when calling AWS APIs. error details - GetMessages
Error: RequestError: send request failed caused by: Post
https://ec2messages.eu-west-2.amazonaws.com/: dial tcp: lookup
ec2messages.eu-west-2.amazonaws.com: getaddrinfow: No such host is known.*

or 

*[HealthCheck] error when calling AWS APIs. error details - RequestError: send
request failed caused by: Post https://ssm.eu-west-2.amazonaws.com/: dial tcp:
lookup ssm.eu-west-2.amazonaws.com: getaddrinfow: No such host is known.*

## Terraform

You can use the code below to automate the creation of the policy/role with Terraform.

Terraform does not currently support Systems Manager Run Command, but a
[request for enhancement is open](https://github.com/hashicorp/terraform/issues/13271).
You can also add the JSON file in your AMI and add the IAM Role when you launch
your new instances.

## Powershell

XXX Add powershell code Jérémie Rodon.

That’s it for monitoring Performance Counters and event logs with SSM Agent and
CloudWatch. You can also use Systems Manager to store passwords, license keys or
database connection strings. It will be covered soon in another post.
