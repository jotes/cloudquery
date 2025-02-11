<p align="center">
<a href="https://cloudquery.io">
<img alt="cloudquery logo" width=75% src="https://github.com/cloudquery/cloudquery/raw/main/docs/images/logo.png" />
</a>
</p>

cloudquery transforms your cloud infrastructure into queryable SQL or Graphs for easy monitoring, governance and security.

### What is cloudquery and why use it?

cloudquery pulls, normalize, expose and monitor your cloud infrastructure and SaaS apps as SQL database.
This abstracts various scattered APIs enabling you to define security,governance,cost and compliance policies with SQL.

cloudquery can be easily extended to more resources and SaaS providers (open an [Issue](https://github.com/cloudquery/cloudquery/issues)). 

cloudquery comes with built-in policy packs such as: [AWS CIS](#running-policy-packs) (more is coming!).

Think about cloudquery as a compliance-as-code tool inspired by tools like [osquery](https://github.com/osquery/osquery)
and [terraform](https://github.com/hashicorp/terraform), cool right?

### Links
* Homepage: https://cloudquery.io
* Releases: https://github.com/cloudquery/cloudquery/releases
* Documentation: https://docs.cloudquery.io
* Schema explorer (schemaspy): https://schema.cloudquery.io/
* Database Configuration: https://docs.cloudquery.io/database-configuration

### Supported providers (Actively expanding)

Checkout https://hub.cloudquery.io

If you want us to add a new provider or resource please open an [Issue](https://github.com/cloudquery/cloudquery/issues).

## Download & install

You can download the precompiled binary from [releases](https://github.com/cloudquery/cloudquery/releases), or using CLI:

```shell script
export OS=Darwin # Possible values: Linux,Windows,Darwin
curl -L https://github.com/cloudquery/cloudquery/releases/latest/download/cloudquery_${OS}_x86_64 -o cloudquery
chmod a+x cloudquery
./cloudquery --help

# if you want to download a specific version and not latest use the following endpoint
export VERSION= # specifiy a version
curl -L https://github.com/cloudquery/cloudquery/releases/download/${VERSION}/cloudquery_${OS}_x86_64 -o cloudquery
```

Homebrew

```shell script
brew install cloudquery/tap/cloudquery
# After initial install you can upgrade the version via:
brew upgrade cloudquery
```

## Quick Start

### Running

First generate a `config.yml` file that will describe which resources you want cloudquery to pull, normalize
and transform resources to the specified SQL database by running the following command:
 
```shell script
cloudquery init aws # choose one or more from: [aws azure gcp okta]
# cloudquery init gcp azure # This will generate a config containing gcp and azure providers
# cloudquery init --help # Show all possible auto generated configs and flags
 ```

Once your `config.yml` is generated run the following command to fetch the resources:

```shell script
# you can spawn a local postgresql with docker
# docker run -p 5432:5432 -e POSTGRES_PASSWORD=pass -d postgres 
cloudquery fetch --dsn "host=localhost user=postgres password=pass DB.name=postgres port=5432"
# cloudquery fetch --help # Show all possible fetch flags
```

Using `psql -h localhost -p 5432 -U postgres -d postgres`

```shell script
postgres=# \dt
                                    List of relations
 Schema |                            Name                             | Type  |  Owner   
--------+-------------------------------------------------------------+-------+----------
 public | aws_autoscaling_launch_configuration_block_device_mapping   | table | postgres
 public | aws_autoscaling_launch_configurations                       | table | postgres
```

Run the following example queries from `psql` shell

List ec2_images
```sql
SELECT * FROM aws_ec2_images;
```

Find all public facing AWS load balancers
```sql
SELECT * FROM aws_elbv2_load_balancers WHERE scheme = 'internet-facing';
```

#### Running policy packs

cloudquery comes with some ready compliance policy pack which you can use as is or modify to fit your use-case.

Currently, cloudquery support [AWS CIS](https://d0.awsstatic.com/whitepapers/compliance/AWS_CIS_Foundations_Benchmark.pdf)
policy pack (it is under active development, so it doesn't cover the whole spec yet).

To run AWS CIS pack enter the following commands (make sure you fetched all the resources beforehand by the `fetch` command):

```shell script
./cloudquery policy --path=<PATH_TO_POLICY_FILE> --output=<PATH_TO_OUTPUT_POLICY_RESULT> --dsn "host=localhost user=postgres password=pass DB.name=postgres port=5432"
``` 

You can also create your own policy file. E.g.:

```yaml
views:
  - name: "my_custom_view"
    query: >
        CREATE VIEW my_custom_view AS ...
queries:
  - name: "Find thing that violates policy"
    query: >
        SELECT account_id, arn FROM ...
```

The `policy` command uses the policy file path `./policy.yml` by default, but this can be overridden via the `--path` flag, or the `CQ_POLICY_PATH` environment variable.

Full Documentation, resources and SQL schema definitions are available [here](https://docs.cloudquery.io)

### Providers Authentication

#### AWS 
You should be authenticated with an AWS account with correct permission with either option (see full [documentation](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/credentials.html)):
 * `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
 * `~/.aws/credentials` created via `aws configure`
 * `AWS_PROFILE`
 
Multi-account AWS support is available by using an account which can [AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) to other accounts.

In your config.hcl you need to specify role_arns if you want to query multiple accounts in the following way:
```hcl
accounts "<YOUR ACCOUNT ID>"{
 // Optional. Role ARN we want to assume when accessing this account
 role_arn = "<YOUR_ROLE_ARN>"
}
```
 
#### Azure

You should set the following environment variables:
`AZURE_CLIENT_ID`,`AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID` which you can generate via `az ad sp create-for-rbac --sdk-auth`.
See full details at [environment based authentication for sdk](https://docs.microsoft.com/en-us/azure/developer/go/azure-sdk-authorization#use-environment-based-authentication)

 
#### GCP

You should be authenticated with a GCP that has correct permissions for the data you want to pull.
You should set `GOOGLE_APPLICATION_CREDENTIALS` to point to your downloaded credential file.

#### Okta

You need to set `OKTA_TOKEN` environment variable

#### Query Examples

##### Find GCP buckets with public facing read permissions:

```sql
SELECT gcp_storage_buckets.name
FROM gcp_storage_buckets
         JOIN gcp_storage_bucket_policy_bindings ON gcp_storage_bucket_policy_bindings.bucket_id = gcp_storage_buckets.id
         JOIN gcp_storage_bucket_policy_binding_members ON gcp_storage_bucket_policy_binding_members.bucket_policy_binding_id = gcp_storage_bucket_policy_bindings.id
WHERE gcp_storage_bucket_policy_binding_members.name = 'allUsers' AND gcp_storage_bucket_policy_bindings.role = 'roles/storage.objectViewer';
```

##### Find all public facing AWS load balancers

```sql
SELECT * FROM aws_elbv2_load_balancers WHERE scheme = 'internet-facing';
```

##### Find all unencrypted RDS instances

```sql
SELECT * from aws_rds_clusters where storage_encrypted = 0;
```

##### Find all unencrypted AWS buckets

```sql
SELECT * from aws_s3_buckets
    JOIN aws_s3_bucket_encryption_rules ON aws_s3_buckets.id != aws_s3_bucket_encryption_rules.bucket_id;
```

More examples are available [here](https://docs.cloudquery.io)

## Compile and run

```
go build .
./cloudquery # --help to see all options
```

## Running on AWS (Lambda, Terraform)
You can use the `Makefile` to build, deploy, and destroy the entire tf infrastructure.
The default execution configuration file can be found on: `./deploy/aws/terraform/tasks/us-east-1`

You can define more tasks by adding cloudwatch periodic events.

#### For example:
The default configuration will execute the cloudquery every one day with the default configuration.
```terraform
resource "aws_cloudwatch_event_rule" "scan_schedule" {
  name = "Cloudquery-us-east-1-scan"
  description = "Run cloudquery everyday on us-east-1 resources"

  schedule_expression = "rate(1 day)"
}

resource "aws_cloudwatch_event_target" "sns" {
  rule      = aws_cloudwatch_event_rule.scan_schedule.name
  arn       = aws_lambda_function.cloudquery.arn
  input     = file("tasks/us-east-1/input.json")
}

```

Install terraform if not exists:
`https://learn.hashicorp.com/tutorials/terraform/install-cli`

Build cloudquery binary
```
make build
```

Deploy to aws using terraform
```
make apply
```

You can also use `init`, `plan` and `destroy` 

## License

By contributing to cloudquery you agree that your contributions will be licensed as defined on the LICENSE file.

## Contribution

Feel free to open Pull-Request for small fixes and changes. For bigger changes and new providers please open an issue first to prevent double work and discuss relevant stuff.
