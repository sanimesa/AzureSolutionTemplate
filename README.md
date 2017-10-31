# Solution overview
This solution creates a simply deployable reference architecture in Azure that allows the user to ingest process store and analyze large quantity of sensor data coming in from the Loriot gateway.  This repository offers to Loriot customer an almost ready to use solution that allows them to quickly visualize Lora connected device data. This could be a starting point to a more complex productive solution. (link get started)

Loriot provides LoRaWAN software products and software services. With the new tool, customers can now provision the IOT infrastructure out of the Loriot Management Portal into the end customers Azure subscription with almost no configuration.  

The proposed solution allows the end user to provision new IoT devices very quickly into their Azure subscription and synchronize the device provisioning into the Loriot account. Thanks to the Loriot-Azure IoT Hub connector data can be easily collected by the IoT Hub and then decoded through a custom decoding Function. Through the optional Time Series component users can quickly analyze and display sensor data. The solution also implement an example how to store all sensor data to Cosmos DB and the example temperature data to SQL Database. The template alos offer a PowerBI Dashboard with some preconfigured charts including realtime temperature data. Furthermore the PowerBI report is able to detect and display inactive or broken devices.  

![Architecture Diagram Loriot](images/Loriot_Architecture.jpg)

## Get Started

Please ensure you have your Loriot App ID / App Token / API URL, you will need it during deployment.
To run directly on Azure press the button below:
<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FLoriot%2FAzureSolutionTemplate%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>

To run locally, ensure you have the latest Azure CLI installed from [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).
Run with the following command:

```powershell
az group deployment create --name ExampleDeployment --resource-group YourResourceGroup --template-file azuredeploy.json --parameters azuredeploy.parameters.json
```

Once the deployment has completed perform the following actions:
1. navigate to the [Azure portal](https://portal.azure.com) and start the Stream Analytics job manually using the button at the top of the blade. For instructions on how to do this if you have chosen to deploy the Power BI component of the template, please see the [power-bi](power-bi/) folder readme
2. Add your new IoT Devices in Azure IoT Hub, the devices will be automatically provisioned into the Loriot Portal
3. Add the Device Twin Meta Data as specified [here](#device-twin-setup) .


## Solution Content

[IoT Hub](#iot-hub)
- [Device Twin Setup (Required)](#device-twin-setup)

[Azure Functions](#azure-functions)
- [Router Function](#router-function)
- [Setup Function](#setup-function)
- [Decoder Function](#decoder-function)
- [Detect Inactive Devices Function](#detect-inactive-devices-function)
- [Loriot Lifecycle Function](#loriot-lifecycle-function)
- [Function Environment Variables](#environment-variables):

    - [LORIOT_APP_ID](#loriot_app_id)
    - [LORIOT_API_KEY](#loriot_api_key)
    - [LORIOT_API_URL](#loriot_api_url)
    - [IOT_HUB_OWNER_CONNECTION_STRING](#iot_hub_owner_connection_string)
    - [EVENT_HUB_ROUTER_INPUT](#event_hub_router_input)
    - [EVENT_HUB_ROUTER_OUTPUT](#event_hub_router_output)
    - [DOCUMENT_DB_NAME](#document_db_name)
    - [DOCUMENT_DB_ACCESS_KEY](#document_db_access_key)
    - [SQL_DB_CONNECTION](#sql_db_connection)
    - [DEVICE_LIFECYCLE_CONNECTION_STRING](#device_lifecycle_connection_string)
    - [DEVICE_LIFECYCLE_QUEUE_NAME](#device_lifecycle_queue_name)
    - [DEVICE_LIFECYCLE_IMPORT_TIMER](#device_lifecycle_import_timer)

[Testing](#testing)
- [Device Emulation (Optional)](#device-emulation)

[Time Series Insights (Optional)](#time-series-insights)

[CosmosDB](#cosmos-db)

[Azure SQL Database](#azure-sql-database)

[Stream Analytics](#stream-analytics)

[Power BI (Optional)](#power-bi)

[About Loriot](#about-loriot)



## IoT Hub
Azure IoT Hub is a scalable, multi-tenant cloud platform (IoT PaaS) that includes an IoT device registry, data storage, and security. It also provides a service interface to support IoT application development.

### Device Twin Setup

When adding new devices to the IoT Hub, ensure you modify the Device Twin to include the following tags in order for the routing function to assign the correct decoder:

![Device Twin - Add Tags](images/DeviceTwinAddTags.png)

## Azure Functions

Azure Functions is a solution for easily running small pieces of code, or "functions" in the cloud. You can write the code you need for the problem at hand, without worrying about a whole application or the infrastructure to run it. Azure Functions lets you develop serverless applications on Microsoft Azure.
In this project you will find the following functions:

### Router Function

The router function is triggered by messages coming from the IoT Hub (connection defined in the EVENTHUB_ROUTER_INPUT environment variable) and routes them to the appropriate decoder.

Routing is done based on the *sensordecoder* property present in the device twins tags in the IoT Hubs (connection defined in the IOT_HUB_OWNER_CONNECTION_STRING environment variable) - see [Device Twin Setup](#device-twin-setup) for more information. The function can access this information using the *iothub-connection-device-id* message property automatically added by the IoT Hub.
In a nutshell, routing takes the following strategy:

- If an environment variable with name "DECODER_URL_*sensordecoder*" or "DECODER_URL_DEFAULT_*sensordecoder*" exists, the message will be routed there.
- If those environment variables are not present in the web app, the message will be automatically routed to a function named after the *sensordecoder* located on the same function app. The route will be https://{nameOfCurrentFunctionApp}.azurewebsites.net/api/{*sensordecoder*}

The output of the function will be directed to an Event Hub (connection defined by EVENT_HUB_ROUTER_OUTPUT environment variable). Output messages are composed of the following subsections:

- MessageGuid: A unique GUID generated by the Router function to track each message.
- Raw: An exact copy of the raw message received from the IoT Hub.
- Metadata: Device twins tags from the IoT Hub.
- Decoded: Message from the IoT device decoded by the appropriate decoder.

### Setup Function

The setup function initialises the Cosmos DB collection and the SQL table needed by the pipeline. The function is automatically triggered at the end of the execution of the ARM template. The function uses environment variables DOCUMENT_DB_NAME, DOCUMENT_DB_ACCESS_KEY and SQL_DB_CONNECTION to connect to the two resources.

If the collection or table already exist, the function will return without doing anything.

### Decoder Function

Decoder functions perform the decoding of the sensor raw data payload. By default our solution provide you with 3 example decoder, but the system is built to be easily extensible. Additional decoders can be hosted anywhere and must simply be HTTP REST accessible and be routed correctly by the [router function rules](#router-function).

### Detect Inactive Devices Function

### Loriot Lifecycle Function

### Functions Environment variables

The function application is using a certain amount of environment variables populated at the ARM deploy time. You will in this section description and example of each variables.

#### LORIOT_APP_ID

The LORIOT App ID used to identify under which app the devices are synced.

```
BA7B0CF5
```

#### LORIOT_API_KEY

Key used to authenticate requests towards LORIOT servers.

```
********************x9to
```

#### LORIOT_API_URL

The base URL of the Network Server Management API used to sync device information between Azure IoT Hub and LORIOT servers.

```
https://eu1.loriot.io/1/nwk/app/
```

#### IOT_HUB_OWNER_CONNECTION_STRING

The connection string to the IoT Hub used for device syncing and reading the device registry.

```
HostName=something.azure-devices.net;SharedAccessKeyName=iothubowner;SharedAccessKey=fU3Kw5M5J5QXP1QsFLRVjifZ1TeNSlFEFqJ7Xa5jiqo=
```

#### EVENT_HUB_ROUTER_INPUT

The connection string of the IoT Hub's Event Hub, used as trigger on the RouterFunction to send the messages to the appropriate decoders.

```
Endpoint=Endpoint=sb://something.servicebus.windows.net/;SharedAccessKeyName=iothubowner;SharedAccessKey=UDEL1prJ9THqLJel+uk8UeU8fZVkSSi2+CMrp5yrrWM=;EntityPath=iothubname;
SharedAccessKeyName=iothubowner;SharedAccessKey=2n/TlIoLJbMjmJOmadPU48G0gYfRCU28HeaL0ilkqMU=
```

#### EVENT_HUB_ROUTER_OUTPUT

Connection string defining the output of the router function to the enriched and decoded message Event Hub.

```
Endpoint=sb://something.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=Ei8jNFRlH/rAjYKTTNxh7eIHlgeleffFekHhnyAxrZ4=
```

#### DOCUMENT_DB_NAME

Document Database name

#### DOCUMENT_DB_ACCESS_KEY

Key of the Document Database

#### SQL_DB_CONNECTION

Connection String of the SQL Database

```
Server=tcp:something.database.windows.net,1433;Initial Catalog=testdbmikou;Persist Security Info=False;
User ID=username;Password=password;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
```

#### DEVICE_LIFECYCLE_CONNECTION_STRING

#### DEVICE_LIFECYCLE_QUEUE_NAME

#### DEVICE_LIFECYCLE_IMPORT_TIMER

## Testing 
We do provide a ready to use testing setup in order to test and understand the infrastructure you deployed in a matter of minutes.

### Device Emulation

Provided with this project is a script that can be used to generate device messages to test the pipeline in the absence of a real device. For more information, see the [test](test/) folder.


## Time Series Insights 

Azure Time Series Insights (TSI) is a fully managed analytics, storage, and visualization service that makes it simple to explore and analyze billions of IoT events simultaneously. It gives you a global view of your data, letting you quickly validate your IoT solution and avoid costly downtime to mission-critical devices by helping you discover hidden trends, spot anomalies, and conduct root-cause analyses in near real-time. 

### How to set it up?

Deployment of TSI is fully optional and can be toggled on and off without impacting other components. Data ingestion by the service is already configured, admin will have to grant access to the data dashboard to the users (TSI blade -> Data Access Policies --> Add). Users will then be able to see the data flowing in real time by going to the Time Series URL (displayed on the overview blade).

## Cosmos DB 

Azure Cosmos DB is Microsoft's globally distributed, multi-model database. With the click of a button, Azure Cosmos DB enables you to elastically and independently scale throughput and storage across any number of Azure's geographic regions.

Cosmos DB is used on our architecture to provide data to the offline client [@Anita]. 

## Azure SQL database

Azure SQL Database is the intelligent, fully-managed relational cloud database service built for developers. Accelerate app development and make maintenance easy and productive using the SQL tools you love to use. Take advantage of built-in intelligence that learns app patterns and adapts to maximize performance, reliability, and data protection. 
[@Anita]

## Power BI

This deployment also provides (optional) Power BI visualisation functionality as a starting point for data analysis (both realtime and historical). For instructions on how to make use of this capability please look in the [power-bi](power-bi/) folder.

## About Loriot

LORIOT AG is a Swiss start-up in the field of Internet of Things, founded in 2015.
The core product today is software for scalable, distributed, resilient operation of LoRaWAN networks and end-to-end applications, which is offered under a variety of business models.
Due to their positioning in the LoRa ecosystem as both software provider and network operator, they are in direct contact with LoRa hardware producers and integrate many of their solutions directly with their services.
The collaboration allows them to offer not only network software, but a complete end-to-end solution for a real-world IoT application, including gateway and sensor hardware.
Their typical customers are small and medium enterprises in the Internet of Things business, cities, municipalities and wireless network operators.
