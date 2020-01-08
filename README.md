**Create a Kubernetes (EKS) Cluster.**

Create cluster.
`eksctl create cluster --name simple-jwt-api`
  
Check cluster progress in the [CloudFormation console](https://us-east-2.console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks?filteringText=&filteringStatus=active&viewNested=true&hideStacks=false).

Check the health of your cluster nodes.
`kubectl get nodes`

**Create an IAM role that CodeBuild can use to interact with EKS.**
Set an environment variable `ACCOUNT_ID` to the value of your AWS account id.
```
  ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Create a role policy document that allows the actions "eks:Describe*" and "ssm:GetParameters". You can do this by setting an environment variable with the role policy:

```
TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"
```
Create a role named 'UdacityFlaskDeployCBKubectlRole' using the role policy document (and attach all policies):

```
aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy

aws iam attach-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

aws iam attach-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-arn arn:aws:iam::aws:policy/AWSCodeBuildAdminAccess
```

You have now created a role named 'UdacityFlaskDeployCBKubectlRole'

**Grant the role access to the cluster. The 'aws-auth ConfigMap' is used to grant role based access control to your cluster.**

Get the current configmap and save it to a file:
```
kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml
```
This saves the config map to a temporary file located at ```/tmp/aws-auth-patch.yml```.

Check that file and make sure it’s all correct. Should look similar to the file below. Make sure your account id is filled out (don't copy/paste this example, your instance role node should not have the same name as mine does below, your node will have it's own unique name/id):
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn:  arn:aws:iam::{YOUR ACCOUNT ID HERE}:role/devel-worker-nodes-NodeInstanceRole-14W1I3VCZQHU7
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::{YOUR ACCOUNT ID HERE}:role/UdacityFlaskDeployCBKubectlRole
      username: build
      groups:
        - system:masters
```
Now update your cluster's configmap:

```
  kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```

OR

```
kubectl apply -f /tmp/aws-auth-patch.yaml
```

## Create the Pipeline

You will now create a pipeline which watches your Github. When changes are checked in, it will build a new image and deploy it to your cluster.

1.  Generate a GitHub access token. A Github acces token will allow CodePipeline to monitor when a repo is changed. A token can be generated  [here](https://github.com/settings/tokens/). You should generate the token with full control of private repositories, as shown in the image below. Be sure to save the token somewhere that is secure.

[](https://classroom.udacity.com/nanodegrees/nd0044/parts/df863fd7-e7ca-4b3a-9ec3-725b7a9cba70/modules/ac13842f-c841-4c1a-b284-b47899f4613d/lessons/becb2dac-c108-4143-8f6c-11b30413e28d/concepts/044acb09-a784-4af4-9383-a4b39c915915#)

![](https://video.udacity-data.com/topher/2019/September/5d6e8e1b_access-token/access-token.png)

2.  The file  _buildspec.yml_  instructs CodeBuild. We need a way to pass your jwt secret to the app in kubernetes securly. You will be using  [AWS Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)  to do this. First add the following to your buildspec.yml file:
    
    ```
    env:
      parameter-store:         
        JWT_SECRET: JWT_SECRET
    ```
    
    This lets CodeBuild know to set an evironment variable based on a value in the parameter-store.
    
3.  Put secret into AWS Parameter Store
    
    ```
    aws ssm put-parameter --name JWT_SECRET --value "YourJWTSecret" --type SecureString
    ```
    
4.  Modify CloudFormation template.
    
    There is file named  _ci-cd-codepipeline.cfn.yml_, this the the template file you will use to create your CodePipeline pipeline. Open this file and go to the 'Parameters' section. These are parameters that will accept values when you create a stack. Fill in the 'Default' value for the following:
    
    -   **EksClusterName**  : use the name of the EKS cluster you created above
    -   **GitSourceRepo**  : use the name of your project's github repo.
    -   **GitHubUser**  : use your github user name
    -   **KubectlRoleName**  : use the name of the role you created for kubectl above
    
    Save this file.
    
5.  Create a stack for CodePipeline
    
    -   Go the the  [CloudFormation service](https://us-east-2.console.aws.amazon.com/cloudformation/)  in the aws console.
    -   Press the 'Create Stack' button.
    -   Choose the 'Upload template to S3' option and upload the template file 'ci-cd-codepipeline.cfn.yml'
    -   Press 'Next'. Give the stack a name, fill in your GitHub login and the Github access token generated in step 1.
    -   Confirm the cluster name matches your cluster, the 'kubectl IAM role' matches the role you created above, and the repository matches the name of your forked repo.
    -   Create the stack.
        
        You can check it's status in the  [CloudFormation console](https://us-east-2.console.aws.amazon.com/cloudformation/).
        
6.  Check the pipeline works. Once the stack is successfully created, commit a change to the master branch of your github repo. Then, in the aws console go to the  [CodePipeline UI](https://us-east-2.console.aws.amazon.com/codesuite/codepipeline). You should see that the build is running.
    
7.  To test your api endpoints, get the external ip for your service:
    
    ```
    kubectl get services simple-jwt-api -o wide
    ```
    
    Now use the external ip url to test the app:
    
    ```
    export TOKEN=`curl -d '{"email":"<EMAIL>","password":"<PASSWORD>"}' -H "Content-Type: application/json" -X POST <EXTERNAL-IP URL>/auth  | jq -r '.token'`
    curl --request GET '<EXTERNAL-IP URL>/contents' -H "Authorization: Bearer ${TOKEN}" | jq 
    ```
    
8.  **Save the external IP from above to provide to the reviewer when you submit your project.**
