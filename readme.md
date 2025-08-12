With Tableflow Catalog Integration, Tableflow acts as the catalog provider. Kafka topics are materialized as Iceberg tables stored in cloud storage (such as S3) and registered with Tableflow's built-in REST Catalog. Snowflake connects to this Tableflow REST Catalog as an external compute/query engine, using API credentials (API key and secret) generated within the Confluent Cloud Console. All metadata and table management remain under Tableflow’s control, and authentication is handled via Tableflow’s REST API keys—not Snowflake OAuth or native service connections. This approach is ideal when you want centralized catalog operations within Tableflow, while enabling Snowflake to query and analyze data without transferring catalog ownership.

Configure your Amazon S3 Bucket and Provider Integration

Once you have your Kafka topics created, you can now get started with Tableflow to sync your data from your Kafka topics into Snowflake Data Cloud.

But first, we need an Amazon S3 Bucket to write our Apache Iceberg data to. 

If you already have an Amazon S3 bucket, feel free to skip to the next section

In a new tab, navigate to the Amazon S3 console or click here

Click on Create Bucket on the top right.

Leave everything default and give your bucket a name. The name of the bucket must be unique so let’s name it tableflow-s3-bucket.

Leave the rest of the options as default, scroll down and click Create Bucket. 

Configure Storage Provider

Back on Confluent Cloud, navigate to the Tableflow page by clicking on Tableflow on the left-hand side of the menu bar within your Confluent Environment.

In the center of the page, you’ll see an option to configure a Storage Connection. Click on Go to Provider Integrations in order to configure the connection to Amazon S3.

From Provider Integrations, click on  + Add Integration. 

You can either create or use an existing role. For the sake of this example, we will create a new role. Click Continue.

On the Create Permission Policy in AWS page, select Tableflow S3 Bucket in the dropdown.

You will need to copy this permissions-policy.json for use in the next step

Create a new permissions policy on AWS

We need to create a permissions policy for Amazon S3 Access. In this section, we will create a permissions policy to be later used with an AWS Identity Access Management (IAM) Role.

In a new tab, open your Amazon Web Services (AWS) account and navigate to IAM Policies or click this link.

Click Create Policy

From the Create Policy screen, switch to the JSON editor

Copy the permissions-policy.json from the Confluent Provider Create Wizard into your policy editor on AWS. Ensure to replace the values for the Amazon S3 bucket we created above (tableflow-s3-bucket).

// Example structure from the document - ensure your bucket name is correct
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketLocation",
                "s3:ListBucketMultipartUploads",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::<<Your S3 Bucket Name>>" // Replace with your S3 bucket
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectTagging", // May differ slightly based on wizard version
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::<<Your S3 Bucket Name>>/*" // Replace with your S3 bucket
            ]
        }
    ]
}

Click Next

Give the policy the name tableflow-s3-access-policy and click Create Policy.

Click Create Policy.

Once the policy has been created, you can click Continue on Confluent Cloud.

The next page, Create role in AWS and map to Confluent will present a trust-policy.json that we will leverage for the next section of these instructions. Copy this down for later use.

Create role in AWS and map to Confluent

Next, we are going to create a role in AWS that will leverage the policy we just created, plus a trust-policy.json to allow Confluent to assume this role. 

Click here or navigate to the Roles sub-menu under the IAM service on AWS

Once on the Roles sub-menu, on the top right of the screen, click Create role in order to begin creating the role for Tableflow to assume.

For the trusted entity type, select AWS account.

Under An AWS account, select This account. In a later step, you modify the trust relationship and grant access to Snowflake.

Select the Require external ID option. Enter an external ID of your choice. For example, iceberg_table_external_id.

An external ID is used to grant access to your AWS resources (such as S3 buckets) to a third party like Snowflake.

Scroll down and click Next

In the next page, Add Permissions, search for the policy you created in the previous step, named tableflow-s3-access-policyand attach it to the role. Click Next.

Give the role a name, s3-tableflow-assume-role, scroll down and click Create Role

Once the role is created, you should see a green banner at the top of the console stating that the role was created. Click View Role to see the role details.

Copy the AWS ARN of the role and paste it in the Mapping component on the Provider Integration wizard on Confluent Cloud.

Give the Provider integration the name s3-provider-integration and click Continue.

You will now copy the updated Trust Policy from Confluent which contains the Confluent External ID and role. Copy the trust-policy.json from Confluent to your clipboard.

Go back to the AWS IAM Role you created (e.g., s3-tableflow-assume-role).

Select the Trust relationships tab.

Click Edit trust policy (or Edit trust relationship).

Replace the entire existing JSON with the updated trust-policy.json you copied from Confluent Cloud in the previous step. This adds the necessary External ID condition.

Click Update policy (or Save changes).

Return to the Confluent Cloud wizard one last time and click Continue (or Finish/Create).

Enable Tableflow on Your Kafka Topics

With the Provider Integration successfully configured, you can now enable Tableflow for your desired Kafka topics.

In your Confluent Cloud console, go to your Environment, then select your Kafka Cluster.

In the left-hand navigation menu for your cluster, click on Topics. You should see a list of your topics.

Find the specific topic you want to enable Tableflow for in the list.

On the right-hand side of the row for that topic, in the "Tableflow" column, click the Enable Tableflow button/link.

You will be asked to choose storage. Select Configure custom storage.

In the next menu, you will be able to choose the Provider Integration we created in the previous section. You can identify it by either the name of the provider integration or the IAM Role you created.

Provide the AWS S3 bucket name (tableflow-s3-bucket)

In the next screen, review the details and then click Launch.

You will now see Tableflow sync pending at the top of your topic information. This should transition to the Syncing status shortly.

Query Iceberg Tables via Snowflake

Now that we’ve got our Snowflake Open Catalog set up receiving data from our Kafka topics, we can now see the full end-to-end integration by referencing our Snowflake Open Catalog within our Snowflake environment.

Start by navigating back to your Snowflake environment. You can click this link in order to open your Snowsight UI.

Open or Create a new SQL Worksheet by clicking + Create at the top left hand corner, and select SQL Worksheet.

Create an External Volume

Ensure you have AccountAdmin Privileges granted to your account in order to create an external volume

We will now create an external volume referencing our Amazon S3 bucket. Enter the following SQL statement, replacing the Amazon S3 bucket with the one you created earlier (s3://tableflow-s3-bucket), the IAM role you created earlier (arn:aws:iam::<<account-id>>:role/s3-tableflow-assume-role), and leave the rest.

CREATE OR REPLACE EXTERNAL VOLUME iceberg_external_volumes
   STORAGE_LOCATIONS =
      (
         (
            NAME = 'my-iceberg-external-volume'
            STORAGE_PROVIDER = 'S3'
            STORAGE_BASE_URL = 's3://tableflow-s3-bucket'
            STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<<account-id>>:role/s3-tableflow-assume-role'
            STORAGE_AWS_EXTERNAL_ID = 'snowflake-external-id'
         )
      );

Execute this statement by highlighting the text and clicking the Play button  ▶️ to the top-right.

You will see the success message appear at the bottom of the page, showing that the external volume was successfully created.

We need to get the STORAGE_AWS_IAM_USER_ARN to provide to our IAM Role trust policy. We can do so by running a describe query on the external volume we just created.

DESC EXTERNAL VOLUME iceberg_external_volume;

Run this command like we did with the first command by highlighting the text and clicking on the Play button  ▶️to the top right.

The output of this command will appear at the bottom of the page. Under the results for STORAGE_LOCATIONS there will be a property_value, click this value to see the results on the right hand side.

From the value of the property_value column, copy the value of STORAGE_AWS_IAM_USER_ARN which should look like an IAM ARN

Update Trust Policy for Snowflake

Let’s now navigate back to our AWS IAM Role that we created previously (s3-tableflow-assume-role) and update the trust policy again:

Navigate back to your Role on AWS and switch to the Trust Relationships tab

Click Edit Trust Policy

We will again be adding another entry to our trust policy under the Statement block that will look like this:

{
			"Sid": "",
			"Effect": "Allow",
			"Principal": {
				"AWS": "<<STORAGE_AWS_IAM_USER_ARN>>"
			},
			"Action": "sts:AssumeRole",
			"Condition": {
				"StringEquals": {
					"sts:ExternalId": "iceberg_table_external_id"
				}
			}
  }

Under the AWS key, paste your STORAGE_AWS_IAM_USER_ARNwhich was copied from snowflake query (DESC EXTERNAL VOLUME iceberg_external_volume;)

Keep the external ID as iceberg_table_external_id as that’s the external ID we created in the external volume

Once you add the extra trust policy, your full trust policy should look like this:

{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				// from Confluent
				"AWS": "arn:aws:iam::<<account-id>>:role/<<role-id>>"
			},
			"Action": "sts:AssumeRole",
			"Condition": {
				"StringEquals": {
					// from Confluent
					"sts:ExternalId": "<<external-id>>"
				}
			}
		},
		{
			"Effect": "Allow",
			"Principal": {
				// from Confluent
				"AWS": "arn:aws:iam::<<account-id>>:role/<<role-id>>"
			},
			"Action": "sts:TagSession"
		},
{
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
		   // from Snowflake 
                "AWS": "arn:aws:iam::<<account-id>>:role/<<role-id>>"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
		   // from Snowflake
                    "sts:ExternalId": "snowflake-external-id"
                }
            }
        }
	]
}


The next step is verifying that we can actually access the external volume via Snowflake. Execute this command:

SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('iceberg_external_volume');

When this runs, you should see the keywords “success”:”true” and “PASSED” in the results set. This means the trust policy update took effect.

Create a Tableflow API Key

Click on the hamburger icon (three horizontal lines) in the top right of the screen.

Click API Keys in the menu under Administration.

Click Create Key in order to create your first API Key. If you have an existing API Key, click + Add Key to create another API Key.

Select My account and then click Next.

Select Tableflow, Click Next.

Give your API Key a name.

Enter a description for your API Key (e.g. API Key to Tableflow Integration).

Click Create API Key.

Click Download API key to save both the Key and Secret to your computer.

Click Complete.

Create Catalog Integration

We can now create our catalog integration, to link Snowflake to our Snowflake Open Data Catalog. For this SQL statement you will need to retrieve some information in the placeholders. Start by pasting in this placeholder create statement into your SQL worksheet.

CREATE OR REPLACE CATALOG INTEGRATION tableflow_rest_catalog_integration
    CATALOG_SOURCE=ICEBERG_REST
    TABLE_FORMAT=ICEBERG
    CATALOG_NAMESPACE='<cluster-id>'
    REST_CONFIG = (
        CATALOG_URI = 'https://tableflow.<region>.aws.confluent.cloud/iceberg/catalog/organizations/<org>/environments/<envID>'
        CATALOG_API_TYPE = PUBLIC
    )
    REST_AUTHENTICATION=(
        TYPE=OAUTH
        OAUTH_CLIENT_ID='<Tableflow-API-Key>'
        OAUTH_CLIENT_SECRET='<Tableflow-API-secret>'
        OAUTH_ALLOWED_SCOPES=('catalog')
    )
ENABLED=true;

Note: a. CATALOG_NAMESPACE should be your Kafka cluster ID. This can be found by navigating to your Confluent Cloud Kafka cluster and checking the cluster details on the right hand menu. Copy this value and paste it in the value for CATALOG_NAMESPACE.
b. The next placeholder, CATALOG_URI will be yourREST Catalog Endpoint(Found in Confluent Cloud Console → Your Environment → Your Cluster → Tableflow → "REST Catalog Endpoint" (copy this exact URL to use as your CATALOG_URI value))
c. For the OAUTH client ID and client secret, these are the Tableflow API key and secret values you created in the previous step when setting up API access in Confluent Cloud. Copy and paste these values as appropriate.

You are now ready to create your catalog integration. Highlight the command and click the Play button just as before. 

After running the command, you should see that the catalog integration was successfully created.

After running the command, you should see that the catalog integration was successfully created.

Creating the Iceberg Table and Querying

Once we have the catalog integration set up, you can now create an Iceberg table to query your snowflake open data catalog data.

Execute the following command, ensuring that all variables match exactly. If you named your external volume, catalog or kafka topic something different, this query will not work:

  CREATE OR REPLACE ICEBERG TABLE confluent_orders
  EXTERNAL_VOLUME = 'iceberg_external_volumes'
  CATALOG = 'tableflow_rest_catalog_integration1'
  CATALOG_TABLE_NAME = 'sample_orders';

Finally we are ready to query the table. Here are some queries you can try and evaluate the results.

These query results should update periodically when run again, showcasing that we are continuously writing data to our Iceberg tables.

References:

Query Iceberg Tables with Snowflake and Tableflow in Confluent Cloud
