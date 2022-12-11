# CloudFront A/B testing 

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



