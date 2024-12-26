<!-- BEGIN_TF_DOCS -->
# Terraform Bedrock Module

Amazon Bedrock is a fully managed service that offers a choice of foundation models (FMs) along with a broad set of capabilities for building generative AI applications.

This module includes resources to deploy Bedrock features.

## Knowledge Bases

With Knowledge Bases for Amazon Bedrock, you can give FMs and agents contextual information from your company’s private data sources for Retrieval Augmented Generation (RAG) to deliver more relevant, accurate, and customized responses.

### Create a Knowledge Base

A vector index on a vector store is required to create a Knowledge Base. This construct currently supports Amazon OpenSearch Serverless, Amazon RDS Aurora PostgreSQL, Pinecone, and MongoDB. By default, this resource will create an OpenSearch Serverless vector collection and index for each Knowledge Base you create, but you can provide an existing collection to have more control. For other resources you need to have the vector stores already created and credentials stored in AWS Secrets Manager.

The resource accepts an instruction prop that is provided to any Bedrock Agent it is associated with so the agent can decide when to query the Knowledge Base.

To create a knowledge base, make sure you pass in the appropriate variables and set the `create_kb` variable to `true`.

Example default Opensearch Serverless Agent with Knowledgebase

```hcl
provider "opensearch" {
  url         = module.bedrock.default_collection[0].collection_endpoint
  healthcheck = false
}

module "bedrock" {
  source  = "aws-ia/bedrock/aws"
  version = "0.0.5"
  create_kb = true
  create_default_kb = true
  foundation_model = "anthropic.claude-v2"
  instruction = "You are an automotive assisant who can provide detailed information about cars to a customer."
}
```

### Knowledge Base - Data Sources

Data sources are the various repositories or systems from which information is extracted and ingested into the knowledge base. These sources provide the raw content that will be processed, indexed, and made available for querying within the knowledge base system. Data sources can include various types of systems such as document management systems, databases, file storage systems, and content management platforms. Suuported Data Sources include Amazon S3 buckets, Web Crawlers, SharePoint sites, Salesforce instances, and Confluence spaces.

- Amazon S3. You can either create a new data source by passing in the existing data source arn to the input variable `kb_s3_data_source` or create a new one by setting `create_s3_data_source` to true.

- Web Crawler. You can create a new web crawler data source by setting the `create_web_crawler` input variable to true and passing in the necessary variables for urls, scope, etc.

- SharePoint. You can create a new SharePoint data source by setting the `create_sharepoint` input variable to true and passing in the necessary variables for site urls, filter patterns, etc.

- Salesforce. You can create a new Salesforce data source by setting the `create_salesforce` input variable to true and passing in the necessary variables for site urls, filter patterns, etc.

- Confluence. You can create a new Confluence data source by setting the `create_confluence` input variable to true and passing in the necessary variables for site urls, filter patterns, etc.

## Agents

Enable generative AI applications to execute multistep tasks across company systems and data sources.

### Create an Agent

The following example creates an Agent with a simple instruction and without any action groups or knowedlge bases.

```hcl
module "bedrock" {
  source  = "aws-ia/bedrock/aws"
  version = "0.0.5"
  foundation_model = "anthropic.claude-v2"
  instruction = "You are an automotive assisant who can provide detailed information about cars to a customer."
}
```

To create an Agent with a default Knowledge Base you simply set `create_kb` and `create_default_kb` to `true`:

```hcl
module "bedrock" {
  source  = "aws-ia/bedrock/aws"
  version = "0.0.5"
  create_kb = true
  create_default_kb = true
  foundation_model = "anthropic.claude-v2"
  instruction = "You are an automotive assisant who can provide detailed information about cars to a customer."
}
```

### Action Groups

An action group defines functions your agent can call. The functions are Lambda functions. The action group uses an OpenAPI schema to tell the agent what your functions do and how to call them. You can configure an action group by passing in the appropriate input variables.

### Prepare the Agent

The Agent constructs take an optional parameter shouldPrepareAgent to indicate that the Agent should be prepared after any updates to an agent, Knowledge Base association, or action group. This may increase the time to create and update those resources. By default, this value is true.

### Prompt Overrides

Bedrock Agents allows you to customize the prompts and LLM configuration for its different steps. You can disable steps or create a new prompt template. Prompt templates can be inserted from plain text files.

## Bedrock Guardrails

Amazon Bedrock's Guardrails feature enables you to implement robust governance and control mechanisms for your generative AI applications, ensuring alignment with your specific use cases and responsible AI policies. Guardrails empowers you to create multiple tailored policy configurations, each designed to address the unique requirements and constraints of different use cases. These policy configurations can then be seamlessly applied across multiple foundation models (FMs) and Agents, ensuring a consistent user experience and standardizing safety, security, and privacy controls throughout your generative AI ecosystem.

With Guardrails, you can define and enforce granular, customizable policies to precisely govern the behavior of your generative AI applications. You can configure the following policies in a guardrail to avoid undesirable and harmful content and remove sensitive information for privacy protection.

Content filters – Adjust filter strengths to block input prompts or model responses containing harmful content.

Denied topics – Define a set of topics that are undesirable in the context of your application. These topics will be blocked if detected in user queries or model responses.

Word filters – Configure filters to block undesirable words, phrases, and profanity. Such words can include offensive terms, competitor names etc.

Sensitive information filters – Block or mask sensitive information such as personally identifiable information (PII) or custom regex in user inputs and model responses.

You can create a Guardrail by setting `create_guardrail` to true and passing in the appropriate input variables:

```hcl
module "bedrock" {
  source  = "aws-ia/bedrock/aws"
  version = "0.0.5"
  create_kb = false
  create_default_kb = false
  create_guardrail = true
  blocked_input = "Blocked input"
  blocked_output = "Blocked output"
  filters_config = [
      {
        input_strength  = "MEDIUM"
        output_strength = "MEDIUM"
        type            = "HATE"
      },
      {
        input_strength  = "HIGH"
        output_strength = "HIGH"
        type            = "VIOLENCE"
      }
  ]
  pii_entities_config = [
      {
        action = "BLOCK"
        type   = "NAME"
      },
      {
        action = "BLOCK"
        type   = "DRIVER_ID"
      },
      {
        action = "ANONYMIZE"
        type   = "USERNAME"
      },
  ]
  regexes_config = [{
      action      = "BLOCK"
      description = "example regex"
      name        = "regex_example"
      pattern     = "^\\d{3}-\\d{2}-\\d{4}$"
  }]
  managed_word_lists_config = [{
      type = "PROFANITY"
  }]
  words_config = [{
    text = "HATE"
  }]
  topics_config = [{
      name       = "investment_topic"
      examples   = ["Where should I invest my money ?"]
      type       = "DENY"
      definition = "Investment advice refers to inquiries, guidance, or recommendations regarding the management or allocation of funds or assets with the goal of generating returns ."
  }]
  foundation_model = "anthropic.claude-v2"
  instruction = "You are an automotive assisant who can provide detailed information about cars to a customer."
}
```

## Prompt Management

Amazon Bedrock provides the ability to create and save prompts using Prompt management so that you can save time by applying the same prompt to different workflows. You can include variables in the prompt so that you can adjust the prompt for different use case.

### Prompt Variants

Prompt variants in the context of Amazon Bedrock refer to alternative configurations of a prompt, including its message or the model and inference configurations used. Prompt variants allow you to create different versions of a prompt, test them, and save the variant that works best for your use case. You can add prompt variants to a prompt by passing in the values for the ```variants_list
``` variable:

```hcl
  variants_list = [
    {
      name          = "variant-example"
      template_type = "TEXT"
      model_id      = "amazon.titan-text-express-v1"
      inference_configuration = {
        text = {
          temperature    = 1
          top_p          = 0.9900000095367432
          max_tokens     = 300
          stop_sequences = ["User:"]
          top_k          = 250
        }
      }
      template_configuration = {
        text = {
          input_variables = [
            {
              name = "topic"
            }
          ]
          text = "Make me a {{genre}} playlist consisting of the following number of songs: {{number}}."
        }
      }
    }
  ]
```

### Prompt Version

A prompt version is a snapshot of a prompt at a specific point in time that you create when you are satisfied with a set of configurations. Versions allow you to deploy your prompt and easily switch between different configurations for your prompt and update your application with the most appropriate version for your use-case.

You can create a Prompt version by setting `create_prompt_version` to true and adding an optional `prompt_version_description` and optional `prompt_version_tags`.

Creating a prompt with a prompt version would look like:

```hcl
module "bedrock" {
  source = "../.." # local example
  create_kb = false
  create_default_kb = false
  create_s3_data_source = false
  create_agent = false

  # Prompt Management
  prompt_name = "prompt"
  default_variant = "variant-example"
  create_prompt = true
  create_prompt_version = true
  prompt_version_description = "Example prompt version"
  variants_list = [
    {
      name          = "variant-example"
      template_type = "TEXT"
      model_id      = "amazon.titan-text-express-v1"
      inference_configuration = {
        text = {
          temperature    = 1
          top_p          = 0.9900000095367432
          max_tokens     = 300
          stop_sequences = ["User:"]
          top_k          = 250
        }
      }
      template_configuration = {
        text = {
          input_variables = [
            {
              name = "topic"
            }
          ]
          text = "Make me a {{genre}} playlist consisting of the following number of songs: {{number}}."
        }
      }
    }

  ]

}
```

## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.0.7 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | ~>5.0 |
| <a name="requirement_awscc"></a> [awscc](#requirement\_awscc) | >= 1.0.0 |
| <a name="requirement_opensearch"></a> [opensearch](#requirement\_opensearch) | = 2.2.0 |
| <a name="requirement_random"></a> [random](#requirement\_random) | >= 3.6.0 |
| <a name="requirement_time"></a> [time](#requirement\_time) | ~> 0.6 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | ~>5.0 |
| <a name="provider_awscc"></a> [awscc](#provider\_awscc) | >= 1.0.0 |
| <a name="provider_opensearch"></a> [opensearch](#provider\_opensearch) | = 2.2.0 |
| <a name="provider_random"></a> [random](#provider\_random) | >= 3.6.0 |
| <a name="provider_time"></a> [time](#provider\_time) | ~> 0.6 |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [aws_bedrockagent_data_source.knowledge_base_ds](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/bedrockagent_data_source) | resource |
| [aws_cloudwatch_log_group.knowledge_base_cwl](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group) | resource |
| [aws_iam_policy.bedrock_kb_s3_decryption_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_iam_policy.bedrock_knowledge_base_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_iam_policy.bedrock_knowledge_base_policy_s3](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | resource |
| [aws_iam_role.agent_role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | resource |
| [aws_iam_role.bedrock_knowledge_base_role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | resource |
| [aws_iam_role_policy.action_group_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_iam_role_policy.agent_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_iam_role_policy.bedrock_kb_oss](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_iam_role_policy.guardrail_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_iam_role_policy.kb_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy) | resource |
| [aws_iam_role_policy_attachment.bedrock_kb_s3_decryption_policy_attachment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment) | resource |
| [aws_iam_role_policy_attachment.bedrock_knowledge_base_policy_attachment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment) | resource |
| [aws_iam_role_policy_attachment.bedrock_knowledge_base_policy_s3_attachment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment) | resource |
| [aws_lambda_permission.allow_bedrock_agent](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_permission) | resource |
| [aws_opensearchserverless_access_policy.data_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/opensearchserverless_access_policy) | resource |
| [aws_opensearchserverless_security_policy.nw_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/opensearchserverless_security_policy) | resource |
| [aws_opensearchserverless_security_policy.security_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/opensearchserverless_security_policy) | resource |
| [awscc_bedrock_agent.bedrock_agent](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_agent) | resource |
| [awscc_bedrock_agent_alias.bedrock_agent_alias](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_agent_alias) | resource |
| [awscc_bedrock_data_source.knowledge_base_confluence](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_data_source) | resource |
| [awscc_bedrock_data_source.knowledge_base_salesforce](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_data_source) | resource |
| [awscc_bedrock_data_source.knowledge_base_sharepoint](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_data_source) | resource |
| [awscc_bedrock_data_source.knowledge_base_web_crawler](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_data_source) | resource |
| [awscc_bedrock_guardrail.guardrail](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_guardrail) | resource |
| [awscc_bedrock_guardrail_version.guardrail](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_guardrail_version) | resource |
| [awscc_bedrock_knowledge_base.knowledge_base_default](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_knowledge_base) | resource |
| [awscc_bedrock_knowledge_base.knowledge_base_mongo](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_knowledge_base) | resource |
| [awscc_bedrock_knowledge_base.knowledge_base_opensearch](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_knowledge_base) | resource |
| [awscc_bedrock_knowledge_base.knowledge_base_pinecone](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_knowledge_base) | resource |
| [awscc_bedrock_knowledge_base.knowledge_base_rds](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_knowledge_base) | resource |
| [awscc_bedrock_prompt.prompt](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_prompt) | resource |
| [awscc_bedrock_prompt_version.prompt_version](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/bedrock_prompt_version) | resource |
| [awscc_logs_delivery.knowledge_base_log_delivery](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/logs_delivery) | resource |
| [awscc_logs_delivery_destination.knowledge_base_log_destination](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/logs_delivery_destination) | resource |
| [awscc_logs_delivery_source.knowledge_base_log_source](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/logs_delivery_source) | resource |
| [awscc_opensearchserverless_collection.default_collection](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/opensearchserverless_collection) | resource |
| [awscc_s3_bucket.s3_data_source](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs/resources/s3_bucket) | resource |
| [opensearch_index.default_oss_index](https://registry.terraform.io/providers/opensearch-project/opensearch/2.2.0/docs/resources/index) | resource |
| [random_string.solution_prefix](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string) | resource |
| [time_sleep.wait_after_index_creation](https://registry.terraform.io/providers/hashicorp/time/latest/docs/resources/sleep) | resource |
| [time_sleep.wait_before_index_creation](https://registry.terraform.io/providers/hashicorp/time/latest/docs/resources/sleep) | resource |
| [aws_caller_identity.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/caller_identity) | data source |
| [aws_iam_policy_document.agent_permissions](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_iam_policy_document.agent_trust](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_iam_policy_document.knowledge_base_permissions](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_partition.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/partition) | data source |
| [aws_region.current](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/region) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_action_group_description"></a> [action\_group\_description](#input\_action\_group\_description) | Description of the action group. | `string` | `null` | no |
| <a name="input_action_group_name"></a> [action\_group\_name](#input\_action\_group\_name) | Name of the action group. | `string` | `null` | no |
| <a name="input_action_group_state"></a> [action\_group\_state](#input\_action\_group\_state) | State of the action group. | `string` | `null` | no |
| <a name="input_agent_alias_description"></a> [agent\_alias\_description](#input\_agent\_alias\_description) | Description of the agent alias. | `string` | `null` | no |
| <a name="input_agent_alias_name"></a> [agent\_alias\_name](#input\_agent\_alias\_name) | The name of the guardrail. | `string` | `"TerraformBedrockAgentAlias"` | no |
| <a name="input_agent_alias_tags"></a> [agent\_alias\_tags](#input\_agent\_alias\_tags) | Tag bedrock agent alias resource. | `map(string)` | `null` | no |
| <a name="input_agent_description"></a> [agent\_description](#input\_agent\_description) | A description of agent. | `string` | `null` | no |
| <a name="input_agent_id"></a> [agent\_id](#input\_agent\_id) | Agent identifier. | `string` | `null` | no |
| <a name="input_agent_name"></a> [agent\_name](#input\_agent\_name) | The name of your agent. | `string` | `"TerraformBedrockAgents"` | no |
| <a name="input_api_schema_payload"></a> [api\_schema\_payload](#input\_api\_schema\_payload) | String OpenAPI Payload. | `string` | `null` | no |
| <a name="input_api_schema_s3_bucket_name"></a> [api\_schema\_s3\_bucket\_name](#input\_api\_schema\_s3\_bucket\_name) | A bucket in S3. | `string` | `null` | no |
| <a name="input_api_schema_s3_object_key"></a> [api\_schema\_s3\_object\_key](#input\_api\_schema\_s3\_object\_key) | An object key in S3. | `string` | `null` | no |
| <a name="input_auth_type"></a> [auth\_type](#input\_auth\_type) | The supported authentication type. | `string` | `null` | no |
| <a name="input_base_prompt_template"></a> [base\_prompt\_template](#input\_base\_prompt\_template) | Defines the prompt template with which to replace the default prompt template. | `string` | `null` | no |
| <a name="input_bedrock_agent_version"></a> [bedrock\_agent\_version](#input\_bedrock\_agent\_version) | Agent version. | `string` | `null` | no |
| <a name="input_blocked_input_messaging"></a> [blocked\_input\_messaging](#input\_blocked\_input\_messaging) | Messaging for when violations are detected in text. | `string` | `"Blocked input"` | no |
| <a name="input_blocked_outputs_messaging"></a> [blocked\_outputs\_messaging](#input\_blocked\_outputs\_messaging) | Messaging for when violations are detected in text. | `string` | `"Blocked output"` | no |
| <a name="input_collection_arn"></a> [collection\_arn](#input\_collection\_arn) | The ARN of the collection. | `string` | `null` | no |
| <a name="input_collection_name"></a> [collection\_name](#input\_collection\_name) | The name of the collection. | `string` | `null` | no |
| <a name="input_confluence_credentials_secret_arn"></a> [confluence\_credentials\_secret\_arn](#input\_confluence\_credentials\_secret\_arn) | The ARN of an AWS Secrets Manager secret that stores your authentication credentials for your Confluence instance URL. | `string` | `null` | no |
| <a name="input_connection_string"></a> [connection\_string](#input\_connection\_string) | The endpoint URL for your index management page. | `string` | `null` | no |
| <a name="input_crawl_filter_type"></a> [crawl\_filter\_type](#input\_crawl\_filter\_type) | The crawl filter type. | `string` | `null` | no |
| <a name="input_crawler_scope"></a> [crawler\_scope](#input\_crawler\_scope) | The scope that a web crawl job will be restricted to. | `string` | `null` | no |
| <a name="input_create_ag"></a> [create\_ag](#input\_create\_ag) | Whether or not to create an action group. | `bool` | `false` | no |
| <a name="input_create_agent"></a> [create\_agent](#input\_create\_agent) | Whether or not to deploy an agent. | `bool` | `true` | no |
| <a name="input_create_agent_alias"></a> [create\_agent\_alias](#input\_create\_agent\_alias) | Whether or not to create an agent alias. | `bool` | `false` | no |
| <a name="input_create_confluence"></a> [create\_confluence](#input\_create\_confluence) | Whether or not create a Confluence data source. | `bool` | `false` | no |
| <a name="input_create_default_kb"></a> [create\_default\_kb](#input\_create\_default\_kb) | Whether or not to create the default knowledge base. | `bool` | `false` | no |
| <a name="input_create_guardrail"></a> [create\_guardrail](#input\_create\_guardrail) | Whether or not to create a guardrail. | `bool` | `false` | no |
| <a name="input_create_kb"></a> [create\_kb](#input\_create\_kb) | Whether or not to attach a knowledge base. | `bool` | `false` | no |
| <a name="input_create_kb_log_group"></a> [create\_kb\_log\_group](#input\_create\_kb\_log\_group) | Whether or not to create a log group for the knowledge base. | `bool` | `false` | no |
| <a name="input_create_mongo_config"></a> [create\_mongo\_config](#input\_create\_mongo\_config) | Whether or not to use MongoDB Atlas configuration | `bool` | `false` | no |
| <a name="input_create_opensearch_config"></a> [create\_opensearch\_config](#input\_create\_opensearch\_config) | Whether or not to use Opensearch Serverless configuration | `bool` | `false` | no |
| <a name="input_create_pinecone_config"></a> [create\_pinecone\_config](#input\_create\_pinecone\_config) | Whether or not to use Pinecone configuration | `bool` | `false` | no |
| <a name="input_create_prompt"></a> [create\_prompt](#input\_create\_prompt) | Whether or not to create a prompt resource. | `bool` | `false` | no |
| <a name="input_create_prompt_version"></a> [create\_prompt\_version](#input\_create\_prompt\_version) | Whether or not to create a prompt version. | `bool` | `false` | no |
| <a name="input_create_rds_config"></a> [create\_rds\_config](#input\_create\_rds\_config) | Whether or not to use RDS configuration | `bool` | `false` | no |
| <a name="input_create_s3_data_source"></a> [create\_s3\_data\_source](#input\_create\_s3\_data\_source) | Whether or not to create the S3 data source. | `bool` | `true` | no |
| <a name="input_create_salesforce"></a> [create\_salesforce](#input\_create\_salesforce) | Whether or not create a Salesforce data source. | `bool` | `false` | no |
| <a name="input_create_sharepoint"></a> [create\_sharepoint](#input\_create\_sharepoint) | Whether or not create a Share Point data source. | `bool` | `false` | no |
| <a name="input_create_web_crawler"></a> [create\_web\_crawler](#input\_create\_web\_crawler) | Whether or not create a web crawler data source. | `bool` | `false` | no |
| <a name="input_credentials_secret_arn"></a> [credentials\_secret\_arn](#input\_credentials\_secret\_arn) | The ARN of the secret in Secrets Manager that is linked to your database | `string` | `null` | no |
| <a name="input_custom_control"></a> [custom\_control](#input\_custom\_control) | Custom control of action execution. | `string` | `null` | no |
| <a name="input_customer_encryption_key_arn"></a> [customer\_encryption\_key\_arn](#input\_customer\_encryption\_key\_arn) | A KMS key ARN. | `string` | `null` | no |
| <a name="input_database_name"></a> [database\_name](#input\_database\_name) | Name of the database. | `string` | `null` | no |
| <a name="input_default_variant"></a> [default\_variant](#input\_default\_variant) | Name for a variant. | `string` | `null` | no |
| <a name="input_endpoint"></a> [endpoint](#input\_endpoint) | Database endpoint | `string` | `null` | no |
| <a name="input_endpoint_service_name"></a> [endpoint\_service\_name](#input\_endpoint\_service\_name) | MongoDB Atlas endpoint service name. | `string` | `null` | no |
| <a name="input_exclusion_filters"></a> [exclusion\_filters](#input\_exclusion\_filters) | A set of regular expression filter patterns for a type of object. | `list(string)` | `[]` | no |
| <a name="input_existing_kb"></a> [existing\_kb](#input\_existing\_kb) | The ID of the existing knowledge base. | `string` | `null` | no |
| <a name="input_filters_config"></a> [filters\_config](#input\_filters\_config) | List of content filter configs in content policy. | `list(map(string))` | `null` | no |
| <a name="input_foundation_model"></a> [foundation\_model](#input\_foundation\_model) | The foundation model for the Bedrock agent. | `string` | `null` | no |
| <a name="input_guardrail_description"></a> [guardrail\_description](#input\_guardrail\_description) | Description of the guardrail. | `string` | `null` | no |
| <a name="input_guardrail_kms_key_arn"></a> [guardrail\_kms\_key\_arn](#input\_guardrail\_kms\_key\_arn) | KMS encryption key to use for the guardrail. | `string` | `null` | no |
| <a name="input_guardrail_name"></a> [guardrail\_name](#input\_guardrail\_name) | The name of the guardrail. | `string` | `"TerraformBedrockGuardrail"` | no |
| <a name="input_guardrail_tags"></a> [guardrail\_tags](#input\_guardrail\_tags) | A map of tags keys and values for the knowledge base. | `list(map(string))` | `null` | no |
| <a name="input_host_type"></a> [host\_type](#input\_host\_type) | The supported host type, whether online/cloud or server/on-premises. | `string` | `null` | no |
| <a name="input_host_url"></a> [host\_url](#input\_host\_url) | The host URL or instance URL. | `string` | `null` | no |
| <a name="input_idle_session_ttl"></a> [idle\_session\_ttl](#input\_idle\_session\_ttl) | How long sessions should be kept open for the agent. | `number` | `600` | no |
| <a name="input_inclusion_filters"></a> [inclusion\_filters](#input\_inclusion\_filters) | A set of regular expression filter patterns for a type of object. | `list(string)` | `[]` | no |
| <a name="input_instruction"></a> [instruction](#input\_instruction) | A narrative instruction to provide the agent as context. | `string` | `""` | no |
| <a name="input_kb_description"></a> [kb\_description](#input\_kb\_description) | Description of knowledge base. | `string` | `"Terraform deployed Knowledge Base"` | no |
| <a name="input_kb_embedding_model_arn"></a> [kb\_embedding\_model\_arn](#input\_kb\_embedding\_model\_arn) | The ARN of the model used to create vector embeddings for the knowledge base. | `string` | `"arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v1"` | no |
| <a name="input_kb_log_group_retention_in_days"></a> [kb\_log\_group\_retention\_in\_days](#input\_kb\_log\_group\_retention\_in\_days) | The retention period of the knowledge base log group. | `number` | `0` | no |
| <a name="input_kb_monitoring_arn"></a> [kb\_monitoring\_arn](#input\_kb\_monitoring\_arn) | The ARN of the target for delivery of knowledge base application logs | `string` | `null` | no |
| <a name="input_kb_name"></a> [kb\_name](#input\_kb\_name) | Name of the knowledge base. | `string` | `"knowledge-base"` | no |
| <a name="input_kb_role_arn"></a> [kb\_role\_arn](#input\_kb\_role\_arn) | The ARN of the IAM role with permission to invoke API operations on the knowledge base. | `string` | `null` | no |
| <a name="input_kb_s3_data_source"></a> [kb\_s3\_data\_source](#input\_kb\_s3\_data\_source) | The S3 data source ARN for the knowledge base. | `string` | `null` | no |
| <a name="input_kb_s3_data_source_kms_arn"></a> [kb\_s3\_data\_source\_kms\_arn](#input\_kb\_s3\_data\_source\_kms\_arn) | The ARN of the KMS key used to encrypt S3 content | `string` | `null` | no |
| <a name="input_kb_state"></a> [kb\_state](#input\_kb\_state) | State of knowledge base; whether it is enabled or disabled | `string` | `"ENABLED"` | no |
| <a name="input_kb_storage_type"></a> [kb\_storage\_type](#input\_kb\_storage\_type) | The storage type of a knowledge base. | `string` | `null` | no |
| <a name="input_kb_tags"></a> [kb\_tags](#input\_kb\_tags) | A map of tags keys and values for the knowledge base. | `map(string)` | `null` | no |
| <a name="input_kb_type"></a> [kb\_type](#input\_kb\_type) | The type of a knowledge base. | `string` | `null` | no |
| <a name="input_kms_key_arn"></a> [kms\_key\_arn](#input\_kms\_key\_arn) | KMS encryption key to use for the agent. | `string` | `null` | no |
| <a name="input_lambda_action_group_executor"></a> [lambda\_action\_group\_executor](#input\_lambda\_action\_group\_executor) | ARN of Lambda. | `string` | `null` | no |
| <a name="input_managed_word_lists_config"></a> [managed\_word\_lists\_config](#input\_managed\_word\_lists\_config) | A config for the list of managed words. | `list(map(string))` | `null` | no |
| <a name="input_max_length"></a> [max\_length](#input\_max\_length) | The maximum number of tokens to generate in the response. | `number` | `0` | no |
| <a name="input_metadata_field"></a> [metadata\_field](#input\_metadata\_field) | The name of the field in which Amazon Bedrock stores metadata about the vector store. | `string` | `"AMAZON_BEDROCK_METADATA"` | no |
| <a name="input_name_prefix"></a> [name\_prefix](#input\_name\_prefix) | This value is appended at the beginning of resource names. | `string` | `"BedrockAgents"` | no |
| <a name="input_namespace"></a> [namespace](#input\_namespace) | The namespace to be used to write new data to your pinecone database | `string` | `null` | no |
| <a name="input_override_lambda_arn"></a> [override\_lambda\_arn](#input\_override\_lambda\_arn) | The ARN of the Lambda function to use when parsing the raw foundation model output in parts of the agent sequence. | `string` | `null` | no |
| <a name="input_parent_action_group_signature"></a> [parent\_action\_group\_signature](#input\_parent\_action\_group\_signature) | Action group signature for a builtin action. | `string` | `null` | no |
| <a name="input_parser_mode"></a> [parser\_mode](#input\_parser\_mode) | Specifies whether to override the default parser Lambda function. | `string` | `null` | no |
| <a name="input_pattern_object_filter_list"></a> [pattern\_object\_filter\_list](#input\_pattern\_object\_filter\_list) | List of pattern object information. | <pre>list(object({<br>    exclusion_filters = optional(list(string))<br>    inclusion_filters = optional(list(string))<br>    object_type       = optional(string)<br><br>  }))</pre> | `[]` | no |
| <a name="input_pii_entities_config"></a> [pii\_entities\_config](#input\_pii\_entities\_config) | List of entities. | `list(map(string))` | `null` | no |
| <a name="input_primary_key_field"></a> [primary\_key\_field](#input\_primary\_key\_field) | The name of the field in which Bedrock stores the ID for each entry. | `string` | `null` | no |
| <a name="input_prompt_creation_mode"></a> [prompt\_creation\_mode](#input\_prompt\_creation\_mode) | Specifies whether to override the default prompt template. | `string` | `null` | no |
| <a name="input_prompt_description"></a> [prompt\_description](#input\_prompt\_description) | Description for a prompt resource. | `string` | `null` | no |
| <a name="input_prompt_name"></a> [prompt\_name](#input\_prompt\_name) | Name for a prompt resource. | `string` | `null` | no |
| <a name="input_prompt_override"></a> [prompt\_override](#input\_prompt\_override) | Whether to provide prompt override configuration. | `bool` | `false` | no |
| <a name="input_prompt_state"></a> [prompt\_state](#input\_prompt\_state) | Specifies whether to allow the agent to carry out the step specified in the promptType. | `string` | `null` | no |
| <a name="input_prompt_tags"></a> [prompt\_tags](#input\_prompt\_tags) | A map of tag keys and values for prompt resource. | `map(string)` | `null` | no |
| <a name="input_prompt_type"></a> [prompt\_type](#input\_prompt\_type) | The step in the agent sequence that this prompt configuration applies to. | `string` | `null` | no |
| <a name="input_prompt_version_description"></a> [prompt\_version\_description](#input\_prompt\_version\_description) | Description for a prompt version resource. | `string` | `null` | no |
| <a name="input_prompt_version_tags"></a> [prompt\_version\_tags](#input\_prompt\_version\_tags) | A map of tag keys and values for a prompt version resource. | `map(string)` | `null` | no |
| <a name="input_rate_limit"></a> [rate\_limit](#input\_rate\_limit) | Rate of web URLs retrieved per minute. | `number` | `null` | no |
| <a name="input_regexes_config"></a> [regexes\_config](#input\_regexes\_config) | List of regex. | `list(map(string))` | `null` | no |
| <a name="input_resource_arn"></a> [resource\_arn](#input\_resource\_arn) | The ARN of the vector store. | `string` | `null` | no |
| <a name="input_salesforce_credentials_secret_arn"></a> [salesforce\_credentials\_secret\_arn](#input\_salesforce\_credentials\_secret\_arn) | The ARN of an AWS Secrets Manager secret that stores your authentication credentials for your Salesforce instance URL. | `string` | `null` | no |
| <a name="input_seed_urls"></a> [seed\_urls](#input\_seed\_urls) | A list of web urls. | `list(object({url = string}))` | `[]` | no |
| <a name="input_share_point_credentials_secret_arn"></a> [share\_point\_credentials\_secret\_arn](#input\_share\_point\_credentials\_secret\_arn) | The ARN of an AWS Secrets Manager secret that stores your authentication credentials for your SharePoint site/sites. | `string` | `null` | no |
| <a name="input_share_point_domain"></a> [share\_point\_domain](#input\_share\_point\_domain) | The domain of your SharePoint instance or site URL/URLs. | `string` | `null` | no |
| <a name="input_share_point_site_urls"></a> [share\_point\_site\_urls](#input\_share\_point\_site\_urls) | A list of one or more SharePoint site URLs. | `list(string)` | `[]` | no |
| <a name="input_skip_resource_in_use"></a> [skip\_resource\_in\_use](#input\_skip\_resource\_in\_use) | Specifies whether to allow deleting action group while it is in use. | `bool` | `null` | no |
| <a name="input_stop_sequences"></a> [stop\_sequences](#input\_stop\_sequences) | A list of stop sequences. | `list(string)` | `[]` | no |
| <a name="input_table_name"></a> [table\_name](#input\_table\_name) | The name of the table in the database. | `string` | `null` | no |
| <a name="input_tags"></a> [tags](#input\_tags) | Tag bedrock agent resource. | `map(string)` | `null` | no |
| <a name="input_temperature"></a> [temperature](#input\_temperature) | The likelihood of the model selecting higher-probability options while generating a response. | `number` | `0` | no |
| <a name="input_tenant_id"></a> [tenant\_id](#input\_tenant\_id) | The identifier of your Microsoft 365 tenant. | `string` | `null` | no |
| <a name="input_text_field"></a> [text\_field](#input\_text\_field) | The name of the field in which Amazon Bedrock stores the raw text from your data. | `string` | `"AMAZON_BEDROCK_TEXT_CHUNK"` | no |
| <a name="input_top_k"></a> [top\_k](#input\_top\_k) | Sample from the k most likely next tokens. | `number` | `50` | no |
| <a name="input_top_p"></a> [top\_p](#input\_top\_p) | Cumulative probability cutoff for token selection. | `number` | `0.5` | no |
| <a name="input_topics_config"></a> [topics\_config](#input\_topics\_config) | List of topic configs in topic policy | <pre>list(object({<br>    name       = string<br>    examples   = list(string)<br>    type       = string<br>    definition = string<br>  }))</pre> | `null` | no |
| <a name="input_variants_list"></a> [variants\_list](#input\_variants\_list) | List of prompt variants. | <pre>list(object({<br>    name                    = optional(string)<br>    template_type           = optional(string)<br>    model_id                = optional(string)<br>    inference_configuration = optional(object({<br>                                text = optional(object({<br>                                  max_tokens = optional(number)<br>                                  stop_sequences = optional(list(string))<br>                                  temperature = optional(number)<br>                                  top_p = optional(number)<br>                                }))<br>                              }))<br><br>    template_configuration  = optional(object({<br>                                text = optional(object({<br>                                  input_variables = optional(list(object({ name = optional(string) })))<br>                                  text = optional(string)<br>                                  text_s3_location = optional(object({<br>                                    bucket = optional(string)<br>                                    key = optional(string)<br>                                    version = optional(string)<br>                                  }))<br>                                }))<br>                              }))<br>  }))</pre> | `null` | no |
| <a name="input_vector_field"></a> [vector\_field](#input\_vector\_field) | The name of the field where the vector embeddings are stored | `string` | `"bedrock-knowledge-base-default-vector"` | no |
| <a name="input_vector_index_name"></a> [vector\_index\_name](#input\_vector\_index\_name) | The name of the vector index. | `string` | `"bedrock-knowledge-base-default-index"` | no |
| <a name="input_words_config"></a> [words\_config](#input\_words\_config) | List of custom word configs. | `list(map(string))` | `null` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_bedrock_agent"></a> [bedrock\_agent](#output\_bedrock\_agent) | The Amazon Bedrock Agent if it is created. |
| <a name="output_cloudwatch_log_group"></a> [cloudwatch\_log\_group](#output\_cloudwatch\_log\_group) | The name of the CloudWatch log group for the knowledge base.  If no log group was requested, value will be null |
| <a name="output_datasource_identifier"></a> [datasource\_identifier](#output\_datasource\_identifier) | The unique identifier of the data source. |
| <a name="output_default_collection"></a> [default\_collection](#output\_default\_collection) | Opensearch default collection value. |
| <a name="output_default_kb_identifier"></a> [default\_kb\_identifier](#output\_default\_kb\_identifier) | The unique identifier of the default knowledge base that was created.  If no default KB was requested, value will be null |
| <a name="output_mongo_kb_identifier"></a> [mongo\_kb\_identifier](#output\_mongo\_kb\_identifier) | The unique identifier of the MongoDB knowledge base that was created.  If no MongoDB KB was requested, value will be null |
| <a name="output_opensearch_kb_identifier"></a> [opensearch\_kb\_identifier](#output\_opensearch\_kb\_identifier) | The unique identifier of the OpenSearch knowledge base that was created.  If no OpenSearch KB was requested, value will be null |
| <a name="output_pinecone_kb_identifier"></a> [pinecone\_kb\_identifier](#output\_pinecone\_kb\_identifier) | The unique identifier of the Pinecone knowledge base that was created.  If no Pinecone KB was requested, value will be null |
| <a name="output_rds_kb_identifier"></a> [rds\_kb\_identifier](#output\_rds\_kb\_identifier) | The unique identifier of the RDS knowledge base that was created.  If no RDS KB was requested, value will be null |
| <a name="output_s3_data_source_arn"></a> [s3\_data\_source\_arn](#output\_s3\_data\_source\_arn) | The Amazon Bedrock Data Source for S3. |
<!-- END_TF_DOCS -->