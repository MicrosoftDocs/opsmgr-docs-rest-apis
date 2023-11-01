---
title: Operations Manager REST API Reference
description: This article describes the System Center Operations Manager REST API reference content.  
author: mgoedtel
ms.author: magoedte
manager: rayoflores
ms.date: 11/1/2023
ms.custom: na
ms.prod: system-center
ms.technology: operations-manager
ms.topic: reference
ms.assetid: bb4bd81c-439c-479d-a045-dd5653812dff
---

# Operations Manager REST API Reference

System Center Operations Manager provides flexible and cost-effective infrastructure monitoring that helps ensure predictable performance and availability of critical applications and infrastructure services. Operations Manager offers comprehensive monitoring for your datacenter by checking the health, performance, and availability of all monitored objects in your environment, and helping to identify problems for rapid resolution. For a more detailed overview, see the System Center Operations Manager [product page](/system-center/scom/).

> [!NOTE]
> Operations Manager 2019 UR1 supports Cross-Site Request Forgery (CSRF) tokens to prevent CSRF attacks. If you are using Operations Manager 2019 UR1, you must initialize the CSRF token. HTML scripts do not work if the CSRF tokens are not initialized.

## Initialize the CSRF token

Required action, applicable for Operations Manager 2019 UR1.

1. In the HTML header of the dashboard, add the following code:

```var requestHeaders = {
            Accept: 'q=0.8;application/json;q=0.9',
            "Content-Type": "application/json"
        };
        InitializeCSRFToken();
        function InitializeCSRFToken() {
            var documentcookies = document.cookie.split('; ');
            var result = {};
            for (var i = 0; i < documentcookies.length; i++) {
                var cur = documentcookies[i].split('=');
                result[cur[0]] = cur[1];
            }
            if (result["SCOM-CSRF-TOKEN"] && result["SCOM-CSRF-TOKEN"] != null) {
                requestHeaders["SCOM-CSRF-TOKEN"] = decodeURIComponent(result["SCOM-CSRF-TOKEN"]);
            }
        }
```

1. In the **onload** function, change the header value to **requestHeaders**.

**Example:**

![Initialize the CSRF token example](./Media/index/116854.png)

## Operations Manager REST API PowerShell example
The System Center Operations Manager (SCOM) API provides a powerful interface for automating and extending the capabilities of SCOM. This PowerShell script is designed to facilitate various operations by interacting with the SCOM API, simplifying the process of retrieving and managing data within your SCOM environment.

The script includes a set of primary functions that lay the groundwork for interacting with the API, such as initializing the base URL and setting up necessary authentication headers. It also includes a suite of optional functions for performing specific tasks, such as fetching the effective monitoring configuration for a particular object or querying for unsealed management packs.

Each function is crafted with parameters that allow for customization based on the user’s needs and will enable the script to be adapted to various SCOM environments.

The execution of the script is straightforward, with examples provided for authenticating using specific credentials or the current user's credentials. It also includes template code for outputting data in a structured JSON format for easy reading and integration with other systems or reporting tools.

### Functions
#### Primary Functions and Variable
```powershell
# Initialize SCOM API Base URL
$URIBase = 'http://<SCOMWebConsoleURL>/OperationsManager'

# Function to initialize HTTP headers and CSRF token for SCOM API
function Initialize-SCOMHeaders {
    $SCOMHeaders = @{
        'Content-Type' = 'application/json; charset=utf-8'
    }

    $CSRFtoken = $WebSession.Cookies.GetCookies($URIBase) | Where-Object { $_.Name -eq 'SCOM-CSRF-TOKEN' }
    $SCOMHeaders['SCOM-CSRF-TOKEN'] = [System.Web.HttpUtility]::UrlDecode($CSRFtoken.Value)
    return $SCOMHeaders
}

# Function to authenticate with the SCOM API
function Authenticate-SCOM {
    param (
        [PSCredential]$Credential = $null
    )

    $bodyRaw = "Windows"
    $encodedText = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($bodyRaw))
    $JSONBody = $encodedText | ConvertTo-Json

    $SCOMHeaders = Initialize-SCOMHeaders

    if ($Credential -ne $null) {
        Invoke-RestMethod -Method Post -Uri "$URIBase/authenticate" -Headers $SCOMHeaders -Body $JSONBody -Credential $Credential -SessionVariable WebSession
    } else {
        Invoke-RestMethod -Method Post -Uri "$URIBase/authenticate" -Headers $SCOMHeaders -Body $JSONBody -UseDefaultCredentials -SessionVariable WebSession
    }
}
```

#### Optional Functions
```powershell
# Function to fetch Effective Monitoring Configuration by GUID
function Get-EffectiveMonitoringConfiguration {
    param (
        [string]$guid
    )

    $uri = "$URIBase/effectiveMonitoringConfiguration/$guid`?isRecursive=True"
    $response = Invoke-WebRequest -Uri $uri -Method GET -WebSession $WebSession
    return $response.Content | ConvertFrom-Json
}

# Function to fetch Unsealed Management Packs
function Get-UnsealedManagementPacks {
    $uri = "$URIBase/data/UnsealedManagementPacks"
    $response = Invoke-WebRequest -Uri $uri -Method GET -WebSession $WebSession
    return $response.Content | ConvertFrom-Json
}

# Function to fetch the state of the Management Group
function Get-ManagementGroupState {
    $query = @(@{
        "classId"         = ""
        "criteria"        = "DisplayName = 'Operations Manager Management Group'"
        "displayColumns"  = "displayname", "healthstate", "name", "path"
    })

    $JSONQuery = $query | ConvertTo-Json
    $response = Invoke-RestMethod -Uri "$URIBase/data/state" -Method Post -Body $JSONQuery -ContentType "application/json" -WebSession $WebSession
    return $response.Rows
}

# Function to fetch all Windows Servers
function Get-WindowsServers {
    $criteria = "DisplayName LIKE 'Microsoft Windows Server%'"
    $JSONBody = $criteria | ConvertTo-Json

    $uri = "$URIBase/data/scomObjects"
    $response = Invoke-WebRequest -Uri $uri -Method Post -Body $JSONBody -WebSession $WebSession
    return ($response.Content | ConvertFrom-Json).scopeDatas
}
```

### Script to Execute
Prior to executing the script you will need to uncomment which line you will use for Authentication (lines 4-5, and line 8).
```powershell
# Main Execution

# Uncomment the 2 lines below if you want to authenticate using specific credentials
# $cred = Get-Credential
# Authenticate-SCOM -Credential $cred

# Uncomment the line below if you want to authenticate using the current user's credentials
# Authenticate-SCOM

#Write-Output "-----------------------------------------"

# Replace 'your-guid-here' with the actual GUID of the monitoring object
#$MonitoringObjectGUID = 'your-guid-here'

# Fetch the effective monitoring configuration for the given GUID
#$EffectiveConfig = Get-EffectiveMonitoringConfiguration -guid $MonitoringObjectGUID

# Output the effective monitoring configuration in JSON format
#Write-Output "Effective Monitoring Configuration:`n$($EffectiveConfig | ConvertTo-Json)"

Write-Output "-----------------------------------------"

# Fetch all Windows Agents
$WindowsServers = Get-WindowsServers
Write-Output "Windows Servers:`n$($WindowsServers | ConvertTo-Json)"

Write-Output "-----------------------------------------"

# Get Unsealed Management Packs
$unsealedMPs = Get-UnsealedManagementPacks
Write-Output "Unsealed MPs:`n$($unsealedMPs | ConvertTo-Json)"

Write-Output "-----------------------------------------"

# Get Management Group Health Status
$state = Get-ManagementGroupState
Write-Output "Monitored Computer State:`n$($state | ConvertTo-Json)"

Write-Output "-----------------------------------------"
```
