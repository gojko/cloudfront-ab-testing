# CloudFront A/B testing 

## Configuring A/B tests with CloudFormation

See [cloudformation/app.yml](cloudformation/app.yml)

- set ABTestExperimentName to a unique experiment name (so multiple tests can run in parallel without mixing results)
- optionally set ABTestURL, this is the web page under test (control). default = index.html
- optionally set ABTestVariantUrl, this is the variant of the test page. default = index2.html
- optionally set ABTestGoalUrl, this is the web page signaling success for the test (users that reach this page should be counted as funnel success). default = success.html

## Deploying

deploy without A/B test

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
aws cloudformation deploy --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND --template-file cloudformation/app.yml --stack-name cf-ab-test --region us-east-1 --parameter-overrides "ABTestExperimentName=dec11payments"
```



