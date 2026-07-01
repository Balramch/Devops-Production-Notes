## Get hardcoded ami for upgrade 1.31 to 1.32
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.32/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --region eu-central-1 \
  --query "Parameter.Value" \
  --output text

  ## Best method

  spec:
  amiFamily: AL2023

  amiSelectorTerms:
    - alias: al2023@latest

  ## Another method

  aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.32/amazon-linux-2/recommended/image_id \
  --region eu-central-1 \
  --query "Parameter.Value" \
  --output text
