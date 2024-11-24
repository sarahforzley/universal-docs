---
description: Receive client events from the PowerShell Universal server.
---

# Event Hubs

{% hint style="info" %}
Event Hubs require a [license](../licensing.md).&#x20;
{% endhint %}

Event Hubs let you connect the client to the PowerShell Universal server. Once connected, the PowerShell Universal server can send messages to the connected clients so that they run a local PowerShell script block.

## Creating an Event Hub

To create an event hub, click APIs \ Event Hub and click Create New Event Hub. Event Hubs are named and can choose to enforce authentication and authorization.

## Agent

The agent process is responsible for responding to Event Hub requests. You can learn more about installing the agent on our [Installation page](../getting-started/).&#x20;

### agent.json

After installing the agent, you need to configure the client with an `agent.json` file. Create this file in `%ProgramData%\PowerShellUniversal`. Changes to this file require a restart of the Agent service.

{% hint style="warning" %}
The installer does not create the folder or file automatically.&#x20;
{% endhint %}

This JSON file configures the Agent to connect to the hub and run scripts when invoked.

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

The below options can be included in the `agent.json` file:

#### Connections

These are the connections to Event Hubs. Each connection can contain its own URL, hub, authentication and script to execute.

#### URL (Required)

The URL of the PowerShell Universal service.

#### Hub (Required)

The name of the Event Hub.

#### AppToken

The app token used to authenticate against the hub.

#### UseDefaultCredentials

Windows Authentication authenticates against the hub.

#### ScriptPath

The script to execute when an event is received. This script is read into memory and not from disk. Variables such as `$PSScriptRoot` are not currently supported. This is optional, as event hubs can also run commands directly.

## Send Events

From within the PowerShell Universal server, you can send events from a hub to connected clients using the `Send-PSUEvent` cmdlet.

```powershell
Send-PSUEvent -Hub 'MyHub' -Data "Hello!"
```

The `-Data` parameter accepts an object and is serialized using CLIXML, then sent to the client. The client deserializes the data before passing to the script block.

You can also run commands. This does not require defining a script on the event hub client. You can also use the `Invoke-PSUCommand` alias to mimic native PowerShell behavior.

```powershell
Invoke-PSUCommand -Hub "MyHub" -Command "Start-Process" -Parameters @{
    FilePath = "Notepad"
}
```

## Receive Data from Clients

This feature is only available when sending data to an individual client, rather than all clients connected to a hub.

```powershell
$Connection = Get-PSUEventHubConnection | Where-Object UserName -eq 'Admin'
$Result = Send-PSUEvent -Hub 'Hub' -Data 'Say Hello!' -Connectionid $Connection.ConnectionId
Show-UDToast $Result
```

## Example: Running Scripts on Remote Machines

{% hint style="info" %}
This example provides a way to run scripts on remote machines without installing another instance of PowerShell Universal.
{% endhint %}

This example allows you to send scripts to remote machines and execute them with a generic event hub script.&#x20;

First, create an event hub in PowerShell Universal.  This example does not use authentication.

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

Next, create a helper script.ps1 to receive the event hub data and process requests from PSU to invoke scripts. It creates a new temporary PS1 file and uses the `$EventData` passed down from the event hub message with the contents and parameters for the script.&#x20;

```powershell
$TempFile = (New-TemporaryFile).FullName + ".ps1"
$EventData.Contents | Out-File -FilePath $TempFile
$Parameters = $EventData.Parameters
& $TempFile @Parameters
```

In PowerShell Universal, add a script that you want to run on the remote machine. In this example, it simply starts a process.

```powershell
param($Name)

Start-Process $Name
```

Finally, add another script that sends the event down to the client. This could be from an API or an App. It uses `Get-PSUEventHubConnection` to get the target computer’s connection ID and then sends an event with the contents of a script and any parameters for that script. Because the script on the event hub side is generic, it will run whatever is passed to it.

```powershell
param($TargetComputer, $ProcessName)
 
$Connection = Get-PSUEventHubConnection | Where-Object { $_.Computer -eq $TargetComputer -and -not $_.Disconnected } | Select-Object -First 1
Send-PSUEvent -Hub eventHub -ConnectionId $Connection.ConnectionId -Data @{
    Contents = Get-Content StartAProcess.ps1 -Raw
    Parameters = @{
        Name = $ProcessName
    }
}
```

From here you could event use the script to schedule jobs to run on the remote machines using the event hub client.&#x20;



&#x20;
