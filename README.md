# serverless-transcribe

A simple web UI for [Amazon Transcribe](https://aws.amazon.com/transcribe/). Supports MP3, MP4, WAV, and FLAC audio without any fixed costs.

## How it Works

Once the project has been launched in [CloudFormation](https://aws.amazon.com/cloudformation/), you will have access to a webpage that allows users to upload audio files. The page uploads the files directly to [S3](https://aws.amazon.com/s3/). The S3 bucket is configured to watch for audio files. When it sees new audio files, an [AWS Lambda](https://aws.amazon.com/lambda/) function is invoked, which starts a transcription job.

Another Lambda function is triggered via [CloudWatch Events](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html) when the transcription job completes (or fails). An email is sent to the user who uploaded file with details about the job failure, or a raw transcript that is extracted from the job results.

The webpage is protected by HTTP Basic authentication, with a single set of credentials that you set when launching the stack. This is handled by an authorizer on the [API Gateway](https://aws.amazon.com/api-gateway/), and could be extended to allow for more complicated authorization schemes.

Amazon Transcribe currently has file limits of 2 hours and 1 GB.

### AWS Costs

The cost of running and using this project are almost entirely based on usage. Media files uploaded to S3 are set to expire after one day, and the resulting files in the transcripts bucket expire after 30 days. The Lambda functions have no fixed costs, so you will only be charged when they are invoked. Amazon Transcribe is "[pay-as-you-go](https://aws.amazon.com/transcribe/pricing/) based on the seconds of audio transcribed per month".

Most resources created from the CloudFormation template include a `Project` resource tag, which you can use for cost allocation. Unfortunately Amazon Transcribe jobs cannot be tracked this way.

## How to Use

The project is organized using a CloudFormation [template](https://github.com/farski/serverless-transcribe/blob/master/serverless-transcribe.yml). Launching a stack from this template will create all the resources necessary for the system to work.

### Requirements

- The stack must be launched in an AWS region that supports [SES](https://aws.amazon.com/ses/). There aren't [many of these](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/regions.html), unfortunately. The addresses that SES will send to and from are determined by your SES domain verification and sandboxing status.
- There must be a pre-existing S3 bucket where resources (like Lambda function code) needed to launch the stack can be found. This can be any bucket that CloudFormation will have read access to when launching the stack (based on the role used to execute the launch). The bucket must have object versioning enabled.

### Using the Deploy Script

Included in this project is a [deploy script](https://github.com/farski/serverless-transcribe/blob/master/deploy.sh). Launching and updating the stack using this script is probably easier than using the AWS Console. In order to use the script, create a `.env` file in the repository directory (look at `.env.example` for reference), and run `./deploy.sh` (You may need to `chmod +x deploy.sh` prior to running the script.)

The deploy script will zip all the files in the `lambdas/` directory, and push the zip files to S3 so that CloudFormation will have access to the code when creating the Lambda functions. The S3 destination is determined by the value of the `STACK_RESOURCES_BUCKET` environment variable (which is also passed into the stack as a stack parameter).

**Please note** that the template expects Lambda function code zip files' object keys to be prefixed with the stack's name. For example, if your `STACK_RESOURCES_BUCKET=my_code_bucket`, and you name the stack `my_transcription_app`, CloudFormation will look for a file such as:

```
s3://my_code_bucket/my_transcription_app/lambdas/TranscriptionJobStartFunction.zip
```

The deploy script will put files in the correct place in S3. If you chose to launch the stack through the Console you will need to create the zip files yourself, and ensure they end up in the correct bucket with the correct prefix.

Once the deploy script has finished running, it will print the URL for the upload webpage. You should be able to visit that page, enter the HTTP basic auth credentials you set, and upload a file for transcription.
