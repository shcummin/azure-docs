﻿---
title: Use a Windows VM system-assigned managed identity  to access Azure SQL
description: A tutorial that walks you through the process of using a Windows VM system-assigned managed identity to access Azure SQL. 
services: active-directory
documentationcenter: ''
author: daveba
manager: mtillman
editor: bryanla

ms.service: active-directory
ms.component: msi
ms.devlang: na
ms.topic: tutorial
ms.tgt_pltfrm: na
ms.workload: identity
ms.date: 11/07/2018
ms.author: daveba
---


# Tutorial: Use a Windows VM system-assigned managed identity to access Azure SQL

[!INCLUDE [preview-notice](../../../includes/active-directory-msi-preview-notice.md)]

This tutorial shows you how to use a system-assigned identity for a Windows virtual machine (VM) to access an Azure SQL server. Managed Service Identities are automatically managed by Azure and enable you to authenticate to services that support Azure AD authentication, without needing to insert credentials into your code. You learn how to:

> [!div class="checklist"]
> * Grant your VM access to an Azure SQL server
> * Enable Azure AD authentication for the SQL server
> * Create a contained user in the database that represents the VM's system assigned identity
> * Get an access token using the VM identity and use it to query an Azure SQL server

## Prerequisites

[!INCLUDE [msi-qs-configure-prereqs](../../../includes/active-directory-msi-qs-configure-prereqs.md)]

[!INCLUDE [msi-tut-prereqs](../../../includes/active-directory-msi-tut-prereqs.md)]

- [Sign in to Azure portal](https://portal.azure.com)

- [Create a Windows virtual machine](/azure/virtual-machines/windows/quick-create-portal)

- [Enable system-assigned managed identity on your virtual machine](/azure/active-directory/managed-service-identity/qs-configure-portal-windows-vm#enable-system-assigned-identity-on-an-existing-vm)

## Grant your VM access to a database in an Azure SQL server

To grant your VM access to a database in an Azure SQL Server, you can use an existing SQL server or create a new one.  To create a new server and database using the Azure portal, follow this [Azure SQL quickstart](https://docs.microsoft.com/azure/sql-database/sql-database-get-started-portal). There are also quickstarts that use the Azure CLI and Azure PowerShell in the [Azure SQL documentation](https://docs.microsoft.com/azure/sql-database/).

There are two steps to granting your VM access to a database:

1.  Enable Azure AD authentication for the SQL server.
2.  Create a **contained user** in the database that represents the VM's system-assigned identity.

## Enable Azure AD authentication for the SQL server

[Configure Azure AD authentication for the SQL server](/azure/sql-database/sql-database-aad-authentication-configure#provision-an-azure-active-directory-administrator-for-your-azure-sql-server) using the following steps:

1.	In the Azure portal, select **SQL servers** from the left-hand navigation.
2.	Click the SQL server to be enabled for Azure AD authentication.
3.	In the **Settings** section of the blade, click **Active Directory admin**.
4.	In the command bar, click **Set admin**.
5.	Select an Azure AD user account to be made an administrator of the server, and click **Select.**
6.	In the command bar, click **Save.**

## Create a contained user in the database that represents the VM's system assigned identity

For this next step, you will need [Microsoft SQL Server Management Studio](https://docs.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) (SSMS). Before beginning, it may also be helpful to review the following articles for background on Azure AD integration:

- [Universal Authentication with SQL Database and SQL Data Warehouse (SSMS support for MFA)](/azure/sql-database/sql-database-ssms-mfa-authentication)
- [Configure and manage Azure Active Directory authentication with SQL Database or SQL Data Warehouse](/azure/sql-database/sql-database-aad-authentication-configure)

1.  Start SQL Server Management Studio.
2.  In the **Connect to Server** dialog, Enter your SQL server name in the **Server name** field.
3.  In the **Authentication** field, select **Active Directory - Universal with MFA support**.
4.  In the **User name** field, enter the name of the Azure AD account that you set as the server administrator, for example, helen@woodgroveonline.com
5.  Click **Options**.
6.  In the **Connect to database** field, enter the name of the non-system database you want to configure.
7.  Click **Connect**.  Complete the sign-in process.
8.  In the **Object Explorer**, expand the **Databases** folder.
9.  Right-click on a user database and click **New query**.
10. In the query window, enter the following line, and click **Execute** in the toolbar:

    > [!NOTE]
    > `VMName` in the following command is the name of the VM that you enabled system assigned identity on in the prerequsites section.
    
     ```
     CREATE USER [VMName] FROM EXTERNAL PROVIDER
     ```
    
     The command should complete successfully, creating the contained user for the VM's system-assigned identity.
11.  Clear the query window, enter the following line, and click **Execute** in the toolbar:

    > [!NOTE]
    > `VMName` in the following command is the name of the VM that you enabled system assigned identity on in the prerequsites section.
     
     ```
     ALTER ROLE db_datareader ADD MEMBER [VMName]
     ```

     The command should complete successfully, granting the contained user the ability to read the entire database.

Code running in the VM can now get a token using its system-assigned managed identity and use the token to authenticate to the SQL server.

## Get an access token using the VM's system-assigned managed identity and use it to call Azure SQL 

Azure SQL natively supports Azure AD authentication, so it can directly accept access tokens obtained using managed identities for Azure resources.  You use the **access token** method of creating a connection to SQL.  This is part of Azure SQL's integration with Azure AD, and is different from supplying credentials on the connection string.

Here's a .Net code example of opening a connection to SQL using an access token.  This code must run on the VM to be able to access the VM's system-assigned managed identity's endpoint.  **.Net Framework 4.6** or higher is required to use the access token method.  Replace the values of AZURE-SQL-SERVERNAME and DATABASE accordingly.  Note the resource ID for Azure SQL is "https://database.windows.net/".

```csharp
using System.Net;
using System.IO;
using System.Data.SqlClient;
using System.Web.Script.Serialization;

//
// Get an access token for SQL.
//
HttpWebRequest request = (HttpWebRequest)WebRequest.Create("http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://database.windows.net/");
request.Headers["Metadata"] = "true";
request.Method = "GET";
string accessToken = null;

try
{
    // Call managed identities for Azure resources endpoint.
    HttpWebResponse response = (HttpWebResponse)request.GetResponse();

    // Pipe response Stream to a StreamReader and extract access token.
    StreamReader streamResponse = new StreamReader(response.GetResponseStream()); 
    string stringResponse = streamResponse.ReadToEnd();
    JavaScriptSerializer j = new JavaScriptSerializer();
    Dictionary<string, string> list = (Dictionary<string, string>) j.Deserialize(stringResponse, typeof(Dictionary<string, string>));
    accessToken = list["access_token"];
}
catch (Exception e)
{
    string errorText = String.Format("{0} \n\n{1}", e.Message, e.InnerException != null ? e.InnerException.Message : "Acquire token failed");
}

//
// Open a connection to the SQL server using the access token.
//
if (accessToken != null) {
    string connectionString = "Data Source=<AZURE-SQL-SERVERNAME>; Initial Catalog=<DATABASE>;";
    SqlConnection conn = new SqlConnection(connectionString);
    conn.AccessToken = accessToken;
    conn.Open();
}
```

Alternatively, a quick way to test the end to end setup without having to write and deploy an app on the VM is using PowerShell.

1.	In the portal, navigate to **Virtual Machines** and go to your Windows virtual machine and in the **Overview**, click **Connect**. 
2.	Enter in your **Username** and **Password** for which you added when you created the Windows VM. 
3.	Now that you have created a **Remote Desktop Connection** with the virtual machine, open **PowerShell** in the remote session. 
4.	Using PowerShell’s `Invoke-WebRequest`, make a request to the local managed identity's endpoint to get an access token for Azure SQL.

    ```powershell
       $response = Invoke-WebRequest -Uri 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fdatabase.windows.net%2F' -Method GET -Headers @{Metadata="true"}
    ```
    
    Convert the response from a JSON object to a PowerShell object. 
    
    ```powershell
    $content = $response.Content | ConvertFrom-Json
    ```

    Extract the access token from the response.
    
    ```powershell
    $AccessToken = $content.access_token
    ```

5.  Open a connection to the SQL server. Remember to replace the values for AZURE-SQL-SERVERNAME and DATABASE.
    
    ```powershell
    $SqlConnection = New-Object System.Data.SqlClient.SqlConnection
    $SqlConnection.ConnectionString = "Data Source = <AZURE-SQL-SERVERNAME>; Initial Catalog = <DATABASE>"
    $SqlConnection.AccessToken = $AccessToken
    $SqlConnection.Open()
    ```

    Next, create and send a query to the server.  Remember to replace the value for TABLE.

    ```powershell
    $SqlCmd = New-Object System.Data.SqlClient.SqlCommand
    $SqlCmd.CommandText = "SELECT * from <TABLE>;"
    $SqlCmd.Connection = $SqlConnection
    $SqlAdapter = New-Object System.Data.SqlClient.SqlDataAdapter
    $SqlAdapter.SelectCommand = $SqlCmd
    $DataSet = New-Object System.Data.DataSet
    $SqlAdapter.Fill($DataSet)
    ```

Examine the value of `$DataSet.Tables[0]` to view the results of the query.

## Next steps

In this tutorial, you learned how to use a system-assigned managed identity to access Azure SQL server.  To learn more about Azure SQL Server see:

> [!div class="nextstepaction"]
>[Azure SQL Database service](/azure/sql-database/sql-database-technical-overview)
