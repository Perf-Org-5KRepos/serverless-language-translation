*Read this in other languages: [日本語](README-ja.md).*

[![Build Status](https://travis-ci.org/IBM/serverless-language-translation.svg?branch=master)](https://travis-ci.org/IBM/serverless-language-translation)

# Deploy a Serverless Multilingual Conference Room

In this code pattern, we will create the workings for a multilingual chat room using OpenWhisk, Watson Text to Speech and Watson Language Translator.  The MQTT messaging protocol is also leveraged, which allows each client to publish and subscribe to one or more channels.  This repository contains a series of serverless functions which are called in sequence determined by the channel to which a message is submitted.  

When the reader has completed this code pattern, they will understand how to:

* Deploy Cloud Function actions and triggers
* Interact with the Watson IoT platform
* Set up a CloudFoundry application

![Architecture](/doc/source/images/architecture.png)

## Flow

1. Message received from a client, which can be a web browser, CLI, OpenWhisk action, SMS text, etc.
2. If message payload contains an audio file, it is transcribed to text.
3. Transcribed text is translated to other supported languages.
4. If message is sent via SMS, sender phone number is added to an etcd key-value store. etcd is used here to maintain a list of subscribers’ phone numbers, as well as their respective languages. An adjustable TTL value is used here to remove numbers from the store if the subscriber does not participate in the conversation for 300 seconds.
5. Translated messages/audio streams are published to various channels on the MQTT broker, which then distributes the messages among subscribing clients.

## Included components

* [IBM Cloud Functions](https://www.ibm.com/cloud/functions): Execute code on demand in a highly scalable, serverless environment.
* [Watson Text to Speech](https://www.ibm.com/cloud/watson-text-to-speech): Converts written text into natural sounding audio in a variety of languages and voices.
* [Watson Language Translator](https://www.ibm.com/cloud/watson-language-translator): Translate text from one language to another. Take text from across the globe and present it in the language of your choice.

## Featured technologies

* [Messaging](https://developer.ibm.com/messaging/message-hub/): Messaging is a key technology for modern applications using loosely decoupled architecture patterns such as microservices.
* [Node.js](https://nodejs.org/): An open-source JavaScript run-time environment for executing server-side JavaScript code.
* [OpenWhisk](https://www.ibm.com/cloud-computing/bluemix/openwhisk): An open source event-driven platform that allows you to execute code in response to an event. This is the underlying technology for the IBM Cloud Functions offering

# Watch the Video
[![](http://img.youtube.com/vi/eXY0uh_SeKs/0.jpg)](https://www.youtube.com/watch?v=eXY0uh_SeKs)

## Prerequisites:
To interact with the hosted offerings, the IBM Cloud CLI will need to be installed beforehand. Instructions can be found [here](https://cloud.ibm.com/docs/cli?topic=cloud-cli-install-ibmcloud-cli).

After installation is complete, confirm the CLI is working by printing the version
```
ibmcloud -v
```
And then install the "Cloud Functions" plugin with
```
ibmcloud plugin install cloud-functions
```

If the IBM Cloud CLI was already installed, please make sure the Cloud Functions plugin is the latest version with
```
ibmcloud plugin update cloud-functions
```


# Steps
1. [Create Services](#1-create-services)
2. [Deploy MQTT Feed](#2-deploy-mqtt-feed)
3. [Upload Actions](#3-upload-actions)
4. [Create Triggers](#4-create-triggers)
5. [Create Rules](#5-create-rules)
6. [Deploy UI](#6-deploy-ui)

### 1. Create Services

Create the required IBM Cloud services.
- [Speech To Text](https://cloud.ibm.com/catalog/services/speech-to-text)
- [Text To Speech](https://cloud.ibm.com/catalog/services/text-to-speech)
- [Watson IoT Platform](https://cloud.ibm.com/catalog/services/internet-of-things-platform)
- [Watson Language Translator](https://cloud.ibm.com/catalog/services/language-translator)

For SMS integration, create the following third party services.
- [Twilio](https://cloud.ibm.com/catalog/services/twilio-programmable-sms)
- [Redis](https://cloud.ibm.com/catalog/services/databases-for-redis)

Each service can be provisioned with the following steps

Navigate to the IBM Cloud dashboard at [https://cloud.ibm.com](https://cloud.ibm.com) and click the "Catalog" button in the upper right
<p align="center">
<img src="https://i.imgur.com/vFCHSF4.png">
</p>

Type in the name of the service and select the resulting icon
<p align="center">
<img src="https://i.imgur.com/X78OVt7.png">
</p>

Select the pricing plan and click "Create". If deploying on an IBM Lite account, be sure to select the free "Lite" plan
<p align="center">
<img src="https://i.imgur.com/S0iNZu0.png">
</p>

#### Additional Configuration: Generate Watson IoT service credentials
After being provisioned, the Watson IoT Platform service will need a bit of additional configuration, as we'll need to generate a set of credentials for connecting to the broker. We can do so by entering the IoT Platform dashboard, selecting "Devices" from the left hand menu, and then clicking the "Add Device" button.

<p align="center">
<img src="https://i.imgur.com/fec24FG.png"  data-canonical-src="https://i.imgur.com/fec24FG.png">
</p>

Next, provide a device type and ID.
<p align="center">
<img src="https://i.imgur.com/REQfYIK.png"  data-canonical-src="https://i.imgur.com/REQfYIK.png">
</p>

The next few tabs (Device Information, Groups, Security) can be left as is with the default settings.

<p align="center">
<img src="https://i.imgur.com/rycnjlF.png"  data-canonical-src="https://i.imgur.com/rycnjlF.png">
</p>

Clicking the "Finish" button will register a device and generate a set of credentials that can be used to publish messages to the IoT Platform. Be sure to take note of the Device type and Device ID, and place both in the `cfcreds.env` file.

We'll need to generate a different set of credentials to be able to publish and subscribe to the MQTT Broker
<!-- <p align="center">
<img src="https://i.imgur.com/A2A6yXW.png" width="650" height="450">
</p> -->

We can do so by selecting the "Apps" option in the left hand menu. Then, click the "Generate API Key" button
<p align="center">
<img src="https://i.imgur.com/b9Iu9DS.png" width="650" height="450">
</p>

We can leave the fields in the "Information" blank and click next.
In the "Permissions" tab, we'll select the "Backend Trusted Application" role. Once this is selected, click "Generate Key"
<p align="center">
<img src="https://i.imgur.com/ss6jpOZ.png" width="650" height="450">
</p>

The result will give us an API Key and Authentication Token. These can be supplied as the username and password for a MQTT client. To make setup a bit easier, place these values in the `cfcreds.env` file as IOT_API_KEY and IOT_APP_AUTH_TOKEN
<p align="center">
<img src="https://i.imgur.com/hfnB1B8.png" width="650" height="450">
</p>

At this point, the values `IOT_ORG_ID`, `IOT_DEVICE_TYPE`, `IOT_DEVICE_ID`, `IOT_AUTH_TOKEN`, and `IOT_API_TOKEN` should be in the cfcreds.env file. In this example, we used the npm [mqtt.js](https://www.npmjs.com/package/mqtt) package, which can be installed with `npm install -g mqtt`. And then, we can run a sample MQTT publish/subscribe command to confirm that the credentials are valid.

Subscribe to the MQTT broker in one tab
```
source cfcreds.env
mqtt_sub -i "a:${IOT_ORG_ID}:test" -u "${IOT_API_KEY}" -P "${IOT_AUTH_TOKEN}" -h "${IOT_ORG_ID}.messaging.internetofthings.ibmcloud.com" -p 1883 -t "iot-2/type/${IOT_DEVICE_TYPE}/id/${IOT_DEVICE_ID}/evt/fromClient/fmt/json"

```

And then publish in another
```
source cfcreds.env
mqtt_pub -i "a:${IOT_ORG_ID}:client_pub" -u "${IOT_API_KEY}" -P "${IOT_AUTH_TOKEN}" -h 'agf5n9.messaging.internetofthings.ibmcloud.com' -p 1883 -t "iot-2/type/${IOT_DEVICE_TYPE}/id/${IOT_DEVICE_ID}/evt/fromClient/fmt/json" -m '{
    "d" : {
          "sourceLanguage" : "en",
          "payload" : "test",
	        "client": "client1"
    }
}'
```

### 2. Deploy MQTT Feed

Install the MQTT package/feed found in the openwhisk-package-mqtt-watson submodule [here](https://github.com/kkbankol-ibm/openwhisk-package-mqtt-watson). This "feed" enables OpenWhisk to subscribe to one or more MQTT topics and invoke actions in response to incoming messages. To see more on how feeds work with IBM Cloud Functions, please visit these [documents](https://github.com/apache/incubator-openwhisk/blob/master/docs/feeds.md)

### 3. Upload Actions
Upload each "Action" to the Cloud Functions codebase with the following commands.
```
ibmcloud fn action create translateText translateText.js
ibmcloud fn action create sendSMS sendSMS.js
ibmcloud fn action create iotPub iotPub.py
ibmcloud fn action create handleIncomingSMS handleIncomingSMS.js
```

After each action is created, set or bind default credentials for the corresponding services.
```
# Most IBM Cloud native service credentials can be easily imported to a Cloud function using the "service bind" command
# ibmcloud fn service bind <service> <action_name>
ibmcloud fn service bind language_translator translateText
ibmcloud fn service bind language_translator handleIncomingSMS


# Credentials for the Watson IoT Platform and third party services can be set using the "update command"
# ibmcloud fn action update <action_name> -p <param_name> <param_value>
ibmcloud fn action update iotPub -p iot_org_id "${IOT_ORG_ID}" -p iot_device_id "${IOT_DEVICE_ID}" -p iot_device_type "${IOT_DEVICE_TYPE}" -p iot_auth_token "${IOT_AUTH_TOKEN}" -p iot_api_key "${IOT_API_KEY}"
ibmcloud fn action update sendSMS -p twilioNumber "${TWILIO_NUMBER}" -p twilioSid "${TWILIO_SID}" -p twilioAuthToken "${TWILIO_AUTH_TOKEN}" -p redisUsername "${REDIS_USER}" -p redisPassword "${REDIS_PASSWORD}" -p redisHost "${REDIS_HOST}" -p redisPort "${REDIS_PORT}"
ibmcloud fn action update handleIncomingSMS -p twilioNumber "${TWILIO_NUMBER}" -p twilioSid "${TWILIO_SID}" -p twilioAuthToken "${TWILIO_AUTH_TOKEN}" -p redisUsername "${REDIS_USER}" -p redisPassword "${REDIS_PASSWORD}" -p redisHost "${REDIS_HOST}" -p redisPort "${REDIS_PORT}"
```

### 4. Create Triggers
Create `Triggers` to represent events.
```
ibmcloud fn trigger create audioMsgReceived
ibmcloud fn trigger create txtMsgReceived
ibmcloud fn trigger create SMSMsgReceived
ibmcloud fn trigger create msgTranslated
```

### 5. Create Rules
Create `Rules`, which execute actions when certain triggers are activated.
```
# ibmcloud fn rule create RULE_NAME TRIGGER_NAME ACTION_NAME
ibmcloud fn rule create handleTxtMessage txtMsgReceived translateText
ibmcloud fn rule create handleMQTTMessage mqttMsgReceived translateText
ibmcloud fn rule create publishtoIOT msgTranslated iotPub
ibmcloud fn rule create publishtoSMS msgTranslated sendSMS
```

After the Feed has been deployed and the rules have been established, test the process by sending a MQTT message to the topic registered with the feed like so
```
source cfcreds.env
mqtt_pub -i "a:${IOT_ORG_ID}:client_pub" -u "${IOT_API_KEY}" -P "${IOT_AUTH_TOKEN}" -h 'agf5n9.messaging.internetofthings.ibmcloud.com' -p 1883 -t "iot-2/type/${IOT_DEVICE_TYPE}/id/${IOT_DEVICE_ID}/evt/fromClient/fmt/json" -m '{
    "d" : {
          "sourceLanguage" : "en",
          "payload" : "test",
	        "client": "client1"
    }
}'
```

As soon as this command is published, we should be able to see a series of actions and triggers being called in the Cloud Functions logs. These logs can be viewed by visiting [https://cloud.ibm.com/openwhisk/dashboard](https://cloud.ibm.com/openwhisk/dashboard) or by running `ibmcloud fn activation poll` in a separate tab.


### 6. Deploy UI

If all you need is the server side logic, you can stop here.  But optionally, you can deploy the UI provided by https://github.com/IBM/language-translation-ui. Be sure to source the `cfcreds.env` file beforehand, as the UI expects the `IOT_ORG_ID`, `IOT_DEVICE_TYPE`, `IOT_DEVICE_ID`, `IOT_AUTH_TOKEN`, and `IOT_API_TOKEN` values to be set as environment variables

```
git clone https://github.com/IBM/language-translation-ui
cd language-translation-ui
npm install
npm start
```

When the npm start command succeeds, you should be able to access the UI at http://127.0.0.1:8080

## Developer Notes

Flow:

- MQTT message received as JSON object to topic `iot-2/type/'${deviceType}/id/${orgId}/evt/${eventName}/fmt/json`
```
{
  client: "client_1234",
  payload: "hello",
  sourceLanguage: "en"
}
```

- Trigger associated with topic forwards message payload/language to translator action.
- Translator action passes message payload through a loop, where each item is a language that the original message will be translated to. After translation is complete, another trigger will be fired, which kicks off two other "publish" actions simultaneously.
  - One action publishes results to all MQTT clients
  - The other action looks up SMS subscriber numbers/language in Redis and sends them the result via Twilio.

<!-- Restrictions:

Watson IOT provides an MQTT broker, but has restrictions on how MQTT topics can be structured. So, only one section of the topic is customizable, the "event name". This limitation prevents us from using self describing topics like `fromClient/text/en`, which would allow clients to subscribe only to the language of their choice. Therefore they'll need to receive all messages and discard the unneeded ones.

MQTT package/feed requires a CF app, which technically means this implementation is not serverless. -->

# Links
<!-- * [Watson Node.js SDK](https://github.com/watson-developer-cloud/node-sdk) -->

## License
This code pattern is licensed under the Apache Software License, Version 2. Separate third party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1](https://developercertificate.org/) and the [Apache License, Version 2](https://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache License FAQ](https://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
