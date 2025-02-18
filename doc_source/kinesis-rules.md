# Amazon Kinesis controls<a name="kinesis-rules"></a>

**Topics**
+ [\[CT\.KINESIS\.PR\.1\] Require any Amazon Kinesis data stream to have encryption at rest configured](#ct-kinesis-pr-1-description)

## \[CT\.KINESIS\.PR\.1\] Require any Amazon Kinesis data stream to have encryption at rest configured<a name="ct-kinesis-pr-1-description"></a>

This control checks whether Amazon Kinesis data streams are encrypted at rest with server\-side encryption\.
+ **Control objective: **Encrypt data at rest
+ **Implementation: **AWS CloudFormation Guard Rule
+ **Control behavior: **Proactive
+ **Resource types: **`AWS::Kinesis::Stream`
+ **AWS CloudFormation guard rule: ** [CT\.KINESIS\.PR\.1 rule specification](#ct-kinesis-pr-1-rule) 

**Details and examples**
+ For details about the PASS, FAIL, and SKIP behaviors associated with this control, see the: [CT\.KINESIS\.PR\.1 rule specification](#ct-kinesis-pr-1-rule) 
+ For examples of PASS and FAIL CloudFormation Templates related to this control, see: [CT\.KINESIS\.PR\.1 example templates](#ct-kinesis-pr-1-templates) 

**Explanation**

Server\-side encryption is a feature in Amazon Kinesis data streams that encrypts data automatically, before the data is at rest, by using an AWS KMS key\. Data is encrypted before it is written to the Kinesis stream storage layer, and decrypted after it is retrieved from storage\. As a result, your data is encrypted at rest within the Amazon Kinesis data stream service\.

### Remediation for rule failure<a name="ct-kinesis-pr-1-remediation"></a>

Specify a `StreamEncryption` configuration, with `EncryptionType` set to `KMS` and `KeyId` set to an AWS KMS key identifier\.

The examples that follow show how to implement this remediation\.

#### Amazon Kinesis Data Stream \- Example<a name="ct-kinesis-pr-1-remediation-1"></a>

Amazon Kinesis data stream configured to encrypt data at rest with server\-side encryption, using an AWS KMS key\. The example is shown in JSON and in YAML\.

**JSON example**

```
{
    "KinesisStream": {
        "Type": "AWS::Kinesis::Stream",
        "Properties": {
            "RetentionPeriodHours": 168,
            "ShardCount": 3,
            "StreamEncryption": {
                "EncryptionType": "KMS",
                "KeyId": {
                    "Ref": "KMSKey"
                }
            }
        }
    }
}
```

**YAML example**

```
KinesisStream:
  Type: AWS::Kinesis::Stream
  Properties:
    RetentionPeriodHours: 168
    ShardCount: 3
    StreamEncryption:
      EncryptionType: KMS
      KeyId: !Ref 'KMSKey'
```

### CT\.KINESIS\.PR\.1 rule specification<a name="ct-kinesis-pr-1-rule"></a>

```
# ###################################
##       Rule Specification        ##
#####################################
# 
# Rule Identifier:
#   kinesis_stream_encrypted_check
# 
# Description:
#   This control checks whether Amazon Kinesis data streams are encrypted at rest with server-side encryption.
# 
# Reports on:
#   AWS::Kinesis::Stream
# 
# Evaluates:
#   AWS CloudFormation, AWS CloudFormation hook
# 
# Rule Parameters:
#   None
# 
# Scenarios:
#   Scenario: 1
#     Given: The input document is an AWS CloudFormation or AWS CloudFormation hook document
#       And: The input document does not contain any Kinesis stream resources
#      Then: SKIP
#   Scenario: 2
#     Given: The input document is an AWS CloudFormation or AWS CloudFormation hook document
#       And: The input document contains an Kinesis stream resource
#       And: 'StreamEncryption' has not been provided
#      Then: FAIL
#   Scenario: 3
#     Given: The input document is an AWS CloudFormation or AWS CloudFormation hook document
#       And: The input document contains an Kinesis stream resource
#       And: 'StreamEncryption' has been provided
#       And: 'StreamEncryption.EncryptionType' has not been provided or provided as an empty string
#       And: 'StreamEncryption.KeyId' has not been provided or provided as an empty string or invalid local reference
#      Then: FAIL
#   Scenario: 4
#     Given: The input document is an AWS CloudFormation or AWS CloudFormation hook document
#       And: The input document contains an Kinesis stream resource
#       And: 'StreamEncryption' has been provided
#       And: 'StreamEncryption.EncryptionType' has been provided as a non-empty string
#       And: 'StreamEncryption.KeyId' has not been provided or provided as an empty string or invalid local reference
#      Then: FAIL
#   Scenario: 5
#     Given: The input document is an AWS CloudFormation or AWS CloudFormation hook document
#       And: The input document contains an Kinesis stream resource
#       And: 'StreamEncryption' has been provided
#       And: 'StreamEncryption.EncryptionType' has not been provided or provided as an empty string
#       And: 'StreamEncryption.KeyId' has been provided as a non-empty string or valid local reference
#      Then: FAIL
#   Scenario: 6
#     Given: The input document is an AWS CloudFormation or AWS CloudFormation hook document
#       And: The input document contains an Kinesis stream resource
#       And: 'StreamEncryption' has been provided
#       And: 'StreamEncryption.EncryptionType' has been provided as a non-empty string
#       And: 'StreamEncryption.KeyId' has been provided as a non-empty string or valid local reference
#      Then: PASS

#
# Constants
#
let KINESIS_STREAM_TYPE = "AWS::Kinesis::Stream"
let INPUT_DOCUMENT = this

#
# Assignments
#
let kinesis_streams = Resources.*[ Type == %KINESIS_STREAM_TYPE ]

#
# Primary Rules
#
rule kinesis_stream_encrypted_check when is_cfn_template(%INPUT_DOCUMENT)
                                         %kinesis_streams not empty {
    check(%kinesis_streams.Properties)
        <<
        [CT.KINESIS.PR.1]: Require any Amazon Kinesis data stream to have encryption at rest configured
            [FIX]: Specify a 'StreamEncryption' configuration, with 'EncryptionType' set to 'KMS' and 'KeyId' set to an AWS KMS key identifier.
        >>
}

rule kinesis_stream_encrypted_check when is_cfn_hook(%INPUT_DOCUMENT, %KINESIS_STREAM_TYPE) {
    check(%INPUT_DOCUMENT.%KINESIS_STREAM_TYPE.resourceProperties)
        <<
        [CT.KINESIS.PR.1]: Require any Amazon Kinesis data stream to have encryption at rest configured
            [FIX]: Specify a 'StreamEncryption' configuration, with 'EncryptionType' set to 'KMS' and 'KeyId' set to an AWS KMS key identifier.
        >>
}

#
# Parameterized Rules
#
rule check(kinesis_stream) {
    %kinesis_stream {
        # Scenario 2
        StreamEncryption exists
        StreamEncryption is_struct

        StreamEncryption {
            # Scenario 3
            EncryptionType exists
            KeyId exists

            # Scenario 4, 5 and 6
            check_is_string_and_not_empty(EncryptionType)
            check_is_string_and_not_empty(KeyId) or
            check_local_references(%INPUT_DOCUMENT, KeyId, "AWS::KMS::Key") or
            check_local_references(%INPUT_DOCUMENT, KeyId, "AWS::KMS::Alias")
        }
    }
}

#
# Utility Rules
#
rule is_cfn_template(doc) {
    %doc {
        AWSTemplateFormatVersion exists or
        Resources exists
    }
}

rule is_cfn_hook(doc, RESOURCE_TYPE) {
    %doc.%RESOURCE_TYPE.resourceProperties exists
}

rule check_is_string_and_not_empty(value) {
    %value {
        this is_string
        this != /\A\s*\z/
    }
}

rule check_local_references(doc, reference_properties, referenced_resource_type) {
    %reference_properties {
        'Fn::GetAtt' {
            query_for_resource(%doc, this[0], %referenced_resource_type)
                <<Local Stack reference was invalid>>
        } or Ref {
            query_for_resource(%doc, this, %referenced_resource_type)
                <<Local Stack reference was invalid>>
        }
    }
}

rule query_for_resource(doc, resource_key, referenced_resource_type) {
    let referenced_resource = %doc.Resources[ keys == %resource_key ]
    %referenced_resource not empty
    %referenced_resource {
        Type == %referenced_resource_type
    }
}
```

### CT\.KINESIS\.PR\.1 example templates<a name="ct-kinesis-pr-1-templates"></a>

You can view examples of the PASS and FAIL test artifacts for the AWS Control Tower proactive controls\.

PASS Example \- Use this template to verify a compliant resource creation\.

```
Resources:
  KMSKey:
    Type: AWS::KMS::Key
    Properties:
      PendingWindowInDays: 7
      KeyPolicy:
        Version: 2012-10-17
        Id: example-key-policy
        Statement:
        - Sid: Enable IAM User Permissions
          Effect: Allow
          Principal:
            AWS:
              Fn::Sub: arn:${AWS::Partition}:iam::${AWS::AccountId}:root
          Action: kms:*
          Resource: '*'
      KeySpec: SYMMETRIC_DEFAULT
  KinesisStream:
    Type: AWS::Kinesis::Stream
    Properties:
      RetentionPeriodHours: 168
      ShardCount: 3
      StreamEncryption:
        EncryptionType: KMS
        KeyId:
          Ref: KMSKey
```

FAIL Example \- Use this template to verify that the control prevents non\-compliant resource creation\.

```
Resources:
  KinesisStream:
    Type: AWS::Kinesis::Stream
    Properties:
      RetentionPeriodHours: 168
      ShardCount: 3
```