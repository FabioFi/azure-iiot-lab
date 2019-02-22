# Azure Industrial IoT Lab: Connecting factory machines in 60 minutes with Azure IoT Edge and OPC UA

Introduction
============

Connecting IIoT (*Industrial Internet of Things*) devices to collect
data and manage the respective devices centrally from the cloud becomes
more and more relevant as the number of devices increase.

As part of our Microsoft strategy we want to achieve convergence between
the Intelligent Cloud and the Intelligent Edge. Making use of data
generated by an IoT device for various use cases is an essential part of
this.

![](./media/image1.png)

In this lab you will learn how you set up a complete IIoT scenario using
the IoT Hub on Azure and IoT Edge to collect data, manage devices and
visualize data again.

The following architecture diagram gives an overview about what you will
deploy in the lab.

![](./media/image2.png)

For the purpose of this lab, the simulated factory OPC UA server is
provided in a Docker container running in an Azure Container Instance. But you can just as easily exchange this for any other OPC UA server.

### **Disclaimer**: This lab requires an Azure Subscription. A couple of resources will be created which do incur some small cost. We recommend to delete the resources when you are finished with the lab and don't need them anymore. If you do not have an Azure Subscription yet, you can also do this lab with a [free trial subscription]( https://azure.microsoft.com/en-us/free/)

Step-by-step guide
==================

Below you find a detailed step-by-step guide that helps you connecting
your first IoT Edge device and gaining insights into the produced data.

---
**Please make sure to read all the instructions carefully.**  
*Note: As certain details in Azure do change over time, some of the instructions or screenshots might be outdated by the time you do this lab. If you come across any major blockers, please open an Issue here on the GitHub page and we will try to fix it as soon as possible.*

---

1.  **Deploy ARM Template**

    As a first step, we deploy the provided ARM (Azure Resource Manager) 
    template for this lab. This includes a couple of resources which will be
    provisioned for you, ready to use in the lab. We rather want to focus
    on certain topics around IIoT than spending our time deploying virtual machines.
	
    *   To start the deployment, click on this button (or better: right-click and open in new tab):

        [![Deploy to Azure](http://azuredeploy.net/deploybutton.png)](https://azuredeploy.net/)

        *Note: Instead of using the Deploy to Azure service, you can also manually deploy the [ARM template](azuredeploy.json), for instance through the Azure [Portal](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-template-deploy-portal#deploy-resources-from-custom-template) or [CLI](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-template-deploy-cli#deploy-external-template)*
	
    *   Use your username and password to log into your Azure subscription.

    *   Select your Directory, Subscription and fill out the parameter fields. It is recommended to create a new resource group for this lab. This makes it easier to dispose of all the created resources after you are done.  
    *Note: The list of available Azure regions for this template is currently limited as there is a dependency on a feature in Azure Container Instances (VNet support).*

        ![](./media/image49.png)
    
    *   Click on *Next* and review the resources to be created

        ![](./media/image50.png)
  
        The ARM template deploys the following resources:
        * 1 IoT Hub (S1 tier)
        * 1 pre-configured IoT Edge Virtual Machine. You can find this
        in the Azure Marketplace if you search for 
        "Ubuntu Server 16.04 LTS + Azure IoT Edge runtime"
        * 1 OPC UA Sample server running in an Azure Container Instance

    *   Click on *Deploy* and wait until all resources are ready. This should only take a few minutes.

        ![](./media/image51.png)

        Once this is finished, click on the *Manage your resources* link. This will take you directly into the Azure Portal
  

1. **Explore resources**

    Let us start by seeing which resources have been deployed for
    you through the provided ARM template.

    *   The Azure Portal should have been opened to your newly created Resource Group. If not, look on the left-hand side of the Azure portal for *"Resource groups"* and click on it.

        ![](./media/image4.png)

    -   Navigate to the created resource group and familiarize yourself with what  has been set up for you.

        ![](./media/image6.png)

        Like in the screenshot above, you should see one virtual machine (and
        its associated resources such as one virtual disk, a network interface,
        public IP address, virtual network etc.), one Azure Container Instance and one IoT Hub.

1.  **Create Edge Device**

    First step is to create a new Edge device identity in the IoT Hub.
    We will use this device identity later for our Edge device to
    authenticate and connect to the IoT Hub.

    -   Navigate to your IoT Hub and open the *IoT Edge* pane.

        ![](./media/image7.png)

    -   Click on *Add an IoT Edge Device* and pick a device name, such as
        *myedgedevice*. Click on *Save* to create the device.

        ![](./media/image8.png)

        ![](./media/image9.png)
        ![](./media/image10.png)

1.  **Get Edge Device Connection String**

    -   Select the newly created device. This will bring you to the device details page, including the connection string which we need soon.

    -   Copy the Connection String (primary key) and store it e.g. in
        Notepad for easy access. Make sure to copy it in the Notepad in your
        lab environment. Copy & Paste to your local client won't work.

        ![](./media/image11.png)

1.  **Connect IoT Edge device**

    Now that we have created the device identity in IoT Hub, we can
    connect our device. As the edge device, we use the pre-provisioned
    VM in your subscription. This VM has the IoT Edge runtime
    pre-installed and just needs your connection string to connect. In a
    real factory scenario, this Edge device would in most cases run
    inside the customer network (on-premises). It could, for instance,
    be a ruggedized hardware device, a server or a VM.

    -   Go back to the resource group and then select your VM

        ![](./media/image12.png)

    -   Copy the public IP address of your VM. Each VM has SSH access
        enabled.

        ![](./media/image13.png)

    -   Open an SSH client, for instance **PuTTY**

    -   Enter the IP and click *Open*

        ![](./media/image15.png)

    -   Accept the certificate prompt and enter the username and
        password which you have chosen when you deployed the ARM template in the beginning

        ![](./media/image16.png)

    -   To configure the Edge runtime, there is a script pre-installed      that comes with the VM. To execute it, run (put in your             connection string from the previous step):

        sudo /etc/iotedge/configedge.sh \"**your-connectionstring**\"
        
        *(Do not forget the double quotes around the connection string!)*

        ![](./media/image17.tmp)

    -   This will set the connection string and restart the Edge runtime.
        This will start to pull the Edge Agent docker container from the
        public Microsoft Container Registry (MCR).

    -   While this is running, we can continue with the next step and come
        back to our VM in a minute. Leave PuTTY open in the background.

1.  **Create Time Series Insights**

    As a first sink ("warm storage") for our IoT data, we use Azure Time
    Series Insights.

    -   Go back to the portal and click on *Create a resource*. Search for
    *Time Series Insights*

        ![](./media/image18.png)

    -   Pick an Environment name and leave the other values as suggested.
    Click on *Next: Event Source*

        ![](./media/image19.png)

    -   Now we directly connect TSI against our IoT Hub.

        Put in a name for the source, e.g. **iothub**

        Select your IoT Hub from the list and select **iothubowner** as the
        access policy name.

        Click on *New* for the consumer group and pick some name, e.g.
        **tsi** and click *Add*

        ![](./media/image20.png)

    -   Click *Review + create*, review the details and click *Create*

        ![](./media/image21.png)

1.  **Deploy Simulated Temperature Sensor**

    As a first check if our Edge device is working, we deploy the
    simulated temperature sensor. This is a module built by Microsoft to
    create simulated sensor readings.

    -   In the Search box on the top in the Azure portal enter **Simulated
    Temperature Sensor**

        ![](./media/image22.png)

    -   Select the item under *Marketplace*

    -   In the following screen your IoT Hub should already have been
    pre-selected. If not, pick the one that was created as part of the ARM template.

    -   Click on *Find Device* to get a list of your Edge devices. Select
    the device you created before.

        ![](./media/image23.png)

    -   Click on *Create*

        ![](./media/image24.png)

    -   This now brings us to the Set Modules screen of the IoT Edge device
    in your IoT Hub. Under Deployment Modules you can see that the
    temperature sensor has been already configured for you.

        ![](./media/image25.png)

    -   You can click on it to see its Image URI (again, hosted on MCR) and
    the preset desired properties of that module. Click on *Save* and
    then on *Next*.

        ![](./media/image26.png)

    -   Make sure to only have one route as shown in the screenshot below.
    If there is more than one route, delete the other ones.
    The route defines how messages flow between modules and/or to the
    IoT Hub in the cloud. In our case we do not do any Edge processing
    and send all messages directly to the cloud. (\$upstream means "send
    to IoT Hub in Azure").

        ![](./media/image27.png)

    -   Click on *Next.* This shows an overview of the complete
    deployment.json that will be sent to the device as configuration.
    Take a look at it and then click *Submit* to start the deployment.

1.  **Check Edge Device**

    Now that we have kicked off the deployment let's see if our Edge
    device got it correctly.

    -   Go back to your SSH client window (e.g. PuTTY).

    -   Enter **sudo iotedge list** This will show you the list of all
    running modules. You should now see three modules running:
    edgeAgent, edgeHub (both modules of the Edge runtime) and the new
    SimulatedTemperatureSensor

        ![](./media/image28.png)

    -   Enter **sudo iotedge logs SimulatedTemperatureSensor -f** to see the
    logs of the module. You will see that the module is sending
    simulated events.

        ![](./media/image29.png)

1.  **See simulated events in Time Series Insights**

    We can now go to our TSI environment and see the events.

    -   Go back to the Azure portal and open your TSI resource. Go into your
    Resource Group and select your Time Series Insights environment.

        ![](./media/image30.tmp)

        ![](./media/image31.png)

    -   Click on *Go to Environment*

    -   This opens the TSI user interface and you should already see of
    spike of events

    -   Right-click on the event graph and select *Explore Events*. This
    will show the raw data as it is being generated and sent by your
    Simulated Temperature Sensor.

        ![](./media/image32.png)

        ![](./media/image33.png)

    Now it's time to move from our generated data to the real thing. We will
    now deploy another module, that connects against our OPC UA-enabled PLC.

    For this we will use the Microsoft-built OPC UA Publisher module
    (https://github.com/Azure/iot-edge-opc-publisher).

1. **Deploy OPC UA Publisher module**

    Just as the Simulated Temperature Sensor, we will now add the OPC Publisher as
    a second module.

    -   Go again to your IoT Hub in the portal (*Hint*: if you click into
    the search box on the top of the portal again, you IoT Hub should
    show up as a recently used resource and thus give you a quick
    navigation to the resource).

    -   Click on *IoT Edge* in the left panel and select your previously
    created edge device.

        ![](./media/image34.png)

    -   Now go to *Set modules*. You will come back to the screen where we
    have deployed the temperature sensor module before.

    -   The container create options are used to set specific command line
    switches for the module. For all the available switches, you can
    take a look at the [module
    documentation](https://github.com/Azure/iot-edge-opc-publisher#using-it-as-a-module-in-azure-iot-edge).

    In order to reduce complexity, the following example config contains
    a minimum set of options.

    `--publishfile`: The name of the file which will later store the
    published nodes configuration

    `--diagnosticsinterval`: Enables the output of diagnostic info in the
    log every 10 seconds

    `--autoaccept`: Automatically accept SSL certificates from the OPC UA
    server

    `--fetchdisplayname`: Enables reading and sending of node display
    names (if set in the server)

    Additionally, in the HostConfig a Bind is created to persist the
    application data, such as the published nodes config on the host
    system over container restarts.

    Under Deployment Modules click on *Add*, choose *IoT Edge Module*
    and enter the following values:

    Name: **opc-publisher**

    Image URI: **mcr.microsoft.com/iotedge/opc-publisher:2.3.3**

    Container Create Options:
    ```
    {
        "Hostname": "opc-publisher",
        "Cmd": [
            "publisher",
            "--publishfile=./publishednodes.json",
            "--diagnosticsinterval=10",
            "--autoaccept",
            "--fetchdisplayname=true"
        ],

        "HostConfig": {
            "Binds": [
                "/iiotedge:/appdata"
            ]
        }
    }
    ```

    - Click on *Save* and then on *Next*.

        ![](./media/image35.png)

    -   The routes will not be changed. Since we still want to send all
    messages from all modules to the cloud, the previous created route
    is still good.

        ![](./media/image36.png)

    -   Click *Next* and *Submit* to send the deployment to the Edge device.
    Leave the following page open as we will come back to it soon.

1. **Check OPC UA Publisher module**

    After a few seconds the Edge device will have pulled the module.

    -   Go back to your PuTTY session and enter **sudo iotedge list** again
        to see if the publisher module is yet there.

        ![](./media/image37.png)

    -   Once it shows up, do **sudo iotedge logs opc-publisher -f** to see
        its logs.

        ![](./media/image38.png)

    -   You will see, that yet no configuration has been set in the
        publisher. I.e. which OPC server to connect to and which nodes to
        read. This comes next. You can leave the log open and running for
        now.

1. **Retrieve OPC UA Server address**

    In this lab, you will connect to a simulated PLC that exposes an OPC
    UA server. 
	This server is running in an Azure Container Instance and was set up as part of the ARM deployment in the beginning of the lab.
	The sample server is available as open-source [here](https://github.com/Azure-Samples/iot-edge-opc-plc).

    -   Go back to your resource group in the Azure Portal
    -   Find your Azure Container instance named *opc-server* and click on it
    - Copy the public IP address of the container, e.g. into a Notepad

1. **Configure published nodes**

    The configuration of the publisher which nodes to read from the
    source OPC UA server, the so called "published nodes", can be done
    via a Direct Method call from the IoT Hub to the module.

    -   Go back to the browser. You should still have the page of your Edge
        device opened.

    -   Click *Refresh* (on the top right on the page, not the browser
        refresh!) to load the current list of modules.

    -   The list of modules on the bottom should show now your opc-publisher
        and its status as running. Click on the module.

        ![](./media/image39.png)

    -   Click on *Direct Method* to open the screen to call methods on the
        module.

        ![](./media/image40.png)

    -   Via the Direct Method's payload, we provide a JSON which contains
        the OPC UA server which we want to connect to (field: EndpointUrl).
        Furthermore, it contains a list of all the OPC nodes on that server,
        that we would like to read and publish to the IoT Hub. Those are
        listed in the format "Id": "ns={namespace};i={index}"

        For more optional parameters you can take a look at the [OPC
        Publisher module
        documentation](https://github.com/Azure/iot-edge-opc-publisher#configuration-of-the-opc-ua-nodes-to-publish).

        Enter the following values and **make sure to put in your OPC UA server IP address**, which you have copied in the previous step.       

        Method Name: **PublishNodes**

        Payload:
        ```
        {
            "EndpointUrl": "opc.tcp://{YOUR-OPC-UA-SERVER-IP-ADDRESS}:50000",
            "OpcNodes": [{
                    "Id": "ns=2;s=DipData"
                }, {
                    "Id": "ns=2;s=SpikeData"
                }, {
                    "Id": "ns=2;s=PositiveTrendData"
                }
            ]
        }
        ```

    -   Then hit *Invoke Method* on the top

    What happens here is that you subscribe to specific nodes of the
    information model of the respective OPC UA server. An information model
    in the context of OPC UA consists of nodes and references that describes
    the relationships and actions between those nodes. Those nodes are
    referenced via their NodeId and can contain different information or may
    invoke actions on the OPC UA server. In our case, the nodes we subscribe
    to deliver data from our process in regular intervals.

    -   In the result window on the bottom you should see a similar output,
    confirming that the new nodes have been configured in the
    opc-publisher:

        ![](./media/image41.tmp)

    -   Now our publisher is all configured and ready-to-go. Switch to your
    PuTTY window and you should see a couple of new messages that the
    publisher is trying to connect to the OPC server.

        ![](./media/image42.png)

    -   The connection will not be established just yet, as the OPC server
        still needs to accept your module's certificate as a security means.
        **Please let your proctors know, that you have reached this step**
        and they will accept your module certificate on the OPC server.

    -   Once the OPC server lets your module connect, you should see more
        log messages from the publisher output. But we don't need to rely on
        those. Instead we can directly see our data in TSI.

1. **Validate data flow**

    You have almost completed this lab! Our last step is to validate
    that we can see our data.

    -   Go once again to your Time Series Insights environment.

    -   Hit the refresh button of your browser, not the one in the TSI
        screen! This will make sure you are seeing data up to the current
        time.

    -   You should now see a bigger spike of events.

        ![](./media/image43.png)

    -   To separate your OPC events from the simulated temperature data,
        click on the *SPLIT BY* dropdown on the left side. Select
        *iothub-connection-module-id*. This separates the incoming events by
        Edge module.

        ![](./media/image44.png)

    -   You should see two different event sources now in the middle chart.
        One for the simulated temp sensor and one for the OPC UA Publisher.

        ![](./media/image45.png)

    -   Right-click on the one of the opc-publisher and select *Show only
        this series*.

        ![](./media/image46.png)

    -   This will now filter to the OPC events. You can again right-click
        the graph to *Explore Events*, to see the raw data.

        ![](./media/image47.png)

        Notice fields like *ApplicationUri*, *Value.SourceTimestamp* or *Value.Value*. Those contain the data as prescribed in the  OPC UA standard.
  
        ![](./media/image48.png)

**Congratulations!** You have successfully completed the lab and connected
your first PLC data via OPC UA and data is flowing into the cloud! Your first steps into the world of Industrial IoT are done!

Clean up resources
-------------

To save unnecessary  costs, we recommend you dispose of your newly created resources once you do not need them anymore. To do this, you can simply delete the entire resource group from the Azure Portal.


Feedback
-------------

If you have any feedback about the lab - positive as well as negative - or run into any issues, please open an Issue here on the GitHub page and we will try to help you as soon as possible.