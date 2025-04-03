---
layout: post
title:  "AWS Native Scanning in ECR has a Gotcha"
date: 2024-12-13  14:30:00 -0700
tags: aws computers
categories: computers aws
---

# AWS Native Scanning in ECR: Gotcha!
TL;DR: If you switch to using AWS Native scanning in ECR, any scans done with the previous Clair scanner will no longer be accessible. In some cases, you can revert the setting and see scan results again, but I've got a 50% success rate, seemingly based on the number of images in the registry. Also, the Describe Images API will no longer report image vulnerability scan findings.

## Background
A great feature of [AWS Elastic Container Registry](https://aws.amazon.com/ecr) is the native functionality to scan new container images when they are pushed to a registry. It's a quick and easy way to run a sanity check on a container image. Up until recently, AWS relied on the open-source [Clair](https://github.com/quay/clair) image scanner, but on [August 6th, 2024](https://aws.amazon.com/about-aws/whats-new/2024/08/new-version-amazon-ecr-basic-scanning/) this changed with little fanfare. I don't remember hearing about it, and a quick look through past editions of the Last Week in AWS newsletter don't mention it either. Philosophers continue to debate if announcement is made, but if nobody blogged/tweeted about it, did it really happen? Even people who track AWS releases skipped over this one. Frankly, it isn't a big deal. AWS changes things all the time, introducing new features and products. The cloud changes underneath our feet, after all. Perhaps the **bigger** news here is a user can actually set their preference. After all, [GuardDuty's Malware Protection for S3](https://docs.aws.amazon.com/guardduty/latest/ug/how-malware-protection-for-s3-gdu-works.html) feature doesn't let users pick the scanning engine. Prior to ECR's change, the options was "Basic" or "Enhanced", where Basic scans were on-push and manually triggered (limit of one scan per image per day) and Enhanced scans fed into Inspector and could be scheduled.

So when I logged in to an account, intent on delivering shareholder value by reporting on vulnerabilities in images, and ECR told me there was a suggested upgrade, I naturally was suspicious. That's because I naturally am suspicious. I looked over the [linked documentation](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-improved-basic-enabling.html) and saw (and confirmed) new registries used the new version and old registries would use the old version. _Note to self: build a tool to iterate through all registries in all regions of all accounts to flip this switch_ I also looked into the supported Operating Systems and found (at time of writing) [AWS native scanning supports all the OSs Clair did, plus more](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-basic.html#image-scan-basic-support-operating-systems). This seems great! No change in cost, no change in usage, just flip the switch conveniently offered in the console.

## Ah, but there's a catch
Friends. I clicked the button. And then, to quote Col. Gathers, "There is no good news! Just bad news and weird news." Immediately, the past four years of container scans disappeared. Gone. Poof. Deeply suspicious of the console owing to a long history with "eventual consistency", I tried using the AWS CLI. Nothing. I switched the setting back, but scan results did not return to the console nor did results appear in the CLI. Annoyed, I flipped the switch _back_ to use `AWS_Native` scanning, figuring at least new images would be scanned, and resolved to do some testing and perhaps blogging.

## Here's what I've learned
* Firstly, this is actually [documented](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)! However, the "Learn More" hypertext in the console takes you to a [page](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-improved-basic-enabling.html) that doesn't reiterate this warning. Miss #1.
* Secondly, in smaller registries, switching between versions of scanning engines seems to work fine. Larger registries probably work eventually.
* Thirdly, the APIs are a mess. Miss #2
* Finally, the permissions repeats past mistakes. Miss #3

## Messy APIs
### [Get|Put]AccountSetting
Friends. The ECR APIs are trying their best. To start, none of the existing ECR APIs will tell you what scanner version you're using. You need to use the new Get/Put Account Setting API. And just like the mistakes of S3 Public Access Block, the permission for this API is simply `ecr:PutAccountSetting`. As an administrator, it isn't unreasonable to want to allow users to control their own upgrade, or prevent downgrading to the older version of scanning. But here, it is all-or-nothing. Users with this permission can change any account setting, which at time of writing, is _just_ the scan version... but that could increase in the future. An improvement would be the ability to craft an SCP where downgrading is not allowed, or _only_ the scanner version can be set.

At least for now, you can't do anything too dangerous with this permission, unlike S3 PAB.

### The Describe Image Scan-way or the Highway
(That would have been a great title. Damn. Next time.)
Querying for scan results must now be done via the Describe Image Scan Findings API, _and that API only_. ***Any tooling which relied on scan results showing up in the DescribeImages API will now tell you there are no vulnerabilities.*** This is... actually really important. If you have tooling which relies on the old version of the unversioned API, you will not see vulnerability scan information. And if you choose to upgrade, perhaps because your tooling already uses this API, you will not see vulnerability scan information for images _unless_ manual scans are triggered for existing images, even if they were scanned by ECR with Clair previously. See the terminal commands and output below:

```
❯ aws ecr put-account-setting --name BASIC_SCAN_TYPE_VERSION --value AWS_NATIVE
{
    "name": "BASIC_SCAN_TYPE_VERSION",
    "value": "AWS_NATIVE"
}

❯ aws ecr describe-image-scan-findings --repository-name "testing-madness" --image-id imageDigest=sha256:f9656f8fed6685bae8c7cd02c24b4d15f46e11764bdc149a25313f8ba19e356c

An error occurred (ScanNotFoundException) when calling the DescribeImageScanFindings operation: Image scan does not exist for the image with '{imageDigest:'sha256:f9656f8fed6685bae8c7cd02c24b4d15f46e11764bdc149a25313f8ba19e356c', imageTag:'null'}' in the repository with name 'testing-madness' in the registry with id '853082215256'

❯ aws ecr put-account-setting --name BASIC_SCAN_TYPE_VERSION --value CLAIR
{
    "name": "BASIC_SCAN_TYPE_VERSION",
    "value": "CLAIR"
}

❯ aws ecr describe-image-scan-findings --repository-name "testing-madness" --image-id imageDigest=sha256:f9656f8fed6685bae8c7cd02c24b4d15f46e11764bdc149a25313f8ba19e356c
{
    "imageScanFindings": {
        "findings": [
            {
                "name": "CVE-1842-1084",
                "description": "An out-of-bounds exception was found in pocketwatches wherein if the mainspring is allowed to unwound completely, the device will lose the ability to keep accurate time even after reapplication of tension to the mainspring.",
                ...snip...
```

The "describe-image-scan-findings" API also does not convey information about the version of the scanner used to generate these results. What about other APIs?
```
❯ aws ecr describe-registry
{
    "registryId": "853082215256",
    "replicationConfiguration": {
        "rules": []
    }
}
```
No mention of the scanner version in use here. You need to query the `get-account-setting` API for that and look at the response. Well, once upon a time, ECR allowed per-repository scan settings which were since superceeded by the registry/account level controls. Let's list the repositories and see if they contain any useful information.
```
❯ aws ecr describe-repositories
{
    "repositories": [
        {
            "repositoryArn": "arn:aws:ecr:us-west-2:853082215256:repository/testing-madness",
            "registryId": "853082215256",
            "repositoryName": "testing-madness",
            "repositoryUri": "853082215256.dkr.ecr.us-west-2.amazonaws.com/testing-madness",
            "createdAt": "2023-08-08T19:35:35.828000-07:00",
            "imageTagMutability": "MUTABLE",
            "imageScanningConfiguration": {
                "scanOnPush": false
            },
            "encryptionConfiguration": {
                "encryptionType": "AES256"
            }
        }
    ]
}
```
Ironically, this is almost even worse, since _all_ images are scanned on push. But technically, in the dusty corner of the console where per-repository settings are kept, this setting is off. (Images are, however, still being scanned!) Further, it isn't clear to the consumer if Basic or Enhanced scanning is configured. For that, you have to ask the `get-registry-scanning-configuration` API, which will return directly contradictory information. This API will also not inform the requestor of which version scanner is in use.

```
❯ aws ecr put-account-setting --name BASIC_SCAN_TYPE_VERSION --value AWS_NATIVE
{
    "name": "BASIC_SCAN_TYPE_VERSION",
    "value": "AWS_NATIVE"
}

❯ aws ecr get-registry-scanning-configuration
{
    "registryId": "853082215256",
    "scanningConfiguration": {
        "scanType": "BASIC",
        "rules": [
            {
                "scanFrequency": "SCAN_ON_PUSH",
                "repositoryFilters": [
                    {
                        "filter": "*",
                        "filterType": "WILDCARD"
                    }
                ]
            }
        ]
    }
}

❯ aws ecr put-account-setting --name BASIC_SCAN_TYPE_VERSION --value CLAIR
{
    "name": "BASIC_SCAN_TYPE_VERSION",
    "value": "CLAIR"
}

❯ aws ecr get-registry-scanning-configuration
{
    "registryId": "853082215256",
    "scanningConfiguration": {
        "scanType": "BASIC",
        "rules": [
            {
                "scanFrequency": "SCAN_ON_PUSH",
                "repositoryFilters": [
                    {
                        "filter": "*",
                        "filterType": "WILDCARD"
                    }
                ]
            }
        ]
    }
}

```
As promised. Contradictory information. This API won't return the scanner version, but will correctly report images pushed to the repository will be scanned.

## Conclusion
There are, as demonstrated, some gotchas to ECR's new native image scanner which tarnishes this quietly-launched feature.

1. You must do `aws ecr get-account-setting --name BASIC_SCAN_TYPE_VERSION` to see what is configured for your account
2. The Get|Put Account Setting API is protected by new `ecr:` permissions, but cannot be restricted to _only_ seeing the `BASIC_SCAN_TYPE_VERSION` parameter nor _only_ allowing the version to be set in one direction.
3. The `aws ecr describe-repositories` command outright lies about the scan-on-push configuration of the repositories in the account, likely as a historical artifact.
4. Upgrading to the AWS Native scanner breaks the Describe Images API by removing some fields. This can make vulnerabilities suddenly invisible to tools which relied on the format of the Describe Images API response. Yes, it's JSON, and yes, a JSON library might throw an exception on a missing key... but come on. From personal experience, configuration of JSON parsing varies wildly, and it is very easy to overlook the API continuing to work (no errors) and software to parse the lack of a key as "null", which gets processed into "Image X has 0 vulnerabilities", which doesn't set off any alerts.
   * I assume this was done as a form of cost-savings. For my teeny repository with just four images, the new response is just under half the size of the old response.
   * `aws ecr put-account-setting --name BASIC_SCAN_TYPE_VERSION --value AWS_NATIVE && aws ecr describe-images --repository-name "testing-madness" | wc -c`: 2468 bytes
   * `aws ecr put-account-setting --name BASIC_SCAN_TYPE_VERSION --value CLAIR && aws ecr describe-images --repository-name "testing-madness" | wc -c`: 4992 bytes
   * The irony of AWS trying to save on their bandwidth bills is not lost on me

### Asks
I don't like showing up to a problem without a solution. So here's a list, in no particular order, of some ways this could be improved:
* The registry scanning configuration API should report _everything_ about the registry scanning configuration, including the scanner version
* The describe image scan findings API should _not_ suddenly report _nothing_ when the scanner version changes. The API could include what scanner reporting a finding. For a short period of time (6-AUG-24 - 1-OCT-25) there could be duplicates if an image was scanned with Clair, then the registry was changed to AWS Native, and the image was scanned again. I don't think there would be too many duplicates, since only one scanner can be active at a time, and it takes extra work which more than anything makes it less likely to happen. AWS would still be saving bandwidth.
* Consider, when adding a new `Put` class permission, how administrators could exert control over _what_ is being `Put`.
* Describing a repository should show the correct configuration, which will be dictated by the account-level setting _unless_ the registry has legacy-mode scan-on-push enabled _and_ the account-level setting includes a filter which would otherwise exclude that repository based on the name. The API should do the work to give me a correct answer.
* Add warnings to the correct documentation pages. Putting the effort into documenting something, but then not linking to it, is an unforced error.
* Providing free container image scans to everyone is great. Despite these flaws, it is much appreciated to have such an easy security win. Please keep it up.
