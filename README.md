# POIS Reference Server
## Release Notes
| Date       | Version | Update Notes |
|------------|---------|--------------|
| 2022-09-30 | 1.0.27  | Initial release containing support for ad rules only. API configuration only - See API section for details
| 2023-03-17 | 1.1.0 | * Added support for time signal outputs<br />* Added support for segmentation descriptor prioritization<br />* Added logic for state lock
| 2026-04-28 | 1.2.0 | * Added virtual input switching feature<br />* Added Web UI for channel management<br />* Virtual input switch rules evaluated before normal SCTE-35 rules

## Overview
This is a POIS reference server capable of interacting with a transcoder using ESAM protocol.

It is intended to be used for testing purposes only.

## Functionality
The functionality includes:
* SCTE35 filtering (deleting unwanted, passing through noop):
    * UPID value
    * segmentation type id (program,network, provider advertisement id,chapter start/stop)
    * splice command type (5,6)
    * UPID type


* Modify / Replace (free text entry with parameter validation)
    * splice command type 5 → 6 *Will be supported at a later date*
    * modify certain characteristics:
        * example: web_delivery_not_allowed
            * be aware that this may add more metadata to payload that needs to be taken into consideration
        * Adding/modifying segmentation_upid value
* SCTE35 driven archives *Will be supported at a later date, integrating with AWS Elemental Live*
    * Utilizing the Elemental Live API (requires public facing Elemental Live IP or NAT proxy)
* SCTE35 input switching *Will be supported at a later date, integrating with AWS Elemental Live*
    * Synchronous using inbound SCTE35 messages (Source switch logged in schedule stored in database)
* Virtual Input Switching (SCTE-35 triggered)
    * Configure trigger conditions based on any SCTE-35 property (e.g. segmentation_type_id)
    * Returns noop action with AlternateContent element containing the target encoder input name
    * Evaluated before normal SCTE-35 conditioning rules
    * Supports per-channel configuration of multiple virtual input switch rules

## Architecture
The architecture consists of several AWS services and is deployed using a CloudFormation template.

![](Architecture-pois-ref-server.png?width=60pc&classes=border,shadow)

For OTT workflows, AWS Elemental Live needs to contribute the ESAM conditioned stream to AWS. Then through MediaConnect, MediaLive and MediaPackage/MediaStore can the OTT package be generated.

![](Architecture-pois-ref-server-aws-video.png?width=60pc&classes=border,shadow)

## How to deploy
Use [this CloudFormation template](pois-ref-server.yaml) to deploy the solution.. Please note, the only resources deployed in this solution pertain to the POIS. Deployed Resources include:

* AWS Lambda Functions x 2
* AWS Lambda Layers * 2
* Amazon API Gateway REST API
* DynamoDB Channels Database
* DynamoDB Schedule Database
* DynamoDB State Database
* Amazon S3 ****bucket deployed to host UI ... later phase*
* AWS IAM Role & Policy creation for Lambda Functions

## Web UI

The POIS includes a web-based dashboard (`ui.html`) for managing channels, SCTE-35 rules, and virtual input switch configuration.

### Loading the UI after CloudFormation deployment

After deploying the CloudFormation stack, the UI is automatically uploaded to the S3 bucket created by the stack. You can find the URL in the CloudFormation **Outputs** tab under the **POISControlUI** key. It will look like:

```
https://<bucket-name>.s3.<region>.amazonaws.com/<path>/ui.html
```

Open that URL in your browser to access the dashboard.

### Loading the UI locally

You can also open `ui.html` directly from your local filesystem — no web server required. Simply open the file in any modern browser:

- **macOS**: `open ui.html` or double-click the file in Finder
- **Windows**: double-click `ui.html` in File Explorer
- **Linux**: `xdg-open ui.html` or open it from your file manager

### Connecting to your POIS API

Once the UI is loaded:

1. Enter your **ChannelConfigurationAPI** endpoint in the "API Endpoint" field at the top of the page. This is the value from the CloudFormation Outputs tab and looks like:
   ```
   https://<api-id>.execute-api.<region>.amazonaws.com/<stage>/pois
   ```
2. Click **Connect** to load all configured channels
3. Select a channel from the list to view or edit its configuration
4. Use the tabs to manage **General** settings, **SCTE-35 Rules**, and **Virtual Input Switch** rules
5. Click **Save Channel** to persist changes, or **+ New Channel** to create a new one

| Note: If you are running the UI locally and connecting to an API Gateway endpoint, make sure CORS is enabled on your API Gateway or you may see connection errors in the browser console. |
|----------|

## API Control
At initial release, the only option to configure the POIS is via API. Once you've deployed the CloudFormation stack, check the **Outputs** tab. This will contain your API Endpoints.

![](cloudformation-output.png?width=60pc&classes=border,shadow)

The two most important output values are the **ESAMEndpoint** which you will configure in your encoder, and the **ChannelConfigurationAPI** value.  This is the API endpoint you need to use to start setting up the POIS.

| Method  | Path             | Description |
|---------|------------------|-------------|
| GET     | /pois/channels  | Returns configuration of all channels in the POIS |
| GET     | /pois/channels/{channel-name} | Where {channel-name} is the name of your channel, this will return the configuration of the channel |
| PUT     | /pois/channels/{channel-name} | Where {channel-name} is the name of your channel, this will create/update the channel configuration in the POIS |
| DELETE  | /pois/channels/{channel-name} | Where {channel-name} is the name of your channel, this will delete the channel from the POIS database

You can download the POSTman collection for this API set here: https://www.getpostman.com/collections/f91925ceece614b8486e

* When creating a channel in the API, only 2 properties are required:
    - default_behavior = this can either be **noop** or **delete**
    - esam_version = currently this value has to be **2013**

## Creating SCTE35 Rules
The POIS offers some rudimentary capabilities for SCTE35 signal conditioning and deleting. Here are some tips when configuring your rules:
* The rules are processed in order, starting with the first rule in the list submitted via the API
* The rule format is as follows:
  - You first specify an action type. What do you want to happen if the rule evaluates to TRUE (supported actions: replace, delete)
  - Then comes the condition; what do you want to evaluate. Currently, the application supports simple checks, ie. splice_command_type = 5 (splice_command_type is the **property**, 5 is the **value**, and the operator is **=**)
    - It is possibly to use multiple comma separated values that will be evaluated in order. ie. "value":"32,48,52"
  - For **replace** actions, you also need to submit with the json a **replace_params** list. If the evaluation of the rule is True, then the SCTE properties listed in the **replace_params** list will all be used to modify the SCTE35 binary  
* You can mix and match action types in your rules. IE. you can have a DELETE rule to delete all splice_command_type 5 signals, then have a subsequent rule to modify duration  
* It's recommended that any DELETE rules should be at earlier indexes in list than REPLACE rules
* There is currently no checking to see if rules you submit contradict each other, so please validate this before you configure the channel or you may not get the desired results
* For major replace actions, ie. if splice_command_type = 5, modify to splice_command_type = 6, the code does not fill in any blanks, so make sure your **replace_params** list contains all the new properties you want included in the new SCTE35 binary
* The code supports multiple operators:
  - '=' equal to
  - '>' greater than
  - '<' less than
  - '-' range (ie. if duration is between 15-30). When the operator is range, the **value** MUST have a min integer value, and a max integer value, separated by a hyphen '-'. For example value="2-4"
  - '!=' not equal to (this is useful for filtering/deleting anything that isn't a specific SCTE35 type. ie. if segmentation_upid_type != "9")
* Mode, this is optional and accepts values: "stateful" or "stateless"
  - stateful. The POIS will write to a DB for a rule it has matched with a replace action. Any subsequent SCTE signal received in the active window of the 'replaced' SCTE signal will be **deleted**
  - stateless. Each ESAM decision will be made independently
* Descriptor Priority, this is in the case of receiving a SCTE35 with multiple segmentation descriptors. Setting a comma delimited priority for which descriptor settings to preserve. ie. "32,34,48"

| Note: If all the rules evaluate to FALSE, the configured **default_behavior** action will apply to the ESAM response |
|----------|

Here's a sample channel configuration:

```
{
  "default_behavior": "noop",
  "esam_version": "2013",
  "descriptor_priority":"",
  "mode":"stateless",
  "rules": [
    {
      "type": "delete",
      "condition":{
      "property": "splice_command_type",
      "value": "5",
      "operator": "="
      }
    },
    {
      "type": "replace",
      "condition":{
      "property": "duration",
      "value": "30",
      "operator": "<"
      },
      "replace_params":[
      {"duration":"60"},
      {"avail_expected":"2"}
      ]
    }
  ]
}
```

The channel has 2 rules; rule 1 is a delete rule that will evaluate to true if the incoming SCTE35 is a splice insert (splice_command_type=5). Rule 2 is evaluating the value of the break_duration property in the incoming SCTE35, and if the condition evaluates to true there are 2 replace_param properties that will override the corresponding properties of the incoming SCTE35 values



## Virtual Input Switching

The POIS supports virtual input switching triggered by SCTE-35 signals. When an incoming signal matches a configured trigger condition, the POIS responds with a `SignalProcessingNotification` containing `action="noop"` and an `AlternateContent` element that directs the encoder to switch to a different input source.

### How It Works

1. Virtual input switch rules are evaluated **before** normal SCTE-35 conditioning rules
2. If a match is found, the response includes an `AlternateContent` element with:
   - `altContent="true"` - indicates this is an alternate content directive
   - `altContentIdentity` - the target encoder input name to switch to
   - `zoneIdentity` - preserved from the incoming signal
3. The original signal properties (`acquisitionPointIdentity`, `acquisitionSignalID`, `acquisitionTime`, `UTCPoint`, SCTE-35 binary) are preserved in the response
4. If no virtual input switch rule matches, normal rule evaluation proceeds

### Configuration

Add `virtual_input_switch_rules` to your channel configuration:

```json
{
  "default_behavior": "noop",
  "esam_version": "2016",
  "virtual_input_switch_rules": [
    {
      "trigger_condition": {
        "property": "segmentation_type_id",
        "value": "1",
        "operator": "="
      },
      "alt_content_identity": "backup_encoder_input"
    },
    {
      "trigger_condition": {
        "property": "segmentation_type_id",
        "value": "48,49",
        "operator": "="
      },
      "alt_content_identity": "slate_input"
    }
  ]
}
```

### Rule Format

Each virtual input switch rule requires:
- **trigger_condition**: An object with:
  - `property` - Any valid SCTE-35 property (e.g. `segmentation_type_id`, `splice_command_type`)
  - `value` - The value to match against (supports comma-separated values for OR logic)
  - `operator` - (optional, defaults to `=`) One of: `=`, `>`, `<`, `-`, `!=`
- **alt_content_identity**: The target encoder input name to switch to when the trigger matches

### Example Response

When a virtual input switch rule matches, the POIS returns:

```xml
<?xml version="1.0" encoding="utf-8"?>
<SignalProcessingNotification xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:sig="urn:cablelabs:md:xsd:signaling:3.0"
    xmlns:core="urn:cablelabs:md:xsd:core:3.0"
    xmlns="urn:cablelabs:iptvservices:esam:xsd:common:1">
  <ResponseSignal action="noop"
      acquisitionPointIdentity="channel1"
      acquisitionSignalID="ffcd7def-bbd7-e7b0-ffff-5de975c726d0"
      acquisitionTime="2021-09-11T03:39:48.337Z">
    <sig:UTCPoint utcPoint="2021-09-11T03:39:48.337Z"/>
    <sig:BinaryData signalType="SCTE35">/DAlAAAAAsrYAP/wFAUAAAABf+/+ACjJaP4AFJlwAAEBAQAA/XeB3g==</sig:BinaryData>
    <AlternateContent altContent="true" altContentIdentity="backup_encoder_input" zoneIdentity="zone1"/>
  </ResponseSignal>
  <StatusCode classCode="0">
    <core:Note>Virtual input switch triggered, target: backup_encoder_input</core:Note>
  </StatusCode>
</SignalProcessingNotification>
```

### Use Cases

- **Content Identification triggers**: Configure `segmentation_type_id = 1` (Content Identification) to trigger an input switch when content identification markers are received
- **Program boundary switching**: Use `segmentation_type_id = 48` or `49` (Provider Ad Start/End) to switch inputs at program boundaries
- **Emergency switching**: Configure a specific UPID or segmentation type to trigger failover to a backup input

### Web UI

The Web UI (ui.html) provides a visual interface for managing virtual input switch rules under the "Virtual Input Switch" tab when editing a channel. You can:
- Add/remove virtual input switch rules
- Configure trigger conditions (property, operator, value)
- Set the target altContentIdentity for each rule
- View the current configuration in a summary table
