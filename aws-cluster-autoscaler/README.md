### Requirements
- check the autoscaling groups tags created with the cluster to ensure that 
  - `k8s.io/cluster-autoscaler/enabled` is `true` and
  -  `k8s.io/cluster-autoscaler/<cluster-name>` is `owned` 

### Steps

1.  Create autoscaler policy <cluster-name>-cluster-autoscaler-policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": ["*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeImages",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": ["*"]
    }
  ]
}
```
2.  Create role of type Web Identity and name <cluster-name>-aws-autoscaler-role and attach the policy above
Note: use the OIDC url of the cluster


3.  Edit trust policy of the role above to allow access to service account

From this
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::389132740677:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXXXXXXXXXXXXX"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXXXXXXXXXXXXX:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```
To this
```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::389132740677:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXXXXXXXXXXXXX"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"oidc.eks.us-east-1.amazonaws.com/id/XXXXXXXXXXXXXXXXXXXXXXX:sub": "system:serviceaccount:kube-system:cluster-autoscaler"
				}
			}
		}
	]
}
```

4. Update the following in the autoscaler deployment 
 -   Go to aws-cluster-autoscaler/deploy.yaml and replace `eks.amazonaws.com/role-arn` with the arn of the role created above

```yaml
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::389132740677:role/rimsys-eks-preview-v2-cluster-autoscaler-role
```

 -   Goto the Deployment instance and replace with the version of your eks cluster, using the [latest tag](https://github.com/kubernetes/autoscaler/tags)

```yaml
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.27.5
```

 -   update the auto discovery with the correct tags, making sure the tags you confirmed in the requirements is specified
```yaml
    command:
      ...
      - --expander=least-waste
      - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/rimsys-eks-preview-v2
      - --balance-similar-node-groups
      ...
```

5. apply aws autoscaler script `kubectl apply -f aws-cluster-autoscaler/deploy.yaml `
