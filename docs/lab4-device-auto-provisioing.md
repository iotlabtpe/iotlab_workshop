---
layout: default
categories: [lab]
tags: [setup]
excerpt_separator: <!--more-->
permalink: /lab/lab-4
name: /lab/lab-4.html
---

# Just-In-Time Registration (JITR) - Lab 4

In this Lab, you will generate JITR approach 2 related configuration and test the follow of JITR approach 2.

## Prerequisites

1. AWS Cloud9 environment and install the building package as Lab0 indicates.
2. Clone the sample package from [aws-iot-device-auto-provisioning-approach](aws-iot-device-auto-provisioning-approach).

## Architecture

For IoT devices onboarding, JITR approach 2 offers a method that devices could generate device certificate and key automatically. Through IoT Core, rule engine and DynamoDB, it becomes more secure in management and more efficient in certificate/key rotation.

![p0_lab4.png](../pics/lab4/p0_lab4.png)

1. Device generates key and certificate based on on-boarded long term CA certificate. This CA should be stored in the secure partition of device.
2. Device connects to AWS IoT Core with the generated key and certificate.
3. Trigger IoT Core Action.
4. Rule engine triggers Lambda.
5. Verify device information with AWS DynamoDB.
6. Lambda creates thing and policy for device.
7. Device connects to AWS IoT Core with the registered key and certificate.

## Generating CA Certificate and Registering to IoT Core

1. Use the following command to generate intermediate CA certificate.

    ```bash
    cd ~/environment
    mkdir CAs
    cd CAs/
    openssl genrsa -out CA_Private.key 2048
    openssl req -x509 -new -nodes -key CA_Private.key -sha256 -days 365 -out CA_Certificate.pem
    ```

2. Use the following AWS CLI command to acquire a registration code. This command will return a randomly generated, unique registration code that is bound to your AWS account. It does not expire until you delete it. Remember it and it will be needed in the coming step.

    ```bash
    aws iot get-registration-code
    ```

3. Use the registration code to create a CSR by following command.

    ```bash
    openssl genrsa -out Verification_Private.key 2048
    openssl req -new -key Verification_Private.key -out Verification.csr
    ```

4. During the CSR creation process, you will be prompted for information. Enter the registration code you created above into the **Common Name** field of the verification

    ![p1_lab4.png](../pics/lab4/p1_lab4.png)
5. Use CSR that includes the registration code and the first sample CA certificate to generate a new certificate by the following command.

    ```bash
    openssl x509 -req -in Verification.csr -CA CA_Certificate.pem -CAkey CA_Private.key -CAcreateserial -out Verification.crt -days 365 -sha256
    ```

6. Use the verification certificate to register your sample CA certificate by the following AWS CLI command. This command also enables CA and allows auto-registration. This command will returns CA certificate ID that you would use in the following step.

    ```bash
    aws iot register-ca-certificate --ca-certificate file://CA_Certificate.pem --verification-certificate file://Verification.crt --set-as-active --allow-auto-registration
    ```

7. You can check it from IoT Core console. Please login to IoT Core console and click **Secure**→**CAs**. You should see the CA certificate ID you just created and registered.

    ![p2_lab4.png](../pics/lab4/p2_lab4.png)

## Configure DynamoDB

In this lab, we use device Wi-Fi MAC address as device serial number (DSN). To verify it, we need to create a database in AWS DynamoDB. When device generates the certificate and key, they will be used to connect to IoT core via MQTT over TLS. IoT Core will trigger rule engine and pass device information to a Lambda function. Lambda will verify this information with DynamoDB. If it is valid, Lambda function will create a thing name and policy for this device.

1. Login to AWS console and search **DynamoDB**.

    ![p3_lab4.png](../pics/lab4/p3_lab4.png)
2. Click **Create table**.

    ![p4_lab4.png](../pics/lab4/p4_lab4.png)
3. In this lab, giving table a name **jitr** in **Table name** and giving a primary key **dsn** in **Primary key**. The type of **Primary key** is **String**, then click **Create**.

    ![p5_lab4.png](../pics/lab4/p5_lab4.png)
4. After creating table, browse to tab **Items** → **Create item**.

    ![p6_lab4.png](../pics/lab4/p6_lab4.png)
5. Fill **dsn** with your device MAC address and click **Save**.

    ![p7_lab4.png](../pics/lab4/p7_lab4.png)

## Configure Lambda

1. Clone the sample code for github. Modify the region definition in lambda_function.py and then pack it.

    ```bash
    cd ~/environment
    git clone https://github.com/aws-samples/aws-iot-device-auto-provisioning-approach.git
    cd aws-iot-device-auto-provisioning-approach/lambda/
    vim lambda_function.py
    pip install -t ./ -r requirements.txt
    zip -r jitrSampleFunction.zip * .[^.]*
    ```

2. Download jitrSampleFunction.zip from AWS Cloud9.

3. Login to AWS management console, search and click **Lambda**.

    ![p8_lab4.png](../pics/lab4/p8_lab4.png)
4. In Lambda console, click **Create function**.

    ![p9_lab4.png](../pics/lab4/p9_lab4.png)
5. Click **Author from scratch**, giving a function name **jitrDemoFunction**. Select **Python 3.6** and **Create a new role with basic Lambda permissions** in **Choose or create an execution role**. Click **Create function**.

    ![p10_lab4.png](../pics/lab4/p10_lab4.png)
6. In **Function code**, choose **Upload a .zip file** then click **Upload** to upload sample code you downloaded.

    ![p11_lab4.png](../pics/lab4/p11_lab4.png)
7. Roll down the page to **Execution role** and click **View the jitrDemoFuntion-role-xxxxx** role on the IAM console.

    ![p12_lab4.png](../pics/lab4/p12_lab4.png)
8. In IAM console, you would see the role you just created. Click **Edit policy** and paste the following policies. After pasting this policies, click **Review policy** and then click **Save changes**.

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                ],
                "Resource": "arn:aws:logs:*:*:*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "iot:DetachThingPrincipal",
                    "iot:CreateThing",
                    "iot:DeleteThing",
                    "iot:DetachPolicy",
                    "iot:AttachThingPrincipal",
                    "iot:DeleteCertificate",
                    "iot:AttachPolicy",
                    "iot:AttachPrincipalPolicy",
                    "iot:DescribeThing",
                    "iot:CreatePolicy",
                    "iot:DescribeCertificate",
                    "iot:ListAttachedPolicies",
                    "iot:DeletePolicy",
                    "iot:ListPrincipalPolicies",
                    "iot:DetachPrincipalPolicy",
                    "iot:ListThingPrincipals",
                    "iot:UpdateCertificate",
                    "iot:ListThings",
                    "dynamodb:Scan",
                    "dynamodb:BatchGetItem",
                    "dynamodb:Query",
                    "dynamodb:List*",
                    "dynamodb:Describe*",
                    "dynamodb:GetItem"
                ],
                "Resource": "*"
            }
        ]
    }
    ```

    ![p13_lab4.png](../pics/lab4/p13_lab4.png)
    ![p14_lab4.png](../pics/lab4/p14_lab4.png)
9. Switch back to Lambda console, click **Save**.
    ![p15_lab4.png](../pics/lab4/p15_lab4.png)

## Configure AWS IoT Rule

1. Login to AWS IoT console and click **Act**→ **Rules**→ **Create**.

    ![p16_lab4.png](../pics/lab4/p16_lab4.png)
2. Giving a name and paste the following code to **Rule query statement** where xxxxxxxxx is your CA certificate ID you generated above.

    ```text
    SELECT * FROM '$aws/events/certificates/registered/xxxxxxxxxxxxx'
    ```

    ![p17_lab4.png](../pics/lab4/p17_lab4.png)
3. Roll down and click **Add action** in **Set one or more actions**.
    ![p18_lab4.png](../pics/lab4/p18_lab4.png)
4. Select **Send a message to a Lambda function** and click **Configure action**.

    ![p19_lab4.png](../pics/lab4/p19_lab4.png)
5. Click **Select** and select the Lambda you created above. Click **Add action**→ **Create rule**

    ![p20_lab4.png](../pics/lab4/p20_lab4.png)
    ![p21_lab4.png](../pics/lab4/p21_lab4.png)

## Build, flash

1. Clone Amazon FreeRTOS and patch for Just-In-Time Registration demonstration.

    ```bash
    cd ~/environment/aws-iot-device-auto-provisioning-approach/amazon-freertos/
    git clone https://github.com/aws/amazon-freertos.git
    cd amazon-freertos/
    git checkout 201906.00_Major -b auto-provisioning
    git am ../0001-Add-IoT-Lab-Workshop-Device-Auto-Provision-sample-co.patch
    ```

2. Before build code, Download the CAs certificate, private key and tools folder from Cloud9. Use **PEMfileToCString.html** to generate C string. Please select the **CA_Certificate.pem** you just create above and click **Display formatted PEM stting to be copied into aws_clientdential_keys.h**. Paste it into filed **keyJITR_DEVICE_CERTIFICATE_AUTHORITY_PEM** in **~/environment/aws-iot-device-auto-provisioning-approach/amazon-freertos/amazon-freertos/demos/include/aws_clientcredential_keys.h**.

    ![p22_lab4.png](../pics/lab4/p22_lab4.png)
    ![p23_lab4.png](../pics/lab4/p23_lab4.png)
3. Please select the **CA_Private.key** you just create above and click **Display formatted PEM stting to be copied into aws_clientdential_keys.h**. Paste it into filed **keyJITR_DEVICE_CERTIFICATE_AUTHORITY_KEY_PEM** in **~/environment/aws-iot-device-auto-provisioning-approach/amazon-freertos/amazon-freertos/demos/include/aws_clientcredential_keys.h**.

    ![p24_lab4.png](../pics/lab4/p24_lab4.png)
4. Edit file **~/environment/aws-iot-device-auto-provisioning-approach/amazon-freertos/amazon-freertos/demos/include/aws_clientcredential.h**.
    For **clientcredentialMQTT_BROKER_ENDPOINT**, you can get this information in **IoT Core console**→ **Settings**.

    ![p25_lab4.png](../pics/lab4/p25_lab4.png)
5. For **clientcredentialWIFI_SSID** and **clientcredentialWIFI_PASSWORD**, you will get customer Wi-Fi information in the class. Please save once you done those modifications.
6. Change directory to **~/environment/aws-iot-device-auto-provisioning-approach/amazon-freertos/amazon-freertos/vendors/espressif/boards/esp32/aws_demos**, use the following command to build code.

    ```shell
    make all -j4
    ```

7. Download the image from AWS Cloud9.
    - build/aws_demo.bin
    - build/partition-table.bin
    - build/bootloader/bootloader.bin
8. Follow the command to flash and monitor device as Lab0 indicates.

## Verification and Test

1. The device will generate key and certificate at the first boot. You can check the log from console.

    ![p26_lab4.png](../pics/lab4/p26_lab4.png)
2. Login to AWS IoT Core console and click **Test**. Fill **Subscription topic** in **#** and click **Subscribe the topic**.

    ![p27_lab4.png](../pics/lab4/p27_lab4.png)

3. The device will use this pair to connect to AWS IoT Core. It would fail at first time and wait Lambda to verify with DynamoDB. After a few second, the device will try to connect to AWS IoT Core again. You can check the MQTT message sended by the device.

    ![p28_lab4.png](../pics/lab4/p28_lab4.png)

4. Login to AWS IoT Core console and click **Secure**→**Certificates**. You are supposed to see a new certificate generated from your device.

    ![p29_lab4.png](../pics/lab4/p29_lab4.png)
