---
author: Gauthier Wallet
layout: post
title: "My Journey into Terraform: Part 1"
twitter: Ninnir
---

## Introduction
At [D2SI](http://d2-si.eu) we truly believe in automating IT infrastructures. As
a DevOps working for a company that promotes automation, one of your main
objectives is to keep seeking the right tools and sometimes bet on them.
Terraform, among many others, is one of them.

[Terraform](https://www.terraform.io/intro/index.html) is an open-source tool
for building, modifying and versioning infrastructure safely and efficiently.
It helps ops (OPerationS) benefit from a concept called **IaC** (Infrastructure
as code) for describing infrastructures based on a declarative language called
**HCL** (HashiCorp Configuration Language). Here is an example that will be
interpreted into an API call creating an AWS instance (aka virtual machine) of
type `t2.nano` (500MB RAM, 1vCPU) and based on the AMI ID `ami-e5083683` (Amazon
Linux 2017.03.0):

```hcl
# main.tf
resource "aws_instance" "example" {
  ami           = "ami-e5083683"
  instance_type = "t2.nano"
}
```

Using the command-line tool, you can then interact with this configuration and
apply your changes. The journey can now begin!

## Getting deeper
With a developer background and currently working as an ops (yeah you got it,
"DevOps"), it brings me the logic and flexibility I need to build
infrastructures for my clients like provisioning an entire **VPC** (Virtual
Private Cloud), managing load balancers, databases, deploying serverless
applications, etc.

A few months ago, I was working on a project that was processing raw data to
structured one for display purposes. We needed to
[deliver notification messages to a WebApp](http://docs.aws.amazon.com/sns/latest/dg/SendMessageToHttp.html)
using a password-protected HTTPS endpoint that looked like this:
`https://username:password@example.com/myroute`.

The tool used for this is called **SNS**: Simple Notification Service. It is an
AWS managed service allowing users to send notifications to multiple endpoints
using a defined protocol: email, HTTP, SMS, etc. To proceed, users first need
to create a topic and then add subscriptions (that must be acknowledged). If we
were to use the AWS cli, it would look like:

```bash
aws sns subscribe --topic-arn arn:aws:sns:eu-west-1:123456789098:my-topic --protocol http --notification-endpoint https://username:password@example.com/myroute
```

This would create a subscription for the
`https://username:password@example.com/myroute` endpoint, linked to the
`arn:aws:sns:eu-west-1:123456789098:my-topic` topic using the `http` protocol.

The documentation explicitly states that the endpoint has to auto-confirm the
unique URL that is sent by AWS, by making a call to it. However, even though the
confirmation was made on the AWS side and the console successfully exposed it,
the Terraform abstraction was not aware of it, resulting in a timeout.

## Fixing the matrix
I then realized that fixing this bug would be the perfect opportunity for me to
contribute to the Terraform project. So I started looking at the code and
learning the [Go](https://golang.org/) programming language which ended up
being a double win for me. Slowly getting into the codebase, I started
documenting my findings and exploring what was already being worked on in the
various pull requests.

Once I gained enough confidence and in order to find that bug, I got through
the process of
[developping Terraform](https://github.com/hashicorp/terraform#developing-terraform).
Being all set-up, I started my debugging session:

```bash
TF_LOG=DEBUG TF_LOG_PATH=terraform.log terraform apply
```

Here are a few hints on what is done:
* `TF_LOG=DEBUG`: set the log level to DEBUG, which allows to trace almost
everything
* `TF_LOG_PATH`: logs everything into the `terraform.log` file

Taking our previous example, i.e.
`https://username:password@example.com/myroute`, I found out that the endpoint
returned by AWS when subscribing contained an obfuscated string, replacing the
password field. What it means is that when I sent:

    https://username:password@example.com/myroute

I got this in return:

    https://username:****@example.com/myroute

This led me to the
[bug](https://github.com/hashicorp/terraform/blob/8ea5d53954baf9b2a24963f1d69a4343864409d5/builtin/providers/aws/resource_aws_sns_topic_subscription.go#L277):
it was comparing the sent value (my endpoint) with the returned one
(obfuscated), which would obviously always return false. So I created a small
function reproducing the obfuscation using Go:

```go
// returns the endpoint with obfuscated password, if any
func obfuscateEndpointPassword(endpoint string) string {
	r := regexp.MustCompile("(://[^:]+):([^@]+)")
	return r.ReplaceAllString(endpoint, fmt.Sprintf("$1:%s", awsSNSPasswordObfuscationPattern))
}
```

This simple code block will translate any URL into its corresponding obfuscated
version, as in:

| URL | Obfuscated version |
|-----|--------------------|
| "https://example.com/myroute" | "https://example.com/myroute" |
| "https://username@example.com/myroute" | "https://username@example.com/myroute" |
| "https://username:password@example.com/myroute" | "https://username:****@example.com/myroute" |

With the code written & unit tests passing, I fixed the buggy instruction and
threw my first contribution to Terraform. All the work resides on
[GitHub](https://github.com/hashicorp/terraform/pull/9696) and should be merged
soon!

## Journey ain't over
As a developer, I have to admit I fell in love with Go and the Terraform API.
This work was the first experience I had with the Terraform codebase and, under
the D2SI flag, I have since contributed many documentation fixes, multiple new
resources, fixed dozens of bugs and helped people. The journey into Terraform
just started, stay tuned for the next episode!
