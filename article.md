# Cheap and scalable A/B testing with CloudFront


## Benefits

- no change to client web pages (no extra JavaScript to add )
- very cheap (contrast this to Optimizely or even CloudWatch evidently)
- no third-party code (reduces gdpr problems and privacy concerns)
- no visible performance impact on loading/transfer

## How this works

- a viewer request function on "/index.html" assigns a test cohort, and redirects between index.html and index2.html based on the cohort. it also adds an internal header to the request with the cohort assignment, so it could be returned back to the user as a cookie. it also logs cohort assignment to cloudwatch so we can get it out later.
- a viewer response function on "/index.html" checks for the internal header, and marks the user with a cookie (we use CSV to support multiple ongoing A/B tests)
- a different viewer response function on "/success.html" checks for the presence of a session cookie with cohort assignment, and logs the fact that the user reached a goal, also logging the cohort, so we can use that for stats later. it then marks the user with a different tag in the session cookie, to ensure that we don't double-count the results.

## Check results in CloudWatch insights

1. Select /aws/cloudfront/function/$STACK-NAME-assign-test-cohort and /aws/cloudfront/function/$STACK-NAME-assign-log-goal

```
parse @message '{"experiment":"*","cohort":"*","action":"*"' as experiment, cohort, action
| filter experiment='dec11reg'
| stats count() by cohort, action
| sort cohort, action
```

(note - the usual syntax with just referring to fields doesn't work as CF Functions always insert some garbage at the start of the log @message, so CW does not recognise it's JSON.

![](cw-results.png)

## Deploy

without A/B test

```
aws cloudformation deploy --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --template-file cloudformation/app.yml --stack-name cf-ab-test --region us-east-1
```

get stack resources

```
aws cloudformation describe-stacks --stack-name cf-ab-test --query "Stacks[].Outputs[]" --output table
```

send web files up

```
aws s3 sync website s3://$BUCKET --acl public-read
```

with A/B test

```
aws cloudformation deploy --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --template-file cloudformation/app.yml --stack-name cf-ab-test --region us-east-1 --parameter-overrides "ABTestExperimentName=dec11reg"
```

## Limitations of this approach

- requires cookies - but could be modified to use IP addresses or something else without cookies
- goal tracking is not perfectly unique (eg in case of two parallel requests to the goal URL, they would probably both be logged as success). this could be reduced by logging a different session cookie, with a client ID, and extracting from CW/saving to DynamoDB or some other source, that can prevent duplicates

## Extensions


### Exclude bots from A/B 

Use WAF Bot control to mark bots; this could reduce risk for SEO/search indexing on variants

### Avoiding cookies completely

Change to IP (challenge - provide balanced allocation - maybe use MD5 or some other hash on the IPs)

### Allow users to achieve a goal more than once

- remove the check for `-g` tag

### Automatically streaming results from CW to a different storage

- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/SubscriptionFilters.html#LambdaFunctionExample
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-logs-subscriptionfilter.html 

### Using CloudFront logs instead of a separate logging function

```
  LoggingBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      LifecycleConfiguration:
        Rules:
          - ExpirationInDays: 180
            Status: Enabled

  WebCFDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      ...
      Logging:
        Bucket: !GetAtt LoggingBucket.DomainName
        IncludeCookies: true
```

### Restricting tests 

- CloudFront-Viewer-Country-Name/CloudFront-Viewer-City/CloudFront-Viewer-Latitude/CloudFront-Viewer-Longitude
- CloudFront-Is-Desktop-Viewer/CloudFront-Is-Mobile-Viewer 

====

Questions to check with AWS

- can a origin-request function modify the response (eg set the cookie there? then we don't need the third function)
- is there some other header or CW synthetic request property that could be used to identify a client reliably (what's CloudFront-Viewer-JA3-Fingerprint )
- do we need to collect logs from multiple regions or just one region?
- does this need to be deployed in US-East-1 (Cloudfront?)
