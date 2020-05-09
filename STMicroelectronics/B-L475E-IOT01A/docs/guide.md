<h1>Getting started with the STMicroelectronics B-L475E-IOT01</h1>

**Total completion time**:  45 minutes

In this tutorial you use Azure RTOS to connect the STMicroelectronics B-L475E-IOT01 (hereafter, the STM  DevKit) to Azure IoT.  The article is part of the series [Getting Started with Azure RTOS](https://review.docs.microsoft.com/azure/rtos/getting-started?branch=master). The series introduces device developers to Azure RTOS, and shows how to connect several micro-controller units (MCU) to Azure IoT.

You will complete the following tasks:

* Install a set of embedded development tools for programming the STM DevKit in C
* Build an image and flash it onto the STM MCU
* Create an Azure IoT hub and securely connect the STM DevKit to it
* Use Azure CLI to view device telemetry, view properties, and invoke cloud-to-device methods

## Prerequisites

* A PC running Microsoft Windows (Windows 10 recommended)
* If you don't have an Azure subscription, [create one for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.
* [Git](https://git-scm.com/downloads)
* Hardware

    > * The [STMicroelectronics B-L475E-IOT01](https://www.st.com/content/st_com/en/products/evaluation-tools/product-evaluation-tools/mcu-mpu-eval-tools/stm32-mcu-mpu-eval-tools/stm32-discovery-kits/b-l475e-iot01a.html) (STM DevKit)
    > * Wi-Fi 2.4 GHz
    > * USB 2.0 A male to Micro USB male cable

## Prepare the development environment

To set up your development environment, first you clone a GitHub repo that contains all the assets you need for the tutorial. Then you install a set of programming tools.

### Clone the repo for the tutorial

Clone the following repo to download all sample device code, setup scripts, and offline versions of the documentation for the series [Getting Started with Azure RTOS](https://review.docs.microsoft.com/azure/rtos/getting-started?branch=master). If you previously cloned this repo, you don't need to do it again.

To clone the repo, run the following command:

```
git clone https://github.com/AzureRTOS/getting-started
```

### Install the tools

The cloned repo contains a setup script that installs and configures the first set of required tools. After you run the setup script, you can install the remaining tools manually.

> Note: The setup script installs the following tools:
> * [GCC](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm): Compile
> * [CMake](https://cmake.org): Build
> * [Ninja](https://ninja-build.org): Build

To run the setup script:

1. Open a console app with administrator privileges, go to the following path in the repo, and run the setup script named *get-toolchain.bat*. If you use File Explorer, right-click the file and select **Run As Administrator**.

    > *getting-started\tools\get-toolchain.bat*

1. After the installation, open a new console window to recognize the configuration changes made by the setup script. Use this console to complete the remaining programming tasks in the tutorial. You can use Windows CMD, Powershell, or Git Bash for Windows.
1. Run the following code to confirm that CMake version 3.14 or later is installed.

    ```
    cmake --version
    ```

To install the remaining tools:

1. Download and install [ST-LINK](https://www.st.com/en/development-tools/stsw-link004.html). You use this utility to flash the device.

    >Note: After you click the button to accept the license, the page prompts you to log in or register. Register for a free account, if you don't have one. Then return to the page and log in to complete the installation.

1. Install [Termite](https://www.compuphase.com/software_termite.htm). You use this utility to monitor your device.

## Prepare Azure resources

To prepare Azure cloud resources and connect a device to Azure, you can use Azure CLI. There are two ways to access the Azure CLI: by using the Azure Cloud Shell, or by installing Azure CLI locally.  Azure Cloud Shell lets you run the CLI in a browser so you don't have to install anything.

Use one of the following options to run Azure CLI.  

If you prefer to run Azure CLI locally:

1. If you already have Azure CLI installed locally, run `az --version` to check the version. This tutorial uses Azure CLI 2.4.0 or later.
1. To install or upgrade, see [Install Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest). If you install Azure CLI locally, you can run CLI commands in the **GCC Command Prompt**, Git Bash for Windows, or Powershell.

If you prefer to run Azure CLI in the browser-based Azure Cloud Shell:

1. Use your Azure account credentials to sign into the Azure Cloud shell at https://shell.azure.com/.
    > Note: If this is the first time you've used the Cloud Shell, it prompts you to create storage, which is required to use the Cloud Shell.  Select a subscription to create a storage account and Microsoft Azure Files share.
1. Select Bash or Powershell as your preferred CLI environment in the **Select environment** dropdown. If you plan to use Azure Cloud Shell, keep your browser open to run the Azure CLI commands in this tutorial.

    ![Select CLI environment](images/cloud-shell-environment.png)

### Create an IoT hub

You can use Azure CLI to create an IoT hub that handles events and messaging for your device.

To create an IoT hub:

1. In your CLI console, run the [az extension add](https://docs.microsoft.com/cli/azure/extension?view=azure-cli-latest#az-extension-add) command to add the Microsoft Azure IoT Extension for Azure CLI to your CLI shell. The IOT Extension adds IoT Hub, IoT Edge, and IoT Device Provisioning Service (DPS) specific commands to Azure CLI.

   ```azurecli
   az extension add --name azure-iot
   ```

1. Run the [az group create](https://docs.microsoft.com/cli/azure/group?view=azure-cli-latest#az-group-create) command to create a resource group. The following command creates a resource group named *MyResourceGroup* in the *eastus* region. To see other available regions, you can run [az account list-locations](https://docs.microsoft.com/cli/azure/account?view=azure-cli-latest#az-account-list-locations).

    ```azurecli
    az group create --name MyResourceGroup --location eastus
    ```

1. Run the [az iot hub create](https://docs.microsoft.com/cli/azure/iot/hub?view=azure-cli-latest#az-iot-hub-create) command to create an IoT hub. It might take a few minutes to create an IoT hub.

    *YourIotHubName*. Replace this placeholder below with the name you chose for your IoT hub. An IoT hub name must be globally unique in Azure. This placeholder is used in the rest of this tutorial to represent your unique IoT hub name.

    ```azurecli
    az iot hub create --resource-group MyResourceGroup --name {YourIoTHubName}
    ```

1. After the IoT hub is created, view the JSON output in the console, and copy the `hostName` value from the following named field to a safe place. You use this value in a later step.

### Register a device

In this section, you create a new device instance and register it with the Iot hub you created. You will use the connection information for the newly registered device to securely connect your physical device in a later section.

To register a device:

1. In your console, run the [az iot hub device-identity create](https://docs.microsoft.com/cli/azure/ext/azure-cli-iot-ext/iot/hub/device-identity?view=azure-cli-latest#ext-azure-cli-iot-ext-az-iot-hub-device-identity-create) command. This creates the simulated device identity.

    *YourIotHubName*. Replace this placeholder below with the name you chose for your IoT hub.

    *MySTMDevice*. You can use this name directly for the device in CLI commands in this tutorial. Optionally, use a different name.

    ```azurecli
    az iot hub device-identity create --device-id MySTMDevice --hub-name {YourIoTHubName}
    ```

1. After the device is created, view the JSON output in the console, and copy the `deviceId` and `primaryKey` values to use in a later step.

Confirm that you have the copied the following values from the JSON output to use in the next section:

> * `hostName`
> * `deviceId`
> * `primaryKey`

## Prepare the device

To connect the STM DevKit to Azure, you'll modify a configuration file for Wi-Fi and Azure IoT settings, rebuild the image, and flash the image to the device.

### Add configuration

1. In a text editor, edit the file *getting-started\STMicroelectronics\B-L475E-IOT01A\app\azure_config.c* to set the Wi-Fi constants to the following values from your local environment.

    |Constant name|Value|
    |-------------|-----|
    |`wifi_ssid` |{*Your Wi-Fi ssid*}|
    |`wifi_password` |{*Your Wi-Fi password*}|
    |`wifi_mode` |{*One of the enumerated Wi-Fi mode values in the file*}|

1. Edit the same file to set the Azure IoT device information constants to the values that you saved after you created Azure resources.

    |Constant name|Value|
    |-------------|-----|
    |`iot_hub_hostname` |{*Your Iot hub hostName value*}|
    |`iot_device_id` |{*Your deviceID value*}|
    |`iot_sas_key` |{*Your primaryKey value*}|

### Build the image

In your console or in File Explorer, run the script *rebuild.bat* at the following path to build the image:

> *getting-started\STMicroelectronics\B-L475E-IOT01A\tools\rebuild.bat*

After the build completes, confirm that a binary file was created in the following path:

> *getting-started\STMicroelectronics\B-L475E-IOT01A\build\app\stm32_azure_iot.bin*

### Flash the image

1. On the STM DevKit MCU, locate the **Reset** button, and the Micro USB port which is labeled **USB STLink**. You use these components in the following steps. Both are highlighted in the following picture:

    ![STM DevKit board reset button and micro usb port](images/stm-devkit-board.png)

1. Connect the Micro USB cable to the Micro USB port on the STM DevKit, and then connect it to your computer.
    > Note: For detailed setup information about the STM DevKit, see the instructions on the packaging, or see [Tools and Resources](https://www.st.com/content/st_com/en/products/evaluation-tools/product-evaluation-tools/mcu-mpu-eval-tools/stm32-mcu-mpu-eval-tools/stm32-discovery-kits/b-l475e-iot01a.html#resource).
1. Start the **STM32 ST-LINK Utility**.
1. Select **Target > Program**.
1. In the **Download** dialog, select **Browse**, select the binary file **stm32_azure_iot.bin** that you built in the [Build the image](#build-the-image) section, and select **Open**.
1. Select **Start**. You don't need to change existing field values.

The flashing process happens quickly. After flashing completes successfully, the **STM32 ST-LINK Utility** displays a message that starts with *Memory programmed in* and indicates how long it took:

![Flash a device with the STM32 ST-LINK Utility](images/stm32-stlink-utility.png)

### Confirm device connection details

You can use the **Termite** utility to monitor communication and confirm that your device is set up correctly.

1. Start **Termite**.
1. Select **Settings**.
1. In the **Serial port settings** dialog, check the following settings and update if needed:
    * **Baud rate**: 115,000
    * **Port**: The port that your STM DevKit is connected to. If there are multiple port options in the dropdown, you can find the correct port to use. Open Windows **Device Manager**, and view **Ports > STMicroelectronics STLink Virtual COM Port** to identify which port to use.
1. Press the **Reset** button on the board. The button is black and is labeled on the board.
1. In the **Termite** console, check the following checkpoint values to confirm that the device is initialized and connected to Azure IoT. If a checkpoint value is missing or incorrect and you can't resolve the issue, see [Troubleshooting](#troubleshooting).

    |Checkpoint name|Output value|
    |---------------|-----|
    |DHCP |DHCP initialized|
    |DNS |DNS client initialized|
    |SNTP |SNTP initialized|
    |MQTT client |MQTT thread initialized|

The Termite console shows the details about the board, your connection, and the checkpoint values.

![Termite output with connection checkpoints](images/termite-output-checkpoints.png)

Keep Termite open to monitor device output in the following steps.

## View telemetry

You can use Azure CLI to inspect the flow of telemetry from the device to Azure IoT.

1. In your CLI console, run the [az iot hub monitor-events](https://docs.microsoft.com/cli/azure/ext/azure-cli-iot-ext/iot/hub?view=azure-cli-latest#ext-azure-cli-iot-ext-az-iot-hub-monitor-events) command to monitor telemetry from your device. Use the names that you created previously in Azure IoT for your device and IoT hub.

    > Note: The first time you run this command, Azure CLI might prompt to install a *Dependency update (uamqp 1.2) required for IoT extension version: 0.9.1*. Select *y* to install the update. If the CLI also prompts to install the older `azure-iot-cli-ext` version of the Azure IoT CLI extension, select *no*.

    ```azurecli
    az iot hub monitor-events --device-id MySTMDevice --hub-name {YourIoTHubName}
    ```

1. To force the STM DevKit to reconnect and send telemetry, press **Reset**.

    View the telemetry in the console's JSON output.

    ```json
    {
        "event": {
            "origin": "MySTMDevice",
            "payload": "{\"temperature\": 25}"
        }
    }
    ```

1. Select CTRL+C to end monitoring.

## View device properties

Using Azure CLI, you can inspect the properties on your Azure resources, including your connected device.

1. Run the [az iot hub device-identity list](https://docs.microsoft.com/cli/azure/ext/azure-cli-iot-ext/iot/hub/device-identity?view=azure-cli-latest#ext-azure-cli-iot-ext-az-iot-hub-device-identity-list) command to list devices attached to your Iot hub.

    ```azurecli
    az iot hub device-identity list --hub-name {YourIoTHubName}
    ```

    The following partial JSON output shows how the connected device is included in the device list.

    ```json
    {
    "authenticationType": "sas",
    "capabilities": {
        "iotEdge": false
    },
    "cloudToDeviceMessageCount": 0,
    "connectionState": "Connected",
    "deviceEtag": "Njc0NTAzODkw",
    "deviceId": "MySTMDevice",
    ```

1. Run the [az iot hub device-identity show](https://docs.microsoft.com/cli/azure/ext/azure-cli-iot-ext/iot/hub/device-identity?view=azure-cli-latest#ext-azure-cli-iot-ext-az-iot-hub-device-identity-show) command to view the properties of your device.

    ```azurecli
    az iot hub device-identity show --device-id MySTMDevice --hub-name {YourIoTHubName}
    ```

## Call a direct method on the device

You can use the Azure CLI to call a direct method on your board from a console. Direct methods have a name, and can optionally have a JSON payload, configurable connection, and method timeout.

To call a direct method on your device:

1. Run the [az iot hub invoke-device-method](https://docs.microsoft.com/cli/azure/ext/azure-cli-iot-ext/iot/hub?view=azure-cli-latest#ext-azure-cli-iot-ext-az-iot-hub-invoke-device-method) command to invoke a direct method.

    <!-- Inline code tag and CSS to wrap long code lines. -->
    <code style="white-space : pre-wrap !important;">
    az iot hub invoke-device-method --device-id MySTMDevice --method-name MyMethod --method-payload '{"Greeting":"Hello world!"}' --hub-name {YourIoTHubName}
    </code>


    **Note**: If you use Powershell use the following code block instead. This code formats the JSON `method-payload` parameter according to Powershell formatting rules.

    <code style="white-space : pre-wrap !important;">
    az iot hub invoke-device-method --device-id MySTMDevice --method-name MyMethod --method-payload '{\"Greeting\": \"Hello world!\"}' --hub-name {YourIoTHubName}
    </code>

    The console shows the status of your method call on the device, where `1` indicates success.

    ```json
    {
        "payload": {},
        "status": 1
    }
    ```

1. View the Termite console to see the JSON payload output:

    ![Termite output for direct methods](images/termite-output-direct-method.png)

## Clean up resources

If you no longer need the Azure resources created in this tutorial, you can use the Azure CLI to delete them.

If you continue to another tutorial in the series [Getting Started with Azure RTOS](https://review.docs.microsoft.com/azure/rtos/getting-started?branch=master), you can keep the resources you've already created and reuse them.

> **Important**: Deleting a resource group is irreversible. The resource group and all the resources contained in it are permanently deleted. Make sure that you do not accidentally delete the wrong resource group or resources.

To delete a resource group by name:
1. Run the [az group delete](https://docs.microsoft.com/cli/azure/group?view=azure-cli-latest#az-group-delete) command. This removes the resource group, the IoT Hub, and the device registration you created.

    ```azurecli
    az group delete --name MyResourceGroup
    ```

1. Run the [az group list](https://docs.microsoft.com/cli/azure/group?view=azure-cli-latest#az-group-list) command to confirm the resource group is deleted.  

    ```azurecli
    az group list
    ```

## Troubleshooting

**TODO**

[We will temporarily include this information here, but move into separate article in the Docs RTOS repo for GA. ]

List of different troubleshooting to deal with edge cases arising from the steps within this guide. This would include device connection errors, Wi-Fi connection errors, Central connection errors, etc. 

Provide a pre-built binary that is known to work so they can test that they can use to test flashing and monitoring.

## Next Steps 
**Ryan TODO**

Describe the shape of the application and how to customize for their own need 
1. List the cmake structure
1. List the app + lib folders
1. List common folder