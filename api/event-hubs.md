---
description: Receive client events from the PowerShell Universal server.
---

# Event Hubs

Event Hubs provide the ability to connect client to the PowerShell Universal server. Once connected, the PowerShell Universal server can send messages to the connected clients and they will run a local PowerShell script block.&#x20;

## Creating an Event Hub

To create an event hub, click APIs \ Event Hub and click Create New Event Hub. Event Hubs are named and can choose to enforce authentication and authorization.

## Connecting an Event Hub

{% hint style="info" %}
We recommend using the event hub client over the `Connect-PSUEventHub` cmdlet for persistent connections.&#x20;
{% endhint %}

Once created, clients can connect to an event hub using the `Connect-PSUEventHub` cmdlet found in the Universal module. The cmdlet connects to the hub using a web socket and provides credentials, if necessary. When connecting, specify the `-ScriptBlock` parameter to define what will happen on the client when an event is received.&#x20;

```powershell
Connect-PSUEventHub -ComputerName http://localhost:5000 -Hub 'MyHub' -ScriptBlock {
     Write-Host "Event Received"
}
```

Objects sent from the hub will be available as `$_` or $`PSItem`.&#x20;

```powershell
Connect-PSUEventHub -ComputerName http://localhost:5000 -Hub 'MyHub' -ScriptBlock {
     Write-Host $_
}
```

## Send Events

From within the PowerShell Universal server, you can send events from a hub to connected clients using the `Send-PSUEvent` cmdlet.&#x20;

```powershell
Send-PSUEvent -Hub 'MyHub' -Data "Hello!"
```

The `-Data` parameter accepts an object and will be serialized using CLIXML and send to the client. The data will be deserialized before passing to the script block.&#x20;

## Receive Data from Clients

As of 4.1, you can now receive data back from clients. This feature is only available when sending data to an individual client, rather than all clients connected to a hub.&#x20;

```powershell
$Connection = Get-PSUEventHubConnection | Where-Object UserName -eq 'Admin'
$Result = Send-PSUEvent -Hub 'Hub' -Data 'Say Hello!' -Connectionid $Connection.ConnectionId
Show-UDToast $Result
```

From the client side, you would return the data from the script block.&#x20;

```powershell
Connect-PSUEventHub -Hub 'Hub' -ScriptBlock {
    Write-Host $EventData 
    "Hello!"
}
```

## Event Hub Client Installer

While you can use the Universal module to connect to event hubs, it may not be the most resilient configuration for your environment. The Event Hub Client Installer provides a MSI that installs a Windows services that will connect to event hubs and run scripts.

### eventHubClient.json

After installing the MSI, you will need to configure the client by using an `eventHubClient.json` file. This file should be created in `%ProgramData%\PowerShellUniversal`. Changes to this file require a restart of the Event Hub Client service.

{% hint style="warning" %}
The installer will not create the folder or file automatically.&#x20;
{% endhint %}

This JSON file configures the Event Hub Client to connect to the hub and run scripts when invoked.&#x20;

```json
{
    "Connections": [
        {
            "Url": "http://localhost:5000",
            "Hub": "eventHub",
            "AppToken": "tokenXyz",
            "ScriptPath": "script.ps1"
        }
    ]
}
```

### Options

The below options can be included in the `eventHubClient.json` file.&#x20;

#### Connections

These are the connections to Event Hubs. Each connection can contain it's own URL, hub, authentication and script to execute.&#x20;

#### Url

The URL of the PowerShell Universal service.&#x20;

#### Hub

The name of the Event Hub.

#### AppToken

The app token used to authentication against the hub.&#x20;

#### UseDefaultCredentials

Windows Authentication will be used to authenticate against the hub.

#### ScriptPath

The script to execute when an event is received. This script is read into memory and not from disk. Variables such as `$PSScriptRoot` are currently not supported.

## Example: Running Scripts on Remote Machines <a href="#example-running-scripts-on-remote-machines" id="example-running-scripts-on-remote-machines"></a>

This example provides a way to run scripts on remote machines without having to install another instance of PowerShell Universal.

This example allows for sending scripts to remote machines and executing them with a generic event hub script.

First, create an event hub in PowerShell Universal. This example does not use authentication.

### eventHubClient.json

Next, install the Event Hub Client on the remote machine. Create a configuration file in `%ProgramData%\PowerShellUniversal\eventHubClient.json`.

```json
{
    "Connections": [
        {
            "Url": "http://localhost:5000",
            "Hub": "eventHub",
            "ScriptPath": "script.ps1"
        }
    ]
}
```

### scripts.ps1

Next, create a helper script.ps1 to receive the event hub data and process requests from PSU to invoke scripts. It creates a new scriptblock and uses the `$EventData` passed down from the event hub message with the `.Contents` and `.Parameters` for the script.

```powershell
Write-Host $EventData.Hello
$scriptBlock = [Scriptblock]::Create($EventData.Contents)
$Parameters = $EventData.Parameters
& $scriptBlock @Parameters
```

### testEventHub.ps1

In PowerShell Universal, add an automation script `testEventHub.ps1`.

This script will send the event down to the client. This could be from an API or an App as well. It uses `Get-PSUEventHubConnection` to get the target computer’s connection ID and then sends an event with the contents of a script and any parameters for that script. The script on event hub side is generic and it will just run whatever is passed to it. Note: `-Data` maps to `$EventData` on the client.

```powershell
param($TargetComputer, $ProcessName)

$scriptBlock = {
    param($Name)
    Start-Process $Name
}

$Connection = Get-PSUEventHubConnection | Where-Object { $_.Computer -eq $TargetComputer -and -not $_.Disconnected } | Select-Object -First 1

Send-PSUEvent -Hub eventHub -ConnectionId $Connection.ConnectionId -Data @{
    Contents = $scriptBlock.ToString() # This will populate $EventData.Contents in the clients script.ps1.
    Parameters = @{
        Name = $ProcessName
    } # This will populate $EventData.Parameters in the clients script.ps1.
    Hello = "Client!" # Having some fun.
}
```

Another way to populate the `.Contents` is by using a script file to get code from.

```powershell
    Contents = Get-Content StartAProcess.ps1 -Raw
```

From here you could event use the script to schedule jobs to run on the remote machines using the event hub client.
