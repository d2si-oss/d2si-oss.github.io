---
author: Guy Rodrigue Koffi
layout: post
title: "Cleaning your Amazon Machine Images (AMI)"
twitter: bonclay7
keywords: aws, ec2, ami, continous integration, continous deployment
---

## Introduction
At D2SI, we help companies operate their workloads in the cloud either by moving existing ones or going from scratch.
One of the first steps is often `continuous integration` and `continous deployment`.
A common and simple pattern I've seen on the **AWS** cloud is:

- **AMI (Amazon Machine Image)** baking to prepare future instance to start with the newest application version, steps including (more or less):
  * System and kernel updates
  * Security patches
  * Application package dependencies
  * Application code deployment
- Using **CloudFormation** or **Terraform** to then deploy/update the stack.

It's pretty basic and works very well, but after many deployments we end up cumulating unecessary old and unsued __AMIs__.
Well, someone got the do the dirty work of cleaning, because they are not for free *and your listing can get messy*.


## Why bother creating another tool ?

The answer is simple, if you've ever done that you probably know that an AMI can be associated to one or more EBS snapshots.
The operation to **deregister** an AMI does not remove associated snapshots.

When I started to think about doing this in a automated way,
I had only one requirement to just remove an AMI with it's snapshots, so I started with a very basic shell script like this:

```bash
for ami in $ami_ids; do
  snap=$(aws ec2 describe-images --image-ids "$ami" --query 'Images[*].BlockDeviceMappings[*].Ebs.SnapshotId' --output text)
  aws ec2 deregister-images --image-id "$ami"
  aws ec2 delete-snapshot --snap-id "$snap"
done
```

Sooner, my list of requirements grew up and I had to maintain this script wich became a real pain with my level of bash scripting.
I decided then to move it to **python** because:
 * **python** is easy, people with whom I work could contribute
 * **AWS** has a great python sdk: boto3
 * As a great fan of Test Driven Developement, I could use quite easily **moto**, a python library to mock AWS endpoints.


## Introducing aws-amicleaner

I created this project on [Github](https://github.com/bonclay7/aws-amicleaner) and packaged it also on **pypi**.
You can install it with:

```bash
pip install aws-amicleaner
```

Showing current version:

```console
$ amicleaner --version
0.1.2
```

The first thing I had to do was to cover the very basic first requirement: remove a list of AMI ids with their snaphots:

```bash
amicleaner --from-ids ami-12345678 ami-23456789
```

Well, it does more than that. Here's the list of all the features included in this project at this time:

* Removing a list of images and associated snaphots
* Mapping AMIs:
  * Using names
  * Using tags
* Filtering AMIs:
  * being used by running instances.
  * from autoscaling groups (launch configurations) with desired capacity set to 0.
  * from launch configurations detached from autoscaling groups.
* Specifying how many AMIs you want to keep
* Cleaning orphan snapshots
* A bit of reporting


## Diving in the features

Now, let's describe the tool features with a real life example.
Let's say I have 20 images to be deleted with two tags (`environment` and `role`) and categorized as following:

* 5 `front` images in `dev` and `prod`
* 5 `back` images in `dev` and `prod`

In this example, I have neither running instances nor autoscaling groups / launch configurations.


<div align="center">
<img src="/assets/2017-06-16-cleaning-your-amazon-machine-images/amis-list.png" alt="images list">
</div>


### Historization

When you launch amicleaner without any arguments, the tool will 'inspect' your enviroment and look for
the images that you own. From the list of items to be deleted, it will regroup them by using AWS tags.


```console
$ amicleaner

Default values : ==>
mapping_key : tags
mapping_values : ['environment', 'role']
keep_previous : 4

Retrieving AMIs to clean ...

AMIs to be removed:
+--------------------+------------+
|     Group name     | candidates |
+--------------------+------------+
|      back.dev      |     1      |
|     back.prod      |     1      |
|     dev.front      |     1      |
|     front.prod     |     1      |
| no-tags (excluded) |     1      |
+--------------------+------------+
Do you want to continue and remove 4 AMIs [y/N] ? :
```

For rollback purposes, you can keep a certain number of previous images for each group of images ready to launch your applications in any case of failure.
amicleaner provides this functionality by default, by excluding already currently used AMIs and a additionnal number of `4` versions.
You can override this behaviour by specifying the `keep-previous` argument. The minimum value here can be down to `0` and they are sorted by creation date.

```console
$ amicleaner --keep-previous 2                                                                                           [ruby-2.4.1p111]

Default values : ==>
mapping_key : tags
mapping_values : ['environment', 'role']
keep_previous : 2

Retrieving AMIs to clean ...

AMIs to be removed:
+--------------------+------------+
|     Group name     | candidates |
+--------------------+------------+
|      back.dev      |     3      |
|     back.prod      |     3      |
|     dev.front      |     3      |
|     front.prod     |     3      |
| no-tags (excluded) |     1      |
+--------------------+------------+
Do you want to continue and remove 12 AMIs [y/N] ? :
```

Note that values not matching the search are automatically excluded: `no-tags (excluded)`

You can see what's being done behind the [scenes](https://github.com/bonclay7/aws-amicleaner/blob/master/amicleaner/core.py#L225-L245):

```python
def reduce_candidates(self, mapped_candidates_ami, keep_previous=0):

    """
    Given a array of AMIs to clean this function return a subsequent
    list by preserving a given number of them (history) based on creation
    time and rotation_strategy param
    """

    if not keep_previous:
        return mapped_candidates_ami

    if not mapped_candidates_ami:
        return mapped_candidates_ami

    amis = sorted(
        mapped_candidates_ami,
        key=self.get_ami_sorting_key,
        reverse=True
    )

    return amis[keep_previous:]
```

### Mapping and filtering

By default, amicleaner use two tags to perform the search `environment` and `role`.
You can provide your own tags by specifying your `mapping-key`. The supported values are:

* `name`:

Here, it will use the name of the AMI to do the search. You are free to give only a part of the name.
You can alse use the `full-report` option for more details

```console
$ amicleaner --mapping-key name --mapping-values front --full-report --keep-previous 8                                   [ruby-2.4.1p111]

Default values : ==>
mapping_key : name
mapping_values : ['front']
keep_previous : 8

Retrieving AMIs to clean ...
front
+--------------+---------------+--------------------------+
|    AMI ID    |    AMI Name   |      Creation Date       |
+--------------+---------------+--------------------------+
| ami-50504e36 | front-a3c56c4 | 2017-06-15T09:55:42.000Z |
| ami-52504e34 | front-ff5eac7 | 2017-06-15T09:56:43.000Z |
+--------------+---------------+--------------------------+


AMIs to be removed:
+------------+------------+
| Group name | candidates |
+------------+------------+
|   front    |     2      |
+------------+------------+
Do you want to continue and remove 2 AMIs [y/N] ? :
```

* `tags`:

Search is perform with a combination of given tag `keys`. The combination of same values are mapped togther and Historization is applied on the result.

Note that the order of tag keys are preserved during the evaluation.


```console
$ amicleaner --mapping-key tags --mapping-values environment role --full-report --keep-previous 3                        [ruby-2.4.1p111]

Default values : ==>
mapping_key : tags
mapping_values : ['environment', 'role']
keep_previous : 3

Retrieving AMIs to clean ...
back.dev
+--------------+--------------+--------------------------+
|    AMI ID    |   AMI Name   |      Creation Date       |
+--------------+--------------+--------------------------+
| ami-a74856c1 | back-285244c | 2017-06-15T11:47:57.000Z |
| ami-954658f3 | back-d358afc | 2017-06-15T11:46:56.000Z |
+--------------+--------------+--------------------------+


back.prod
+--------------+--------------+--------------------------+
|    AMI ID    |   AMI Name   |      Creation Date       |
+--------------+--------------+--------------------------+
| ami-55495733 | back-579c0f6 | 2017-06-15T11:53:07.000Z |
| ami-9d4759fb | back-c58de50 | 2017-06-15T11:52:05.000Z |
+--------------+--------------+--------------------------+


front.prod
+--------------+---------------+--------------------------+
|    AMI ID    |    AMI Name   |      Creation Date       |
+--------------+---------------+--------------------------+
| ami-fd4c529b | front-31d05d8 | 2017-06-15T10:05:39.000Z |
| ami-50514f36 | front-a4b33c9 | 2017-06-15T10:06:42.000Z |
+--------------+---------------+--------------------------+

dev.front
+--------------+---------------+--------------------------+
|    AMI ID    |    AMI Name   |      Creation Date       |
+--------------+---------------+--------------------------+
| ami-50504e36 | front-a3c56c4 | 2017-06-15T09:55:42.000Z |
| ami-52504e34 | front-ff5eac7 | 2017-06-15T09:56:43.000Z |
+--------------+---------------+--------------------------+

no-tags (excluded)
+--------------+-------------+--------------------------+
|    AMI ID    |   AMI Name  |      Creation Date       |
+--------------+-------------+--------------------------+
| ami-2bf48158 | test-delete | 2016-08-16T17:40:15.000Z |
+--------------+-------------+--------------------------+


AMIs to be removed:
+--------------------+------------+
|     Group name     | candidates |
+--------------------+------------+
|      back.dev      |     2      |
|     back.prod      |     2      |
|     dev.front      |     2      |
|     front.prod     |     2      |
| no-tags (excluded) |     1      |
+--------------------+------------+
Do you want to continue and remove 8 AMIs [y/N] ? :
```

Here's a piece of the code to flatten the tags :

```python
def tags_values_to_string(tags, filters=None):
    """
    filters tags(key,value) array and return a string with tags values
    :tags is an array of AWSTag objects
    """

    if tags is None:
        return None

    tag_values = []

    filters = filters or []
    filters_to_string = ".".join(filters)

    for tag in tags:
        if not filters:
            tag_values.append(tag.value)
        elif tag.key in filters_to_string:
            tag_values.append(tag.value)

            return ".".join(sorted(tag_values))
```


### Cleaning snapshots left orphan

Before writing this tool, I have been cleaning my images without removing the associated snapshots.

During the AMI-creation process, Amazon EC2 creates snapshots of your instance's root volume and any other EBS volumes
attached to your instance. You can read the full explanation on the AWS [documentation](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html)
Long story short, there's a `Description` property of the resulting snapshot filled with a string like this:

```
Created by CreateImage(i-06a469f2d64822242) for ami-654b5503 from vol-06caf0d5af00505e7
```

For snapshot cleaning, I do not take many risks. There's no rocket science, I call `describe-snaphot` ec2 api and filter with `Created by CreateImage*`


```python
def get_snapshots_filter(self):

    return [{
        'Name': 'status',
        'Values': [
            'completed',
        ]}, {
        'Name': 'description',
        'Values': [
            'Created by CreateImage*'
        ]
    }]

```

That's maybe a subject for improvement.


## Conclusion

There are some optimization that could be done, many improvements such as python3 support for example.
Feel free to use it, abuse it, open issues if you face some bugs or have another ideas and contribute with pull requests.

One more thing, in all the tests above, you saw that there's an interaction asking for confirmation. There is an `--force-delete` or `-f` that can be useful for automation purposes.
