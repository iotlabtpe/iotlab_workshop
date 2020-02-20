---
layout: default
categories: [lab]
tags: [setup]
excerpt_separator: <!--more-->
permalink: /lab/lab-3
name: /lab/lab-3.html
---
# Lab3 - User Credential approach 1 Lab

In this lab, we will create a demo to provision personal user credential and Wi-Fi credential to device by android app once the device is powered at first time.

## Solution Architecture

The solution architect would be separated into two part: (1) Get User Credential by Android app. (2) Android app provision the user credential to device.

(1)  Get User Credential by Android app：
![Device_certificate_creation.png](../pics/user_provision_approach/Device_certificate_creation.png) (2) Android app provision the user credential to device.
![Android_app_provision.png](../pics/user_provision_approach/Android_app_provision.png)

## Prerequisite

1. This demo take Ameba Z2 as a reference board, which have to build code by IAR.
2. An android Phone with Android 6 at least to support the AWS android SDK.
3. Download [Android Studio](https://developer.android.com/studio) and Android app code from:
4. Download Amazon FreeRTOS code which support approach 1 from:

## Step 1. - Use AWS Cloudformation to deploy your AWS Service configuration related to this demo.

Here’s the tutorial for setting up AWS Service including AWS Cognito, AWS API Gateway, Lambda, Dynamo DB.

1. Setup **AWS Cognito**
    1. Choose “Mange User Pools”.
    ![AWS_Cognito.png](../pics/user_provision_approach/AWS_Cognito.png)
    2. Choose “Create a user pool”
    ![Create_User_Pool.png](../pics/user_provision_approach/Create_User_Pool.png)
    3. Fill in Pool name“IoTProvisionUserPool”, and then choose “Review Defaults“.
    ![Pool_Name.png](../pics/user_provision_approach/Pool_Name.png)
    4. Setup app client
    ![UserPool_AppClient.png](../pics/user_provision_approach/UserPool_AppClient.png)
        1. Edit App client name “IoTProvisionApp”.
        2. check “Enable username password auth for admin APIs for authentication (ALLOW_ADMIN_USER_PASSWORD_AUTH)“.
        ![UserPool_AppClient_Name.png](../pics/user_provision_approach/UserPool_AppClient_Name.png)
    5. choose Create pool
       ![UserPool_CreatePool.png](../pics/user_provision_approach/UserPool_CreatePool.png)

2. Setup **Lambda**
    1. Create Function, Function name: "IoTProvision_Lambda"
       ![Lambda_setting_1.png](../pics/user_provision_approach/Lambda_setting_1.png)
    2. copy the following code to replace the code, lambda_function.py
       ![Lambda_setting_2.png](../pics/user_provision_approach/Lambda_setting_2.png)

        ```python
        import boto3
        import json
        import logging

        logger = logging.getLogger()
        logger.setLevel(logging.INFO)

        iot = boto3.client('iot')
        dynamodb = boto3.client('dynamodb')
        table = 'DSN'

        # Create a policy
        policyDocument = {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Action": "iot:*",
              "Resource": "*"
            }
          ]
        }

        def delete_cert_and_policy(deviceId, principals):
            for principal in principals:
                certificateId = principal.split('/')[-1]
                policies = iot.list_attached_policies(target=principal)
                for policy in policies['policies']:
                    iot.detach_policy(
                        policyName=policy['policyName'], target=principal
                    )
                    iot.delete_policy(policyName=policy['policyName'])
                iot.update_certificate(
                    certificateId=certificateId, newStatus='INACTIVE'
                )
        #        iot.detach_thing_principal(
        #            thingName=deviceId,
        #            principal=principal
        #        )
        #        wait_for_detach_finish(deviceID, principal)
        #        iot.delete_certificate(
        #            certificateId=certificateId, forceDelete=True
        #        )

        def lambda_handler(event, context):
            logger.info("request: " + json.dumps(event));
            # check device serial number is valid
            deviceId = event['params']['querystring']['SN']
            deviceItem = dynamodb.get_item(TableName=table, Key={'sn': {'S': deviceId}})
            if not deviceItem.get('Item'):
                return {
                    'statusCode': 403,
                    'body': {'msg': 'Invalid device serial number'}
                }

            # create thing
            try:
                iot.describe_thing(
                    thingName=deviceId
                )
            except iot.exceptions.ResourceNotFoundException:
                response = iot.create_thing(
                    thingName=deviceId
                )

            # delete certificates which are attached to this thing
            response = iot.list_thing_principals(
                thingName=deviceId
            )
            if response['principals']:
                delete_cert_and_policy(deviceId, response['principals'])

            # generate keys and certificate
            response = iot.create_keys_and_certificate(setAsActive=True)
            certificateArn = response['certificateArn']
            certificatePem = response['certificatePem']
            privateKey = response['keyPair']['PrivateKey']

            # attach certificate to thing
            iot.attach_thing_principal(
                thingName=deviceId,
                principal=certificateArn
            )

            # create a policy for thing
            policyName = 'Policy_' + deviceId
            iot.create_policy(
                policyName=policyName,
                policyDocument=json.dumps(policyDocument)
            )

            # attach policy to certificate
            iot.attach_policy(
                policyName=policyName,
                target=certificateArn
            )

            return {
                'statusCode': 200,
                'body': {
                    'certificatePem': certificatePem,
                    'privateKey': privateKey
                }
            }
        ```

    3. Go to Execution role to modify policy.
       ![Lambda_policy_1.png](../pics/user_provision_approach/Lambda_policy_1.png)
       Add policy: AmazonDynamoDBFullAccess, AmazonAPIGatewayAdministrator, AWSIoTFullAccess
       ![Lambda_policy_2.png](../pics/user_provision_approach/Lambda_policy_2.png)

3. Setup **AWS API Gateway**
    1. Create API, and then choose REST API.
       ![Create_API.png](../pics/user_provision_approach/Create_API.png)
    2. Choose NewAPI and fill in API name “IoTProvision_API”, then create.
        ![Create_API_Name.png](../pics/user_provision_approach/Create_API_Name.png)
        1. Now that, we can start configure the following settings with red rectangle, then back to setup Resource method.

        ![API_configuration.png](../pics/user_provision_approach/API_configuration.png)
    3. Authorizers configuration
        1. Name: “IoTProvision_Authorizer” 
        2. Select Cognito
        3. Cognito User Pool “IoTProvisionUserPool”
        4. Token Source “Authorization”
        5. Create
        ![API_Authorizer.png](../pics/user_provision_approach/API_Authorizer.png)
    4. Model configuration
        1. Create method used by API Method.
            1. Name: “CertReplay“
            2. Content type: “applicaiton/json”
            3. Fill in Model schema.
            ![API_Model.png](../pics/user_provision_approach/API_Model.png)

            ```json
            {
              "$schema": "http://json-schema.org/draft-04/schema#",
              "title": "IoTProvisionInputModel",
              "type": "object",
              "properties": {
                "certificatePem": { "type": "string" },
                "privateKey": { "type": "string" }
              }
            }
            ```

    5. Resource configuration
        1. Resource Name “cert”
        2. Create Resource
            ![API_Resource_Create.png](../pics/user_provision_approach/API_Resource_Create.png)
        3. Action → Create Method.
            1. Select “GET”
            ![API_Resorce_create_method.png](../pics/user_provision_approach/API_Resorce_create_method.png)
        4. The method setup
            ![API_Model_setup.png](../pics/user_provision_approach/API_Model_setup.png)
            1. Integration type → Lambda function.
            2. Lambda function → IoTProvision_Lambda
            3. Then save.
        5. Configure GET - Method Execution
            ![API_Get_method_execution.png](../pics/user_provision_approach/API_Get_method_execution.png)
            1. Configure Method Request:
                Authorization:"IoTProvision_Authorizer"
                URL QRL String Parameters, Name: "sn", then check Required.
                HTTP Request Headers, Name: "Authorization", then check Required.
               ![API_Method_Request.png](../pics/user_provision_approach/API_Method_Request.png)
            2. Configure Integration Request
               ![API_Integration_Request_1.png](../pics/user_provision_approach/API_Integration_Request_1.png)
                1. application/json

                    ```json
                    #set($inputRoot = $input.path('$'))
                    {
                      "params": {
                        "querystring": {
                          "SN": "$input.params('sn')"
                        }
                      }
                    }
                    ```

            3. Configure Integration Response
                ![API_Integration_Response.png](../pics/user_provision_approach/API_Integration_Response.png)
                1. application/json

                    ```json
                    #set($inputRoot = $input.path('$'))
                    {
                      "certificatePem" : $input.json('$.body.certificatePem'),
                      "privateKey" : $input.json('$.body.privateKey')
                    }
                    ```

            4. Configure Method Response
                Configure Response Body for 200, Contect type:"application/json", Models:"CertReply".
                ![API_Method_Response.png](../pics/user_provision_approach/API_Method_Response.png)
    6. Deploy API to generate endpoint
        ![API_Deploy.png](../pics/user_provision_approach/API_Deploy.png)
        ![API_Deploy_stage_name.png](../pics/user_provision_approach/API_Deploy_stage_name.png)
    7. Check endpoint in Stages
        ![API_Stage_endpoint.png](../pics/user_provision_approach/API_Stage_endpoint.png)

4. Setup **Dynamo DB**
    1. Create Table:
        1. Table name: DSN
        2. Primary key: sn
       ![DynamoDB_tableName.png](../pics/user_provision_approach/DynamoDB_tableName.png)
    2. create your own serial number item, then save it.
       ![DynamoDB_create_item.png](../pics/user_provision_approach/DynamoDB_create_item.png)
       ![DynamoDB_item_content.png](../pics/user_provision_approach/DynamoDB_item_content.png)

## Step 2. - Configure, build, and install the android app

1. Fill in app/src/main/java/com/amazonaws/youruserpools/AppHelper.java with the userPoolId, clientId, and clientSecret.
    1. check these information in your AWS Cognito Service:
       Get User Pool ID from Pool ID.
       ![Cognito_general_setting.png](../pics/user_provision_approach/Cognito_general_setting.png)
       Get client id and client secret.
       ![info_in_cognito.png](../pics/user_provision_approach/info_in_cognito.png)
    2. Fill in these information at this location:
       ![cognito_info_in_code.png](../pics/user_provision_approach/cognito_info_in_code.png)
2. Fill in your API Gateway endpoint in app\src\main\java\com\amazonaws\youruserpools\IoTProvisionAPIClient.java
    ![Android_app_config_api_endpoint.png](../pics/user_provision_approach/Android_app_config_api_endpoint.png)
3. Build and app and install it to the Android Phone, then register your account by AWS Cognito.

**Notice: If you run into build error because of sdk, please install the sdk by clicking "install missing sdk package(s)".**
![Android_app_install_sdk.png](../pics/user_provision_approach/Android_app_install_sdk.png)

## Step 3. - Build and Run the Amazon FreeRTOS approach 1 Project

1. Fill in Soft AP SSID and password in amazon-freertos\demos\realtek\amebaz2\common\config_files\aws_wifi_config.h
![SoftAPSSID_in_code.png](../pics/user_provision_approach/SoftAPSSID_in_code.png)
2. Fill in your endpoint information in amazon-freertos\demos\common\include\aws_clientcredential.h
![endpoint_in_code.png](../pics/user_provision_approach/endpoint_in_code.png)
3. Before flash the device, please earse the memory first.
![IAR_erase_memory.png](../pics/user_provision_approach/IAR_erase_memory.png)
4. Build and flash to the device by IAR IDE, then reset the device to let it enter SoftAP mode.
![IAR_build.png](../pics/user_provision_approach/IAR_build.png)

**Notice: You could take use of the following link to install USB UART driver to check system log on UART port.**
https://www.ftdichip.com/Drivers/CDM/CDM21228_Setup.zip

## Step 4. - Use the android app to get the user credential from AWS Service

1. Sign up your account in the android app.
2. Log in the App.

    ![cognito_login.png](../pics/user_provision_approach/cognito_login.png)

3. Go to “cert Provision” page and use the UID (Device ID) to get the user credential from AWS Service (push the RE-GENKEY).
    ![goto_certprovision_page.png](../pics/user_provision_approach/goto_certprovision_page.png)

    ![step1_page.png](../pics/user_provision_approach/step1_page.png)

## Step 5. - Go to STEP 2 Page to fill in the Wi-Fi credential information which you want the device connect to, then push the “SETUP WIFI CONFIG“

![step2_page.png](../pics/user_provision_approach/step2_page.png)

## Step 6. - Reset the device to start to the Soft AP mode of the device, then use android app setting to connect to it

Use the following button to reset the device.
![device_reset_button.png](../pics/user_provision_approach/device_reset_button.png)
Then it will start Soft AP mode with previous Soft AP setting in Step 3, use android default app to connect to it.
![wifi_setting.jpg](../pics/user_provision_approach/wifi_setting.jpg)

## Step 7. - Go to page 3 (app STEP 3 page) to push the button to provision the user credential

Use the button “CONNECT TO PROVISION” to connect to Soft AP, it will start provisioning once it connect to the Soft AP.
![step3_page.jpg](../pics/user_provision_approach/step3_page.jpg)

There will be a pop-up a message to show the provision result.

![step3_page_reply.jpg](../pics/user_provision_approach/step3_page_reply.jpg)

**Notice: There is a timeout in device soft AP started after android app connecting to it. If time’s up with no provision, then you need to push the reset button to restart the soft AP process. You can configure this timeout for your customization here:**
![app_device_connection_timeout.png](../pics/user_provision_approach/app_device_connection_timeout.png)

## Step 8. - Check the user credential work or not. (By AWS IoT Core Test)

Resetting the device to let it connect to designated  WiFi SSID and send publish message to IoT Core. You can check MQTT log in AWS IoT Core.

![AWS_IoTCore_Test.png](../pics/user_provision_approach/AWS_IoTCore_Test.png)
