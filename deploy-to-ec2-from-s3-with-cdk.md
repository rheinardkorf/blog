---
title: Deploy to EC2 from S3 with CDK
seoDescription: Using CDK to deploy an EC2 instance sourced with an S3 bucket.
---

I had a need to use CDK Pipelines to deploy an EC2 service. The code for the pipeline is part of the app repo. In building the construct I quickly ran into some issues:  

* The code is in a private repo.
* There is no simple sources option in the EC2 core constructs that accepts sources.

This makes things tricky. Do I create a shared SSH keypair and add it to the repo (and put the private key somewhere)? Do I use Personal Access Tokens (my least preferred option because of the "blast radius")? Do I make the repo public?

All I wanted was to use a source artifact as my source for the EC2 instance. Fortunately I found an initial workaround here: [Bootstrapping an Amazon EC2 instance using user-data to run a Python web app](https://www.buildon.aws/tutorials/using-ec2-userdata-to-bootstrap-python-web-app?ref=dc&id=redirect#adding-user-data-to-your-ec2-instance). Not my goal, but an insight to a solution!

The steps:  

* Use `aws-cdk-lib/aws-s3-assets` Asset to upload my files as a zipped archive to S3.
* Use the `userData.addS3DownloadCommand` to download the asset and get a file reference for later.
* Create a script that will be run on instance creation, accepting one argument (a file reference). This will be the zipped file that can be unzipped at a desired location on the EC2 instance. 
* Use `aws-cdk-lib/aws-s3-assets` Asset to upload the `userData` script to S3.
* Use the `userData.addExecuteFileCommand` to execute the script, passing in the file reference as the only argumeng.

Just remember to allow the EC2 instance to read the bucket. 

Okay! Lets do this...

```typescript
// cdk/lib/my-stack.ts
// This assumes you already created an `aws-cdk-lib/aws-ec2` Instance object.
// Also, this is CDKv2.

import { 
  // ...
  aws_ec2 as ec2, 
  aws_s3_assets as s3assets,
  // ...
} from 'aws-cdk-lib';
import * as path from 'path';
import { Construct } from 'constructs';

export class MyStack extends ec2.Stack {
  constructor(scope: Construct, id: string, props: ec2.StackProps) {
    super(scope, id, props);

    // ... above create an object called ec2Instance.

    // Some files should be excluded... .gitignore is a good source.
    const excludePattern = [ "node_modules", "dist", ".cdk.staging", "cdk.out" ];

    // Step 1: Lets upload the project. Path is relative to this file.
    const projectAsset = new s3assets.Asset(this, "ProjectFiles", {
        path: path.join(__dirname, "../.."),
        exclude: excludePattern
    });
    // Allow EC2 to access the asset.
    projectAsset.grantRead(ec2Instance.role);

    // Step 2: Download the project zip file on the instance and get the reference.
    const projectZipFilePath = ec2Instance.userData.addS3DownloadCommand({
        bucket: projectAsset.bucket,
        bucketKey: projectAsset.s3ObjectKey,
    });

    // Step 3: Upload the configuration user script. Path is relative to this file.
    const configScriptAsset = new s3assets.Asset(this, "ProjectInstanceConfig", {
        path: path.join(__dirname, "user-data-script.sh"),
    });
    // Allow EC2 instance to read the file
    configScriptAsset.grantRead(ec2Instance.role);

    // Step 4: Download the project config file get the reference.
    const configScriptFilePath = ec2Instance.userData.addS3DownloadCommand({
        bucket: configScriptAsset.bucket,
        bucketKey: configScriptAsset.s3ObjectKey,
    });

    // Step 5: Execute the script on the instance passing in the zip reference.
    ec2Instance.userData.addExecuteFileCommand({
        filePath: configScriptFilePath,
        arguments: projectZipFilePath,
    });

    // ... rest of the stack.
  }
}
```

And now the config file...

```bash
#!/bin/bash -xe

# Read the first parameter into $PROJECT_ZIP
if [[ "$1" != "" ]]; then
    PROJECT_ZIP="$1"
else
    echo "No project to deploy."
    exit 1
fi

# Instance apps and dependencies.
yum update -y
yum groupinstall -y "Development Tools"

# Extract the project zip at desired location.
mkdir -p /opt/the-project
cp $PROJECT_ZIP /opt/the-project/the-project.zip
cd /opt/the-project
unzip -o the-project.zip
rm the-project.zip

# Any other commands you want to execute.

```

Hope this was as useful to you as it was for me.



[//begin]: # "Autogenerated link references for markdown compatibility"
[ "$1" != "" ]: -$1-= " "$1" != "" "
[//end]: # "Autogenerated link references"