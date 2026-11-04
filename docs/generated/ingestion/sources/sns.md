


# Amazon SNS

## Overview

[Amazon SNS](https://aws.amazon.com/sns/) is AWS's managed pub/sub service. Publishers send messages to a topic and SNS delivers a copy to each subscriber: SQS queues, Lambda functions, Firehose streams, HTTP endpoints, email or SMS.

The `sns` source plugin catalogs the topics of one AWS region as datasets, grouped in a region container, with topic settings and subscription counts as properties, AWS resource tags as DataHub tags, lineage from each topic to its subscribed SQS queues and subscription dead-letter queues, optional tags or glossary terms from message data protection policies, and stateful deletion. Its companion, the [`sqs` source](sqs.md), catalogs the queues; both build URNs the same way so their lineage joins.

## Concept Mapping

| Source Concept                   | DataHub Concept                                                                                     | Notes                                                         |
| -------------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| AWS account                      | [Platform Instance](../../../platform-instances.md)                                                 | Defaults to the account id read from a topic ARN              |
| AWS region                       | [Container](../../metamodel/entities/container.md)                                                  | Subtype `Region`                                              |
| Topic                            | [Dataset](../../metamodel/entities/dataset.md)                                                      | Platform `sns`, subtype `Topic`; name `<region>.<topic name>` |
| SQS subscription, redrive policy | Upstream lineage (patch) on the queue                                                               | Topic to queue, topic to subscription dead-letter queue       |
| Resource tag                     | [Tag](../../metamodel/entities/tag.md)                                                              | `Key:Value`, or `Key` for an empty value                      |
| Data protection identifier       | [Tag](../../metamodel/entities/tag.md) or [Glossary Term](../../metamodel/entities/glossaryTerm.md) | Opt-in; terms only for identifiers you map                    |


## Module `sns`
![Alpha](https://img.shields.io/badge/support%20status-Alpha-lightgrey)


### Important Capabilities
| Capability | Status | Notes |
| ---------- | ------ | ----- |
| Asset Containers | ✅ | Region containers. |
| Descriptions | ❌ | SNS has no description. |
| [Detect Deleted Entities](../../../../metadata-ingestion/docs/dev_guides/stateful.md#stale-entity-removal) | ✅ | Via stateful ingestion. |
| [Domains](../../../domains.md) | ✅ | Supported via the `domain` config field. |
| Extract Tags | ✅ | From AWS resource tags, added without removing UI tags. |
| Glossary Terms | ✅ | Opt-in, from data protection policy identifiers you map to existing terms. |
| [Platform Instance](../../../platform-instances.md) | ✅ | Defaults to the AWS account id. |
| Schema Metadata | ❌ | SNS has no schema. |
| Table-Level Lineage | ✅ | Topic to subscribed queue and to subscription dead-letter queue. |
| Test Connection | ✅ | Enabled by default. |

### Overview

The `sns` module reads the SNS control-plane APIs for one AWS region per recipe. It never publishes or reads messages.

SNS has no schema, so topics carry no schema fields. Topic settings (display name, FIFO, KMS key, archive policy) and subscription counts per protocol are stored as custom properties.

SQS queues are ingested by the separate `sqs` source. Topic-to-queue lineage is written by this source onto the queue URNs, so run both with the same `env`, `platform_instance` and `account_platform_instance_map`, or the edges will point at queue URNs the `sqs` source doesn't emit.

### Prerequisites

#### AWS IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SnsRead",
      "Effect": "Allow",
      "Action": [
        "sns:ListTopics",
        "sns:GetTopicAttributes",
        "sns:ListTagsForResource",
        "sns:ListSubscriptionsByTopic",
        "sns:GetSubscriptionAttributes",
        "sns:GetDataProtectionPolicy"
      ],
      "Resource": "*"
    },
    {
      "Sid": "QueueExistence",
      "Effect": "Allow",
      "Action": ["sqs:ListQueues"],
      "Resource": "*"
    },
    {
      "Sid": "TagsRead",
      "Effect": "Allow",
      "Action": ["tag:GetResources"],
      "Resource": "*"
    }
  ]
}
```

- `sqs:ListQueues` tells the connector which queues in its own account and region exist, so a subscription to a deleted queue doesn't create an empty queue entity. Without it, edges to local queues are skipped with a warning.
- Tags are read with one `tag:GetResources` call. If it is denied the connector warns and calls `sns:ListTagsForResource` per topic, which SNS limits to 10 calls per second.
- Subscriptions are listed per topic with `sns:ListSubscriptionsByTopic`, because `sns:ListSubscriptions` only returns the caller's own subscriptions and would miss ones other accounts created on your topics. That is one call per topic, and SNS allows 30 per second.
- `sns:GetSubscriptionAttributes` is only called when `include_subscription_attributes` is on, and `sns:GetDataProtectionPolicy` only when `data_protection.enabled` is on.

#### Authentication

Credentials come from `aws_config` or the standard boto3 chain: environment variables, a shared credentials profile, an IAM role on the host, or SSO. `aws_config.aws_role` assumes a role, including one in another account.


### Install the Plugin
```shell
pip install 'acryl-datahub[sns]'
```

### Starter Recipe
Check out the following recipe to get started with ingestion! See [below](#config-details) for full configuration options.


For general pointers on writing and running a recipe, see our [main recipe guide](../../../../metadata-ingestion/README.md#recipes).
```yaml
source:
  type: sns
  config:
    aws_config:
      aws_region: "us-east-1"
      # aws_role: "arn:aws:iam::123456789012:role/datahub-read"
    # platform_instance defaults to the AWS account id. Use the same env and
    # platform_instance as the sqs recipe so topic-to-queue lineage joins.
    env: "PROD"

    topic_pattern:
      allow:
        - ".*"

    extract_tags: true
    # One sns:GetSubscriptionAttributes call per subscription; reads subscription DLQs.
    include_subscription_attributes: false

    data_protection:
      enabled: false
      # glossary_term_map:
      #   EmailAddress: "urn:li:glossaryTerm:pii.email"

    # account_platform_instance_map:
    #   "210987654321": "shared-services"

    stateful_ingestion:
      enabled: true

```

### Config Details

                
#### Options


Note that a `.` is used to denote nested fields in the YAML recipe.


<div className='config-table'>

| Field | Description |
|:--- |:--- |
| <div className="path-line"><span className="path-main">account_platform_instance_map</span></div> <div className="type-name-line"><span className="type-name">map(str,string)</span></div> |   |
| <div className="path-line"><span className="path-main">extract_tags</span></div> <div className="type-name-line"><span className="type-name">boolean</span></div> | Extract AWS resource tags as DataHub tags. SNS allows 10 ListTagsForResource calls per second, so turn this off on the sns source for accounts with tens of thousands of topics. <div className="default-line default-line-with-docs">Default: <span className="default-value">True</span></div> |
| <div className="path-line"><span className="path-main">include_subscription_attributes</span></div> <div className="type-name-line"><span className="type-name">boolean</span></div> | Call sns:GetSubscriptionAttributes for each subscription to read its dead-letter queue. One call per subscription, for every protocol, on every run; subscription-to-queue edges need no extra calls. <div className="default-line default-line-with-docs">Default: <span className="default-value">False</span></div> |
| <div className="path-line"><span className="path-main">platform_instance</span></div> <div className="type-name-line"><span className="type-name">One of string, null</span></div> | The instance of the platform that all assets produced by this recipe belong to. This should be unique within the platform. See https://docs.datahub.com/docs/platform-instances/ for more details. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-main">remove_stale_lineage</span></div> <div className="type-name-line"><span className="type-name">boolean</span></div> | Remove lineage edges this source wrote on an earlier run that no longer exist in AWS, such as a removed redrive policy or subscription. Only edges from resources this run read in full are removed. Needs a DataHub graph connection (the datahub-rest sink or datahub_api); costs one DataHub lookup per queue or topic. <div className="default-line default-line-with-docs">Default: <span className="default-value">True</span></div> |
| <div className="path-line"><span className="path-main">env</span></div> <div className="type-name-line"><span className="type-name">string</span></div> | The environment that all assets produced by this connector belong to <div className="default-line default-line-with-docs">Default: <span className="default-value">PROD</span></div> |
| <div className="path-line"><span className="path-main">aws_config</span></div> <div className="type-name-line"><span className="type-name">AwsConnectionConfig</span></div> | Common AWS credentials config. <br />  <br /> Currently used by: <br />     - Glue source <br />     - SageMaker source <br />     - dbt source  |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_access_key_id</span></div> <div className="type-name-line"><span className="type-name">One of string, null</span></div> | AWS access key ID. Can be auto-detected, see [the AWS boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) for details. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_advanced_config</span></div> <div className="type-name-line"><span className="type-name">object</span></div> | Advanced AWS configuration options. These are passed directly to [botocore.config.Config](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html).  |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_endpoint_url</span></div> <div className="type-name-line"><span className="type-name">One of string, null</span></div> | The AWS service endpoint. This is normally [constructed automatically](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html), but can be overridden here. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_profile</span></div> <div className="type-name-line"><span className="type-name">One of string, null</span></div> | The [named profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) to use from AWS credentials. Falls back to default profile if not specified and no access keys provided. Profiles are configured in ~/.aws/credentials or ~/.aws/config. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_proxy</span></div> <div className="type-name-line"><span className="type-name">One of string, null</span></div> | A set of proxy configs to use with AWS. See the [botocore.config](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html) docs for details. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_region</span></div> <div className="type-name-line"><span className="type-name">One of string, null</span></div> | AWS region code. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_retry_mode</span></div> <div className="type-name-line"><span className="type-name">Enum</span></div> | One of: "legacy", "standard", "adaptive" <div className="default-line default-line-with-docs">Default: <span className="default-value">standard</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_retry_num</span></div> <div className="type-name-line"><span className="type-name">integer</span></div> | Number of times to retry failed AWS requests. See the [botocore.retry](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/retries.html) docs for details. <div className="default-line default-line-with-docs">Default: <span className="default-value">5</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_secret_access_key</span></div> <div className="type-name-line"><span className="type-name">One of string(password), null</span></div> | AWS secret access key. Can be auto-detected, see [the AWS boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) for details. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_session_token</span></div> <div className="type-name-line"><span className="type-name">One of string(password), null</span></div> | AWS session token. Can be auto-detected, see [the AWS boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) for details. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">read_timeout</span></div> <div className="type-name-line"><span className="type-name">number</span></div> | The timeout for reading from the connection (in seconds). <div className="default-line default-line-with-docs">Default: <span className="default-value">60</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.</span><span className="path-main">aws_role</span></div> <div className="type-name-line"><span className="type-name">One of string, array, null</span></div> | AWS roles to assume. If using the string format, the role ARN can be specified directly. If using the object format, the role can be specified in the RoleArn field and additional available arguments are the same as [boto3's STS.Client.assume_role](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sts.html?highlight=assume_role#STS.Client.assume_role). <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">aws_config.aws_role.</span><span className="path-main">union</span></div> <div className="type-name-line"><span className="type-name">One of string, AwsAssumeRoleConfig</span></div> |   |
| <div className="path-line"><span className="path-prefix">aws_config.aws_role.union.</span><span className="path-main">RoleArn</span>&nbsp;<abbr title="Required if union is set">❓</abbr></div> <div className="type-name-line"><span className="type-name">string</span></div> | ARN of the role to assume.  |
| <div className="path-line"><span className="path-prefix">aws_config.aws_role.union.</span><span className="path-main">ExternalId</span></div> <div className="type-name-line"><span className="type-name">One of string, null</span></div> | External ID to use when assuming the role. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-main">data_protection</span></div> <div className="type-name-line"><span className="type-name">SnsDataProtectionConfig</span></div> |   |
| <div className="path-line"><span className="path-prefix">data_protection.</span><span className="path-main">enabled</span></div> <div className="type-name-line"><span className="type-name">boolean</span></div> | Read each topic's message data protection policy (sns:GetDataProtectionPolicy, one call per topic) and tag the topic with the data identifiers it covers. <div className="default-line default-line-with-docs">Default: <span className="default-value">False</span></div> |
| <div className="path-line"><span className="path-prefix">data_protection.</span><span className="path-main">glossary_term_map</span></div> <div className="type-name-line"><span className="type-name">map(str,string)</span></div> |   |
| <div className="path-line"><span className="path-prefix">data_protection.</span><span className="path-main">tag_prefix</span></div> <div className="type-name-line"><span className="type-name">string</span></div> | Tags are named `<tag_prefix>:<identifier>`, e.g. `aws_data_identifier:EmailAddress`. <div className="default-line default-line-with-docs">Default: <span className="default-value">aws&#95;data&#95;identifier</span></div> |
| <div className="path-line"><span className="path-main">domain</span></div> <div className="type-name-line"><span className="type-name">map(str,AllowDenyPattern)</span></div> | A class to store allow deny regexes. <br />  <br /> Patterns are matched against the start of the string only, not the entire <br /> string - a pattern does not need to match to the end to be considered a match. <br /> For example, the pattern "prod" matches "prod", "prod_east", and "production". <br /> To require an exact match, anchor your pattern explicitly, e.g. "^prod$".  |
| <div className="path-line"><span className="path-prefix">domain.`key`.</span><span className="path-main">allow</span></div> <div className="type-name-line"><span className="type-name">array</span></div> | List of regex patterns to include in ingestion. Patterns match from the start of the string only, not the entire string - anchor with '^...$' for an exact match, e.g. '^prod$'. <div className="default-line default-line-with-docs">Default: <span className="default-value">&#91;&#x27;.&#42;&#x27;&#93;</span></div> |
| <div className="path-line"><span className="path-prefix">domain.`key`.allow.</span><span className="path-main">string</span></div> <div className="type-name-line"><span className="type-name">string</span></div> |   |
| <div className="path-line"><span className="path-prefix">domain.`key`.</span><span className="path-main">ignoreCase</span></div> <div className="type-name-line"><span className="type-name">One of boolean, null</span></div> | Whether to ignore case sensitivity during pattern matching. <div className="default-line default-line-with-docs">Default: <span className="default-value">True</span></div> |
| <div className="path-line"><span className="path-prefix">domain.`key`.</span><span className="path-main">deny</span></div> <div className="type-name-line"><span className="type-name">array</span></div> | List of regex patterns to exclude from ingestion. Patterns match from the start of the string only, not the entire string - anchor with '^...$' for an exact match, e.g. '^prod$'. <div className="default-line default-line-with-docs">Default: <span className="default-value">&#91;&#93;</span></div> |
| <div className="path-line"><span className="path-prefix">domain.`key`.deny.</span><span className="path-main">string</span></div> <div className="type-name-line"><span className="type-name">string</span></div> |   |
| <div className="path-line"><span className="path-main">topic_pattern</span></div> <div className="type-name-line"><span className="type-name">AllowDenyPattern</span></div> | A class to store allow deny regexes. <br />  <br /> Patterns are matched against the start of the string only, not the entire <br /> string - a pattern does not need to match to the end to be considered a match. <br /> For example, the pattern "prod" matches "prod", "prod_east", and "production". <br /> To require an exact match, anchor your pattern explicitly, e.g. "^prod$".  |
| <div className="path-line"><span className="path-prefix">topic_pattern.</span><span className="path-main">ignoreCase</span></div> <div className="type-name-line"><span className="type-name">One of boolean, null</span></div> | Whether to ignore case sensitivity during pattern matching. <div className="default-line default-line-with-docs">Default: <span className="default-value">True</span></div> |
| <div className="path-line"><span className="path-main">stateful_ingestion</span></div> <div className="type-name-line"><span className="type-name">One of StatefulStaleMetadataRemovalConfig, null</span></div> | Stateful ingestion configuration for stale entity removal. <div className="default-line default-line-with-docs">Default: <span className="default-value">None</span></div> |
| <div className="path-line"><span className="path-prefix">stateful_ingestion.</span><span className="path-main">enabled</span></div> <div className="type-name-line"><span className="type-name">boolean</span></div> | Whether or not to enable stateful ingest. Default: True if a pipeline_name is set and either a datahub-rest sink or `datahub_api` is specified, otherwise False <div className="default-line default-line-with-docs">Default: <span className="default-value">False</span></div> |
| <div className="path-line"><span className="path-prefix">stateful_ingestion.</span><span className="path-main">fail_safe_threshold</span></div> <div className="type-name-line"><span className="type-name">number</span></div> | Prevents large amount of soft deletes & the state from committing from accidental changes to the source configuration if the relative change percent in entities compared to the previous state is above the 'fail_safe_threshold'. <div className="default-line default-line-with-docs">Default: <span className="default-value">75.0</span></div> |
| <div className="path-line"><span className="path-prefix">stateful_ingestion.</span><span className="path-main">remove_stale_metadata</span></div> <div className="type-name-line"><span className="type-name">boolean</span></div> | Soft-deletes the entities present in the last successful run but missing in the current run with stateful_ingestion enabled. <div className="default-line default-line-with-docs">Default: <span className="default-value">True</span></div> |

</div>




#### Schema


The [JSONSchema](https://json-schema.org/) for this configuration is inlined below.


```javascript
{
  "$defs": {
    "AllowDenyPattern": {
      "additionalProperties": false,
      "description": "A class to store allow deny regexes.\n\nPatterns are matched against the start of the string only, not the entire\nstring - a pattern does not need to match to the end to be considered a match.\nFor example, the pattern \"prod\" matches \"prod\", \"prod_east\", and \"production\".\nTo require an exact match, anchor your pattern explicitly, e.g. \"^prod$\".",
      "properties": {
        "allow": {
          "default": [
            ".*"
          ],
          "description": "List of regex patterns to include in ingestion. Patterns match from the start of the string only, not the entire string - anchor with '^...$' for an exact match, e.g. '^prod$'.",
          "items": {
            "type": "string"
          },
          "title": "Allow",
          "type": "array"
        },
        "deny": {
          "default": [],
          "description": "List of regex patterns to exclude from ingestion. Patterns match from the start of the string only, not the entire string - anchor with '^...$' for an exact match, e.g. '^prod$'.",
          "items": {
            "type": "string"
          },
          "title": "Deny",
          "type": "array"
        },
        "ignoreCase": {
          "anyOf": [
            {
              "type": "boolean"
            },
            {
              "type": "null"
            }
          ],
          "default": true,
          "description": "Whether to ignore case sensitivity during pattern matching.",
          "title": "Ignorecase"
        }
      },
      "title": "AllowDenyPattern",
      "type": "object"
    },
    "AwsAssumeRoleConfig": {
      "additionalProperties": true,
      "properties": {
        "RoleArn": {
          "description": "ARN of the role to assume.",
          "title": "Rolearn",
          "type": "string"
        },
        "ExternalId": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "External ID to use when assuming the role.",
          "title": "Externalid"
        }
      },
      "required": [
        "RoleArn"
      ],
      "title": "AwsAssumeRoleConfig",
      "type": "object"
    },
    "AwsConnectionConfig": {
      "additionalProperties": false,
      "description": "Common AWS credentials config.\n\nCurrently used by:\n    - Glue source\n    - SageMaker source\n    - dbt source",
      "properties": {
        "aws_access_key_id": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "AWS access key ID. Can be auto-detected, see [the AWS boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) for details.",
          "title": "Aws Access Key Id"
        },
        "aws_secret_access_key": {
          "anyOf": [
            {
              "format": "password",
              "type": "string",
              "writeOnly": true
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "AWS secret access key. Can be auto-detected, see [the AWS boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) for details.",
          "title": "Aws Secret Access Key"
        },
        "aws_session_token": {
          "anyOf": [
            {
              "format": "password",
              "type": "string",
              "writeOnly": true
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "AWS session token. Can be auto-detected, see [the AWS boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html) for details.",
          "title": "Aws Session Token"
        },
        "aws_role": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "items": {
                "anyOf": [
                  {
                    "type": "string"
                  },
                  {
                    "$ref": "#/$defs/AwsAssumeRoleConfig"
                  }
                ]
              },
              "type": "array"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "AWS roles to assume. If using the string format, the role ARN can be specified directly. If using the object format, the role can be specified in the RoleArn field and additional available arguments are the same as [boto3's STS.Client.assume_role](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sts.html?highlight=assume_role#STS.Client.assume_role).",
          "title": "Aws Role"
        },
        "aws_profile": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The [named profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) to use from AWS credentials. Falls back to default profile if not specified and no access keys provided. Profiles are configured in ~/.aws/credentials or ~/.aws/config.",
          "title": "Aws Profile"
        },
        "aws_region": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "AWS region code.",
          "title": "Aws Region"
        },
        "aws_endpoint_url": {
          "anyOf": [
            {
              "type": "string"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "The AWS service endpoint. This is normally [constructed automatically](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html), but can be overridden here.",
          "title": "Aws Endpoint Url"
        },
        "aws_proxy": {
          "anyOf": [
            {
              "additionalProperties": {
                "type": "string"
              },
              "type": "object"
            },
            {
              "type": "null"
            }
          ],
          "default": null,
          "description": "A set of proxy configs to use with AWS. See the [botocore.config](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html) docs for details.",
          "title": "Aws Proxy"
        },
        "aws_retry_num": {
          "default": 5,
          "description": "Number of times to retry failed AWS requests. See the [botocore.retry](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/retries.html) docs for details.",
          "title": "Aws Retry Num",
          "type": "integer"
        },
        "aws_retry_mode": {
          "default": "standard",
          "description": "Retry mode to use for failed AWS requests. See the [botocore.retry](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/retries.html) docs for details.",
          "enum": [
            "legacy",
            "standard",
            "adaptive"
          ],
          "title": "Aws Retry Mode",
          "type": "string"
        },
        "read_timeout": {
          "default": 60,
          "description": "The timeout for reading from the connection (in seconds).",
          "title": "Read Timeout",
          "type": "number"
        },
        "aws_advanced_config": {
          "additionalProperties": true,
          "description": "Advanced AWS configuration options. These are passed directly to [botocore.config.Config](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html).",
          "title": "Aws Advanced Config",
          "type": "object"
        }
      },
      "title": "AwsConnectionConfig",
      "type": "object"
    },
    "SnsDataProtectionConfig": {
      "additionalProperties": false,
      "properties": {
        "enabled": {
          "default": false,
          "description": "Read each topic's message data protection policy (sns:GetDataProtectionPolicy, one call per topic) and tag the topic with the data identifiers it covers.",
          "title": "Enabled",
          "type": "boolean"
        },
        "tag_prefix": {
          "default": "aws_data_identifier",
          "description": "Tags are named `<tag_prefix>:<identifier>`, e.g. `aws_data_identifier:EmailAddress`.",
          "title": "Tag Prefix",
          "type": "string"
        },
        "glossary_term_map": {
          "additionalProperties": {
            "type": "string"
          },
          "description": "Data identifier name (e.g. `EmailAddress`) to an existing glossary term urn or name. Only listed identifiers get a term; none are created.",
          "title": "Glossary Term Map",
          "type": "object"
        }
      },
      "title": "SnsDataProtectionConfig",
      "type": "object"
    },
    "StatefulStaleMetadataRemovalConfig": {
      "additionalProperties": false,
      "description": "Base specialized config for Stateful Ingestion with stale metadata removal capability.",
      "properties": {
        "enabled": {
          "default": false,
          "description": "Whether or not to enable stateful ingest. Default: True if a pipeline_name is set and either a datahub-rest sink or `datahub_api` is specified, otherwise False",
          "title": "Enabled",
          "type": "boolean"
        },
        "remove_stale_metadata": {
          "default": true,
          "description": "Soft-deletes the entities present in the last successful run but missing in the current run with stateful_ingestion enabled.",
          "title": "Remove Stale Metadata",
          "type": "boolean"
        },
        "fail_safe_threshold": {
          "default": 75.0,
          "description": "Prevents large amount of soft deletes & the state from committing from accidental changes to the source configuration if the relative change percent in entities compared to the previous state is above the 'fail_safe_threshold'.",
          "maximum": 100.0,
          "minimum": 0.0,
          "title": "Fail Safe Threshold",
          "type": "number"
        }
      },
      "title": "StatefulStaleMetadataRemovalConfig",
      "type": "object"
    }
  },
  "additionalProperties": false,
  "properties": {
    "env": {
      "default": "PROD",
      "description": "The environment that all assets produced by this connector belong to",
      "title": "Env",
      "type": "string"
    },
    "platform_instance": {
      "anyOf": [
        {
          "type": "string"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "The instance of the platform that all assets produced by this recipe belong to. This should be unique within the platform. See https://docs.datahub.com/docs/platform-instances/ for more details.",
      "title": "Platform Instance"
    },
    "stateful_ingestion": {
      "anyOf": [
        {
          "$ref": "#/$defs/StatefulStaleMetadataRemovalConfig"
        },
        {
          "type": "null"
        }
      ],
      "default": null,
      "description": "Stateful ingestion configuration for stale entity removal."
    },
    "aws_config": {
      "$ref": "#/$defs/AwsConnectionConfig",
      "description": "AWS connection config (credentials, region, role assumption, endpoint override). When omitted, boto3 resolves everything from the standard credential chain. One recipe covers one region."
    },
    "extract_tags": {
      "default": true,
      "description": "Extract AWS resource tags as DataHub tags. SNS allows 10 ListTagsForResource calls per second, so turn this off on the sns source for accounts with tens of thousands of topics.",
      "title": "Extract Tags",
      "type": "boolean"
    },
    "domain": {
      "additionalProperties": {
        "$ref": "#/$defs/AllowDenyPattern"
      },
      "description": "Domain urn or id to regex patterns on resource names; a matching queue or topic is assigned that domain. Domain names are not resolved: use the domain's urn or id.",
      "title": "Domain",
      "type": "object"
    },
    "account_platform_instance_map": {
      "additionalProperties": {
        "type": "string"
      },
      "description": "AWS account id to platform_instance, for lineage to resources in other accounts whose own recipes set a platform_instance other than the account id. Accounts not listed use their account id.",
      "title": "Account Platform Instance Map",
      "type": "object"
    },
    "remove_stale_lineage": {
      "default": true,
      "description": "Remove lineage edges this source wrote on an earlier run that no longer exist in AWS, such as a removed redrive policy or subscription. Only edges from resources this run read in full are removed. Needs a DataHub graph connection (the datahub-rest sink or datahub_api); costs one DataHub lookup per queue or topic.",
      "title": "Remove Stale Lineage",
      "type": "boolean"
    },
    "topic_pattern": {
      "$ref": "#/$defs/AllowDenyPattern",
      "description": "Regex patterns on topic names to include."
    },
    "include_subscription_attributes": {
      "default": false,
      "description": "Call sns:GetSubscriptionAttributes for each subscription to read its dead-letter queue. One call per subscription, for every protocol, on every run; subscription-to-queue edges need no extra calls.",
      "title": "Include Subscription Attributes",
      "type": "boolean"
    },
    "data_protection": {
      "$ref": "#/$defs/SnsDataProtectionConfig",
      "description": "SNS message data protection policies as tags or glossary terms."
    }
  },
  "title": "SnsSourceConfig",
  "type": "object"
}
```





### Capabilities

#### Platform instance and URNs

When `platform_instance` is not set, the connector reads the account id from a topic ARN and uses it. Dataset names include the region. The `sqs` source names queues the same way, through the same shared code.

#### Subscription lineage

| Edge                                        | Read from                                                              |
| ------------------------------------------- | ---------------------------------------------------------------------- |
| Topic to SQS queue                          | `sns:ListSubscriptionsByTopic`, protocol `sqs`                         |
| Topic to a subscription's dead-letter queue | the subscription's `RedrivePolicy` (`include_subscription_attributes`) |

Edges are upstream lineage patches on the queue, so the queue keeps its dead-letter lineage from the `sqs` source and edges from other accounts' recipes. Each edge carries a `relationship` property (`subscription` or `subscription_dead_letter`).

When a subscription is removed, the next run removes its edge (`remove_stale_lineage`, on by default). This reads each topic's current lineage from DataHub, so it needs a graph connection (the `datahub-rest` sink or `datahub_api`); without one, old edges stay. Only `subscription` edges are removed, plus `subscription_dead_letter` edges when `include_subscription_attributes` is on, and only from topics whose subscriptions were all read this run. If `sqs:ListQueues` is denied or fails partway, no edges are removed that run. Edges the `sqs` source or other accounts' recipes write are never touched.

`include_subscription_attributes` is off by default: it makes one `sns:GetSubscriptionAttributes` call per subscription, of every protocol, on every run. The count is in the report as `subscription_attribute_calls`. Turn it on if subscriptions have dead-letter queues you want in lineage.

A queue in the recipe's own account and region gets an edge only if `sqs:ListQueues` shows it exists. A queue in another account or region gets an edge under that account's platform instance: the account id, or the value in `account_platform_instance_map`. Unconfirmed and deleted subscriptions, which SNS lists with a placeholder such as `PendingConfirmation` or `Deleted` instead of an ARN, are skipped. Skipped edges are listed in the report under `lineage_edges_skipped`.

Topics record how many subscriptions each protocol has, and the hosts of HTTP(S) endpoints (`web_endpoint_hosts`). Email and SMS endpoints and full endpoint URLs are never stored.

#### Tags and data protection

AWS resource tags become DataHub tags named `Key:Value` (or `Key` for an empty value), the same format as the `kinesis` and `sqs` sources. They are added to the topic rather than replacing its tags, so tags added in the DataHub UI stay; a tag removed in AWS stays in DataHub until removed there. Use the `extract_ownership_from_tags` transformer to turn an `owner` tag into ownership.

With `data_protection.enabled`, each topic's [message data protection](https://docs.aws.amazon.com/sns/latest/dg/sns-message-data-protection.html) policy is read and every data identifier it names (for example `EmailAddress`) becomes a tag `aws_data_identifier:EmailAddress`. Map identifiers to existing glossary terms with `data_protection.glossary_term_map` to attach terms as well; no terms are created. These tags and terms are added the same way as resource tags: if a topic's policy stops covering an identifier, its tag and term stay until removed in DataHub, so check them before relying on them as a current classification.

#### Domains

`domain` maps a domain urn or id to regex patterns on topic names, as in the `glue` source.

### Limitations

- One region per recipe.
- No schema metadata: SNS doesn't store messages.
- Lineage to a queue that the `sqs` recipe's `queue_pattern` excludes is still written, because the queue exists and its URN follows from its ARN. This source doesn't read the `sqs` recipe's filters. The queue then appears in DataHub with only its lineage.
- Subscriptions to Lambda, Firehose and HTTP endpoints are counted but not yet drawn as lineage.

### Troubleshooting

- **"Listing denied" failure on `sns:ListTopics`**: add the statement above.
- **"Could not read a resource ARN to find the AWS account id" error**: the first `sns:ListTopics` call failed, so the run stopped rather than write topics under URNs without the account id. Re-run, or set `platform_instance` explicitly.
- **Edges skipped with "not found in this region"**: the queue doesn't exist, or `sqs:ListQueues` was denied.
- **Lineage doesn't join with the `sqs` source**: check that both recipes use the same `env`, `platform_instance` and `account_platform_instance_map`.


### Code Coordinates
- Class Name: `datahub.ingestion.source.aws_messaging.sns_source.SnsSource`
- Browse on [GitHub](https://github.com/datahub-project/datahub/blob/master/metadata-ingestion/src/datahub/ingestion/source/aws_messaging/sns_source.py)


:::tip Questions?

If you've got any questions on configuring ingestion for Amazon SNS, feel free to ping us on [our Slack](https://datahub.com/slack).
:::



:::note 💡 **Contributing to this documentation**
This page is auto-generated from the underlying source code. To make changes, please edit the relevant source files in the [metadata-ingestion](https://github.com/datahub-project/datahub/tree/master/metadata-ingestion) directory. 

**Tip:** For quick typo fixes or documentation updates, you can click the ✏️ **Edit** icon directly in the GitHub UI to open a Pull Request. For larger changes and PR naming conventions, please refer to our [Contributing Guide](/docs/contributing).
:::
