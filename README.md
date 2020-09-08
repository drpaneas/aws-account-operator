# AWS Account Operator

## Table of Contents

- [AWS Account Operator](#aws-account-operator)
  - [Table of Contents](#table-of-contents)
  - [1. AWS Account Operator](#1-aws-account-operator)
    - [1.1. General Overview](#11-general-overview)
    - [1.2. Requirements](#12-requirements)
    - [1.3. Workflow](#13-workflow)
    - [1.4. Testing your AWS account credentials with the CLI](#14-testing-your-aws-account-credentials-with-the-cli)
    - [1.5. Development](#15-development)
    - [1.5.1 Operator Install](#151-operator-install)
      - [1.5.1.1 Local Mode](#1511-local-mode)
      - [1.5.1.2 Cluster Mode](#1512-cluster-mode)
  - [2. The Custom Resources](#2-the-custom-resources)
    - [2.1. AccountPool CR](#21-accountpool-cr)
    - [2.2. Account CR](#22-account-cr)
    - [2.3. AccountClaim CR](#23-accountclaim-cr)
    - [2.4. AWSFederatedRole CR](#24-awsfederatedrole-cr)
    - [2.4. AWSFederatedAccountAccess CR](#24-awsfederatedaccountaccess-cr)
  - [3. The controllers](#3-the-controllers)
    - [3.1. AccountPool Controller](#31-accountpool-controller)
      - [3.1.1. Constants and Globals](#311-constants-and-globals)
      - [3.1.2. Status](#312-status)
      - [3.1.3. Metrics](#313-metrics)
    - [3.2. Account Controller](#32-account-controller)
      - [3.2.1. Additional Functionality](#321-additional-functionality)
      - [3.2.2. Constants and Globals](#322-constants-and-globals)
      - [3.2.3. Spec](#323-spec)
      - [3.2.4. Status](#324-status)
      - [3.2.5. Metrics](#325-metrics)
    - [3.3. AccountClaim Controller](#33-accountclaim-controller)
      - [3.3.1. Constants and Globals](#331-constants-and-globals)
      - [3.3.2. Spec](#332-spec)
      - [3.3.3. Status](#333-status)
      - [3.3.4. Metrics](#334-metrics)
    - [3.4 AWSFederatedRole Controller](#34-awsfederatedrole-controller)
      - [3.4.1. Constants and Globals](#341-constants-and-globals)
      - [3.4.2. Spec](#342-spec)
      - [3.4.3. Status](#343-status)
      - [3.4.4. Metrics](#344-metrics)
    - [3.5 AWSFederatedAccountAccess Controller](#35-awsfederatedaccountaccess-controller)
      - [3.5.1. Constants and Globals](#351-constants-and-globals)
      - [3.5.2. Spec](#352-spec)
      - [3.5.3. Status](#353-status)
      - [3.4.4. Metrics](#344-metrics-1)
  - [4. Special Items in main.go](#4-special-items-in-maingo)
    - [4.1 Constants](#41-constants)

## 1. AWS Account Operator

### 1.1. General Overview

The operator is responsible for creating and maintaining a pool of AWS accounts and assigning accounts to AccountClaims. The operator creates the account in AWS, does initial setup and configuration of the those accounts, creates IAM resources and expose credentials for a IAM user with enough permissions to provision an OpenShift 4.x cluster.

The operator is deployed to an OpenShift cluster in the `aws-account-operator` namespace.

### 1.2. Requirements

The operator requires a secret named `aws-account-operator-credentials` in the `aws-account-operator` namespace, containing credentials to the AWS payer account you wish to create accounts in. The secret should contain credentials for an IAM user in the payer account with the data fields `aws_access_key_id` and `aws_secret_access_key`. The user should have the following IAM permissions:

Permissions to allow the user to assume the `OrganizationAccountAccessRole` role in any account created:

```json
{
   "Version": "2012-10-17",
   "Statement": {
       "Effect": "Allow",
       "Action": "sts:AssumeRole",
       "Resource": "arn:aws:iam::*:role/OrganizationAccountAccessRole"
   }
}

```

Permissions to allow the user to interact with the support center:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "support:*"
            ],
            "Resource": "*"
        }
    ]
}

```


The operator also needs a configmap that has:
Account Limit: The soft limit of AWS Accounts which is the number compared against when creating new accounts
Base: Base OU ID to place accounts in when claimed
Root: Root OU ID to create new OUs under

```json
{
    "apiVersion": "v1",
    "data": {
        "account-limit": "4801",
        "base": "ou-0wd6-tmsbvahq",
        "root": "r-0wd6"
    },
    "kind": "ConfigMap",
    "metadata": {
        "creationTimestamp": "2020-04-30T19:50:05Z",
        "name": "aws-account-operator-configmap",
        "namespace": "aws-account-operator",
    }
}
```
### 1.3. Workflow

First, an AccountPool must be created to specify the number of desired accounts to be ready. The operator then goes and creates that number of accounts.
When a [Hive](https://github.com/openshift/hive) cluster has a new cluster request, an AccountClaim is created with the name equal to the desired name of the cluster in a unique workspace. The operator links the AccountClaim to an Account in the pool, and creates the required k8s secrets, placing them in the AccountClaim's unique namespace. The AccountPool is then filled up again by the operator.  Hive then uses the secrets to create the AWS resources for the new cluster.

For more information on how this process is done, please refer to the controllers section.

### 1.4. Testing your AWS account credentials with the CLI

The below commands can be used to test payer account credentials where we create new accounts inside the payer accounts organization. Once the account is created in the first step we wait until the account is created with step 2 and retrieve its account ID. Using the account ID we can then test our IAM user has sts:AssumeRole permissions to Assume the OrganizationAccountAccessRole in the new account. The OrganizationAccountAccessRole is created automatically when a new account is created under the organization.

1. `aws organizations create-account --email "username+cli-test@redhat.com" --account-name "username-cli-test" --profile=orgtest`
2. `aws organizations list-accounts --profile=orgtest | jq '.[][] | select(.Name=="username-cli-test")'`
3. `aws sts assume-role --role-arn arn:aws:iam::<ID>:role/OrganizationAccountAccessRole --role-session-name username-cli-test --profile=orgtest`

### 1.5. Development

It is recommended to let the operator know when you're running it for testing purposes.
This has benefits such as skipping AWS support case creation.
This is done by setting `FORCE_DEV_MODE` in the operator's environment.
This is handled for you if you use one of the `make deploy-*` targets described below.

### 1.5.1 Operator Install

The operator can be installed into various cluster and pseudo-cluster environments. Depending which you choose, you can run in "local" mode or in "cluster" mode. The former is known to work in a Minishift or Code-Ready-Containers (CRC) cluster, or a private OpenShift cluster. The latter is known to work in a real OSD cluster. You can try to mix and match; it might work.

Both modes share predeployment steps. These can be done via `make predeploy`, which requires your access key credentials. You must be logged into the cluster as an administrator, or otherwise have permissions to create namespaces and deploy CRDs. For Minishift, this can be done:

```sh
oc login -u system:admin
OPERATOR_ACCESS_KEY_ID="YOUR_ACCESS_KEY_ID" OPERATOR_SECRET_ACCESS_KEY="YOUR_SECRET_ACCESS_KEY" make predeploy
```

This does the following:
- Ensures existence of the namespace in which the operator will run.
- Installs the [credentials described above](#12-requirements).
- Installs the operator's [Custom Resource Definitions](deploy/crds).
- Creates an initially zero-size [accountpool CR](hack/files/aws_v1alpha1_zero_size_accountpool.yaml).

Predeployment only needs to be done once, unless you are modifying the above artifacts.

#### 1.5.1.1 Local Mode

"Local" development mode differs from production in the following ways:
- AWS support case management is skipped. Your Accounts will get an artificial case number.
- Metrics are served from your local system at http://localhost:8080/metrics

On a local cluster, after [predeploying](#151-operator-install), running

```sh
make deploy-local
```

will invoke the `operator-sdk` executable in `local` mode with the `FORCE_DEV_MODE=local` environment variable.
(**Note:** This currently relies on `operator-sdk` at version [0.8.x](https://github.com/operator-framework/operator-sdk/blob/v0.8.x/doc/user/install-operator-sdk.md).
The syntax of the executable has changed over time, so this `make` target may not work with other versions.)

#### 1.5.1.2 Cluster Mode

In "cluster" development mode, as in local mode, AWS support case management is skipped.
However, metrics are served from within the cluster just as they are in a production deployment.

Once logged into the cluster, after [predeploying](#151-operator-install), running

```sh
make deploy-cluster
```

will do the following:
- Create the necessary service accounts, cluster roles, and cluster role bindings.
- Create the operator Deployment, including `FORCE_DEV_MODE=cluster` in the environment of the operator's container.

**Note:** `make deploy-cluster` will deploy the development image created by the `make build` target. As you iterate, you will need to `make build` and `make push` each time before you `make deploy-cluster`.

As with local mode, you must be logged into the cluster as an administrator, or otherwise have permissions to create namespaces and deploy CRDs.

## 2. The Custom Resources

### 2.1. AccountPool CR

The AccountPool CR holds the information about the available number of accounts that can be claimed for cluster provisioning.

```yaml
apiVersion: aws.managed.openshift.io/v1alpha1
kind: AccountPool
metadata:
  name: example-accountpool
  namespace: aws-account-operator
spec:
  poolSize: 50
```

### 2.2. Account CR

The Account CR holds the details about the AWS account that was created, where the account is in the process of becoming ready, and whether its linked to an AccountClaime, i.e. claimed.

```yaml
apiVersion: aws.managed.openshift.io/v1alpha1
kind: Account
metadata:
  name: osd-{accountName}
  namespace: aws-account-operator
spec:
  awsAccountID: "0000000000"
  claimLink: example-link
  iamUserSecret: osd-{accountName}-secret
```

### 2.3. AccountClaim CR

The AccountClaim CR links to an available account and stores the name of the associated secret with AWS credentials for that account.

```yaml
apiVersion: aws.managed.openshift.io/v1alpha1
kind: AccountClaim
metadata:
  name: example-link
  namespace: {NameSpace cluster is being built in}
spec:
  accountLink: osd-{accountName} (From AccountClaim)
  aws:
    regions:
    - name: us-east-1
  awsCredentialSecret:
    name: aws
    namespace: {NameSpace cluster is being built in}
  legalEntity:
    id: 00000000000000
    name: {Legal Entity Name}
```

### 2.4. AWSFederatedRole CR

The AWSFederatedRole CR contains a definition of a desired AWS Role, with both managed and custom Policies included

```yaml
apiVersion: aws.managed.openshift.io/v1alpha1
kind: AWSFederatedRole
metadata:
  name: example-role
  namespace: aws-account-operator
spec:
  roleDisplayName: Example Role
  roleDescription: This is an example Role
  # Custom Policy definition
  awsCustomPolicy:
    name:  ExampleCustomPolicy
    description: Description of Example Custom Policy
    # list of statements for the policy
    awsStatements:
      - effect: Allow
        action:
        - "aws-portal:ViewAccount"
        - "aws-portal:ViewBilling"
        resource: 
        - "*"
  # list of  AWS managed
  awsManagedPolicies:
   - "AWSAccountUsageReportAccess"
   - "AmazonEC2ReadOnlyAccess"
   - "AmazonS3ReadOnlyAccess"
   - "IAMReadOnlyAccess"
```

### 2.4. AWSFederatedAccountAccess CR

The AWSFederatedAccountAccess CR creates an instance of an AWSFederatedRole in AWS and allows the target IAM account to assume it

```yaml
apiVersion: aws.managed.openshift.io/v1alpha1
kind: AWSFederatedAccountAccess
metadata:
  name: example-account-access
  namespace: aws-account-operator
spec:
  awsCustomerCredentialSecret: 
    name: {Name for secret with osdManagedAdmin credentials} 
    namespace: {Namespace for the secret with osdManagedAdmin credentials}
  externalCustomerAWSIAMARN: arn:aws:iam::${EXTERNAL_AWS_ACCOUNT_ID}:user/${EXTERNAL_AWS_IAM_USER}
  awsFederatedRole:
    name: {Name of desired AWSFederatedRole}
    namespace: aws-account-operator  
```

## 3. The controllers

### 3.1. AccountPool Controller

The accountpool-controller is triggered by a create or change to an accountpool CR or an account CR. It is responsible for filling the Acccount Pool by generating new account CRs.

It looks at the accountpool CR *spec.poolSize* and it ensures that the number of unclaimed accounts matches the number of the poolsize. If the number of unclaimed accounts is less then the poolsize it creates a new account CR for the account-controller to process.

#### 3.1.1. Constants and Globals

```go
emailID = "osd-creds-mgmt"
```

#### 3.1.2. Status

Updates accountPool CR

```yaml
  claimedAccounts: 4
  poolSize: 3
  unclaimedAccounts: 3
```

*claimedAccounts* are any accounts with the `status.Claimed=true`

*unclaimedAccounts* are any accounts with `status.Claimed=false` and `status.State!="Failed"`.

*poolSize* is the poolsize from the accountPool spec

#### 3.1.3. Metrics

Updated in the accountPool-controller

```txt
MetricTotalAccountCRs
MetricTotalAccountCRsUnclaimed
MetricTotalAccountCRsClaimed
MetricTotalAccountPendingVerification
MetricTotalAccountCRsFailed
MetricTotalAccountCRsReady
MetricTotalAccountClaimCRs
```

### 3.2. Account Controller

The account-controller is triggered by creating or changing an account CR. It is responsible for following behaviors:

If the *awsLimit* set in the constants is not exceeded:

1. Creates a new account in the organization belonging to credentials in secret `aws-account-operator-credentials`
2. Configures two AWS IAM users from *iamUserNameUHC* and *iamUserNameSRE* as their respective usernames
    - Creates IAM user in new account
    - Attaches Admin policy
    - Generates a secret access key for the user
    - Stores user secret in a AWS secret
3. Creates STS CLI tokens
    - Creates Federated webconsole URL using the *iamUserNameSRE* user
4. Creates and Destroys EC2 instances
5. Creates aws support case to increase account limits

**Note:**
*iamUserNameUHC* is used by Hive to provision clusters
*iamUserNameSRE* is used to generate Federated console URL

#### 3.2.1. Additional Functionality

- If `status.RotateCredentials == true` the account-controller will refresh the STS Cli Credentials
- If the account's `status.State == "Creating"` and the account is older then the *createPendTime* constant the account will be put into a `failed` state
- If the account's `status.State == AccountReady && spec.ClaimLink != ""` it sets `status.Claimed = true`

#### 3.2.2. Constants and Globals

```go
awsLimit                = 2000
awsCredsUserName        = "aws_user_name"
awsCredsSecretIDKey     = "aws_access_key_id"
awsCredsSecretAccessKey = "aws_secret_access_key"
iamUserNameUHC          = "osdManagedAdmin"
iamUserNameSRE          = "osdManagedAdminSRE"
awsSecretName           = "aws-account-operator-credentials"
awsAMI                  = "ami-000db10762d0c4c05"
awsInstanceType         = "t2.micro"
createPendTime          = 10 * time.Minute

// Fields used to create/monitor AWS case
caseCategoryCode              = "other-account-issues"
caseServiceCode               = "customer-account"
caseIssueType                 = "customer-service"
caseSeverity                  = "high"
caseDesiredInstanceLimit      = 25
caseStatusResolved            = "resolved"
intervalAfterCaseCreationSecs = 30
intervalBetweenChecksSecs     = 30

// AccountPending indicates an account is pending
AccountPending = "Pending"
// AccountCreating indicates an account is being created
AccountCreating = "Creating"
// AccountFailed indicates account creation has failed
AccountFailed = "Failed"
// AccountReady indicates account creation is ready
AccountReady = "Ready"
// AccountPendingVerification indicates verification (of AWS limits and Enterprise Support) is pending
AccountPendingVerification = "PendingVerification"
// IAM Role name for IAM user creating resources in account
accountOperatorIAMRole = "OrganizationAccountAccessRole"

var awsAccountID string
var desiredInstanceType = "m5.xlarge"
var coveredRegions = []string{
    "us-east-1",
    "us-east-2",
    "us-west-1",
    "us-west-2",
    "ca-central-1",
    "eu-central-1",
    "eu-west-1",
    "eu-west-2",
    "eu-west-3",
    "ap-northeast-1",
    "ap-northeast-2",
    "ap-south-1",
    "ap-southeast-1",
    "ap-southeast-2",
    "sa-east-1",
}
```

#### 3.2.3. Spec

Updates the Account CR

```yaml
spec:
  awsAccountID: "000000112120"
  claimLink: "claim-name"
  iamUserSecret: accountName-secret
```

*awsAccountID* is updated with the account ID of the aws account that is created by the account controller

*claimLink* holds the name of the accountClaim that has claimed this account CR

*iamUserSecret* holds the name of the secret containing IAM user credentials for the AWS account

#### 3.2.4. Status

Updates the Account CR

```yaml
status:
  claimed: false
  conditions:
  - lastProbeTime: 2019-07-18T22:04:38Z
    lastTransitioNTime: 2019-07-18T22:04:38Z
    message: Attempting to create account
    reason: Creating
    status: "True"
    type: Creating
  rotateCredentials: false
  state: Failed
  supportCaseID: "00000000"
```

*state* can be any f the account states defined in the constants below:

- AccountPending indicates an account is pending
- AccountCreating indicates an account is being created
- AccountFailed indicates account creation has failed
- AccountReady indicates account creation is ready
- AccountPendingVerification indicates verification (of AWS limits and Enterprise Support) is pending

*claimed* is true if `currentAcctInstance.Status.State == AccountReady && currentAcctInstance.Spec.ClaimLink != "`
*rotateCredentials* updated by the secretwatcher pkg which will set the bool to true triggering an reconcile of this controller to rotate the STS credentials
*supportCaseID* is the ID of the aws support case to increase limits
*conditions* indicates the last state the account had and supporting details

#### 3.2.5. Metrics

Update in the account-controller

```txt
MetricTotalAWSAccounts
```

### 3.3. AccountClaim Controller

The accountClaim-controller is triggered when an accountClaim is created in any namespace. It is responsible for following behaviours:

1. Sets account `spec.ClaimLink` to the name of the accountClaim
2. Sets accountClaim `spec.AccountLink` to the name of an unclaimed Account
3. Creates a secret in the accountClaim namespace that contains the credentials tied to the aws account in the accountCR
4. Sets accountClaim `status.State = "Ready"`

#### 3.3.1. Constants and Globals

```go
AccountClaimed          = "AccountClaimed"
AccountUnclaimed        = "AccountUnclaimed"
awsCredsUserName        = "aws_user_name"
awsCredsAccessKeyID     = "aws_access_key_id"
awsCredsSecretAccessKey = "aws_secret_access_key"
```

#### 3.3.2. Spec

Updates the accountClaim CR

```yaml
spec:
  accountLink: osd-{accountName}
  aws:
    regions:
    - name: us-east-1
  awsCredentialSecret:
    name: aws
    namespace: {NameSpace}
  legalEntity:
    id: 00000000000000
    name: {Legal Entity Name}
```

*awsCredentialSecret* holds the name and namespace of the secret with the credentials created for the accountClaim

#### 3.3.3. Status

Updates the accountClaim CR

```yaml
status:
  conditions:
    - lastProbeTime: 2019-07-16T13:52:02Z
      lastTransitionTime: 2019-07-16T13:52:02Z
    message: Attempting to claim account
    reason: AccountClaimed
    status: "True"
    type: Unclaimed
    - lastProbeTime: 2019-07-16T13:52:03Z
      lastTransitionTime: 2019-07-16T13:52:03Z
    message: Account claimed by osd-creds-mgmt-fhq2d2
    reason: AccountClaimed
    status: "True"
    type: Claimed
    state: Ready
```

*state* can be any of the ClaimStatus strings defined in accountclaim_types.go
*conditions* indicates the last state the account had and supporting details

#### 3.3.4. Metrics

Updated in the accountClaim-controller

```txt
MetricTotalAccountClaimCRs
```
### 3.4 AWSFederatedRole Controller

The AWSFederatedRole-controller is triggered when an AWSFederatedRoke is created in any namespace. It is responsible for following behaviours:

1. Building AWS Policy Doc from Role definition in the spec
2. Attempting to validate the Role in AWS by creating the Role, and deleting it if successful
3. Setting the status to Valid or Failed
4. If the status is Valid or Failed, stop all reconciling
5. If an AWSFederatedRole is deleted, cleaning up any instance of the Role in AWS by cleaning up any AWSFederatedAccountAccesses using the AWSFederatedRole 

#### 3.4.1. Constants and Globals

None

#### 3.4.2. Spec

```yaml
spec:
  roleDisplayName: Example Role
  roleDescription: This is an example Role
  # Custom Policy definition
  awsCustomPolicy:
    name:  ExampleCustomPolicy
    description: Description of Example Custom Policy
    # list of statements for the policy
    awsStatements:
      - effect: Allow
        action:
        - "aws-portal:ViewAccount"
        - "aws-portal:ViewBilling"
        resource: 
        - "*"
  # list of  AWS managed
  awsManagedPolicies:
   - "AWSAccountUsageReportAccess"
   - "AmazonEC2ReadOnlyAccess"
   - "AmazonS3ReadOnlyAccess"
   - "IAMReadOnlyAccess"
```
*roleDisplayName* is a human-readable name for the Role
*roleDescription* is a human-readable description for what the Role does
*awsCustomPolicy* is a representation of an AWS Policy to be created as part of the Role. It contains a Policy name a description,
and a list of AWS Statements which Allow or Deny specific actions on specific resources. Please refer to the following documentation
for more information: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
*awsManagedPolicies* is a list of AWS pre-defined policies to add to the Role. Please refer to the following documentation for more information: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#aws-managed-policies


#### 3.4.3. Status
```yaml
  conditions:
  - lastProbeTime: {Time Stamp}
    lastTransitionTime: {Time Stamp}
    message: All managed and custom policies are validated
    reason: AllPoliciesValid
    status: "True"
    type: Valid
  state: Valid
```

*conditions* indicates the last states the AWSFederatedRole had and supporting details. In general for AWSFederatedRoles, only one condition is expected, and it should match the state.
*state* is the current state of the CR. Possible values are Valid and Failed

#### 3.4.4. Metrics

None

### 3.5 AWSFederatedAccountAccess Controller

The AWSFederatedAccountAccess-controller is triggered when an accountClaim is created in any namespace. It is responsible for following behaviours:

1. Ensures the requested AWSFederatedRole exists
2. Converts the AWSFederatedRole spec into an AWS Policy Doc
3. Creates a unique AWS Role in the AWS containing the OSD cluster using the AWSFederatedRole definition
4. Creates a unique AWS Policy if the AWSFederatedRole has awsCustomPolicy defined and attaches it to the Role
5. Attaches any specified AWS Managed Policies to the Role

#### 3.5.1. Constants and Globals

None

#### 3.5.2. Spec

```yaml
spec:
  awsCustomerCredentialSecret: 
    name: {Name for secret with osdManagedAdmin credentials} 
    namespace: {Namespace for the secret with osdManagedAdmin credentials}
  externalCustomerAWSIAMARN: arn:aws:iam::${EXTERNAL_AWS_ACCOUNT_ID}:user/${EXTERNAL_AWS_IAM_USER}
  awsFederatedRole:
    name: {Name of desired AWSFederatedRole}
    namespace: aws-account-operator  
```

*awsCustomerCredentialSecret* is the secret reference for the osdManagedAdmin IAM user in the AWS account where OSD is installed
*externalCustomerAWSIAMARN* is the AWS ARN for the desired IAM user that will use the AWS role when created. This should be in an AWS account external to the one where OSD is installed
*awsFederatedRole* is the reference to the target AWSFederatedRole CR to create an instance of 

#### 3.5.3. Status
 
```yaml
status:
  conditions:
  - lastProbeTime: {Time Stamp}
    lastTransitionTime: {Time Stamp}
    message: Account Access Ready
    reason: Ready
    status: "True"
    type: Ready
  consoleURL: https://signin.aws.amazon.com/switchrole?account=701718415138&roleName=network-mgmt-5dhkmd
  state: Ready
```

*conditions* indicates the states the AWSFederatedAccountAccess had and supporting details
*consoleURL* is a generated URL that directly allows the targeted IAM user to access the AWS Role
*state* is the current state of the CR

#### 3.4.4. Metrics

None

## 4. Special Items in main.go

- Starts a metric server with custom metrics defined in `localmetrics` pkg

### 4.1 Constants

```go
metricsPort               = "8080"
metricsPath               = "/metrics"
secretWatcherScanInterval = time.Duration(10) * time.Minute
```

*metricsPort* is the port used to start the metrics port

*metricsPath* it the path used as the metrics endpoint

*secretWatcherScanInterval* sets the interval at which the secret watcher will look for secrets that are expiring
