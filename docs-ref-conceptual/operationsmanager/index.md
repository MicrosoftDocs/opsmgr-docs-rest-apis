---
title: Operations Manager REST API Reference
description: This article describes the System Center Operations Manager REST API reference content.  
author: mgoedtel
ms.author: magoedte
manager: rayoflores
ms.date: 04/22/2020
ms.custom: na
ms.service: system-center
ms.subservice: operations-manager
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

```powershell
#Set Headers for the request
$userName ="domain\username";
$password ="Password";
$AuthenticationMode= "Network"
$scomHeaders = New-Object “System.Collections.Generic.Dictionary[[String],[String]]”
$scomHeaders.Add('Content-Type','application/json; charset=utf-8')
$bodyraw = "($AuthenticationMode):$($userName):$($password)”
$Bytes = [System.Text.Encoding]::UTF8.GetBytes($bodyraw)
$EncodedText =[Convert]::ToBase64String($Bytes)
$jsonbody = $EncodedText | ConvertTo-Json

$uriBase = 'http://<scomserver>/OperationsManager'

#Authenticate
$auth = Invoke-WebRequest -Method POST -Uri $uriBase/authenticate -Headers $scomheaders -body $jsonbody -UseDefaultCredentials -SessionVariable $websession 
#Include CSRF Token
$csrfTocken = $websession.Cookies.GetCookies($uriBase) | ? { $_.Name -eq 'SCOM-CSRF-TOKEN' }
$scomHeaders.Add('SCOM-CSRF-TOKEN', [System.Web.HttpUtility]::UrlDecode($csrfTocken.Value))

#Examples to Invoke the required SCOM API

#Data/alertDescription/{alertid}
$alertid="6A88FBC1-0EDC-4173-9BB1-30FFEC296672"
$uri = "$uriBase/data/alertDetails"
$Response = Invoke-WebRequest -Uri "$uriBase/data/alertDetails/$alertid" -Headers $scomheaders -Method Get -WebSession $websession
$Response.Content | Convertto-json



#Criteria: Enter the displayname of the SCOM object
$Criteria = "DisplayName LIKE '%SQL%'"
#Convert our criteria to JSON format
$JSONBody = $Criteria | ConvertTo-Json
$Response = Invoke-WebRequest -Uri "$uriBase/data/class/monitors" -Method Post -Body $JSONBody -WebSession $WebSession
$Response.Content | Convertto-json


#Data Request in JSON Format
$dashboardbody='
{
  "refreshing": false,
  "Name": "TestAPIDashboard",
  "path": "35b6eba3-6202-8738-a47f-16acf476230f"
}'
$response = Invoke-WebRequest -Uri "$uriBase/monitoring/dashboard" -Headers $scomheaders -Method POST -Body $dashboardbody -ContentType "application/json" -UseDefaultCredentials -WebSession $websession

```
