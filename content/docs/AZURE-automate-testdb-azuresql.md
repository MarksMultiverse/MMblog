+++
title = 'Automate creation of test databases on Azure SQL with Active Directory security groups'
date = 2023-09-11T12:00:46+02:00
draft = false
tags = ["Azure", "Automation", "Azure SQL"]
categories = ["Azure"]
+++

For database testing it is customary to make periodic copies of the production database and use the latest copy as a test database. But how does that work in a scenario where you use Azure SQL and a traditional Active Directory instance? And on top of that you obviously want to automate that process!

## Steps

Steps involved:
-	Create an Automation Account.
-	Give the automation account the right permissions.
-	Make the automation account administrator on the SQL server.
-	Check and add modules.
-   Set the variables in the automation account.
-	Create a runbook.
-	Schedule the runbook.

This is not a step-by-step tutorial but more a high over instuction.

## Automation Account

First you have to create a standard automation account in Azure. Nothing special. Nowadays a system assigned identity is automatically created when creatin an automation account. Great! 
![Automation Account](/AZURE-automate-testdb-sql/aa-acount.jpg)
Now we have to make sure that we give it the right permissions under Azure role assignments. I made the automation account `Owner` on the subscription. That did the trick for me, because I also would like to experiment with other SQL servers in the future. Please consider giving the account other permissions in your production environment on the basis of least privilege principle.
![Azure role assignment](/AZURE-automate-testdb-sql/Azure-role-assignment.jpg)
Make the automation account `admin` on the SQL server itself. This has to be done or else the automation account cannot alter the permissions of the database.
![Make the account admin](/AZURE-automate-testdb-sql/admin.jpg)
Please make sure to check if the `Az.Sql` module is installed on the automation account. This should be the case. But we need another module to make the changes to the database itself. You need to install the `sqlserver` module by hand.
![Install the right modules](/AZURE-automate-testdb-sql/module.jpg)

And now the fun part. The actual coding of the solution. First we have to fill the variables in the automation account.

![Variables](/AZURE-automate-testdb-sql/variables.jpg)

| Name | Comment |
| - | - |
| subscriptionId | The Subscription ID of the active subscription of the resources. |
| sourceResourceGroupName | The resource group name of the source database. |
| sourceServerName | The name of the SQL server of the source database. |
| sourceDBname | The name of the source database. |
| targetResourceGroupname | The resource group name of the target database. |
| targetServerName | The name of the SQL server of the target database. |
| targetDBname | The name of the target database. |
| ServerFQDN | The full name of the SQL server of the target database (.database.windows.net). |
| Username | Service account used to set the permissions on the target database. |
| Password | The password of the service account used. |

*Please note that the password variable is saved as encrypted.*

## Runbook

Create a new runbook with type `Powershell` and use the code that is mentioned below.
First we set the variables we just set up in the automation account.

```powershell
#Set variables
$subscriptionId = Get-AutomationVariable -Name 'subscriptionId'
$sourceResourceGroupName = Get-AutomationVariable -Name 'sourceResourceGroupName'
$sourceServerName = Get-AutomationVariable -Name 'sourceServerName'
$sourceDBname = Get-AutomationVariable -Name 'sourceDBname'
$targetResourceGroupname = Get-AutomationVariable -Name 'targetResourceGroupname'
$targetServerName = Get-AutomationVariable -Name 'targetServerName'
$targetDBname = Get-AutomationVariable -Name 'targetDBname'
$ServerFQDN = Get-AutomationVariable -Name 'ServerFQDN'
$Username = Get-AutomationVariable -Name 'Username'
$Password = Get-AutomationVariable -Name 'Password'
```

Of course we have to make ourself known within the environment and select the right subscription.

```powershell
#Connect to Azure
Connect-AzAccount -Identity
```

```powershell
#Set subscrition
Set-AzContext -SubscriptionId $subscriptionId
```

Remember the module we checked and added to the automation account? Those need to be imported.

```powershell
#Importing module
Import-Module Az.Sql -Force
Import-Module sqlserver
```

Then we want to make a copy of the PROD database to use as our TEST database. But if the TEST database already exist we first want to delete this database before we make a fresh copy of the database. This way we assure ourselves the we always test on the latest version.

```powershell
#Check if database is available. If not make new database. If present delete database and make new one
$OldDatabase = Get-AzSqlDatabase -ResourceGroupName $sourceResourceGroupName -ServerName $sourceServerName -DatabaseName $targetDBname -ErrorAction SilentlyContinue

if ($OldDatabase -eq $null){
    $NewDatabase = New-AzSqlDatabaseCopy -ResourceGroupName $sourceResourceGroupName `
                    -ServerName $sourceServerName `
                    -DatabaseName $sourceDBname `
                    -CopyResourceGroupName $targetResourceGroupname `
                    -CopyServerName $targetServerName `
                    -CopyDatabaseName $targetDBname
}
else{
    $DeleteDatabase = Remove-AzSqlDatabase -ResourceGroupName $sourceResourceGroupName -ServerName $sourceServerName -DatabaseName $targetDBname
    
    Start-Sleep -Seconds 30
    
    $NewDatabase = New-AzSqlDatabaseCopy -ResourceGroupName $sourceResourceGroupName `
                    -ServerName $sourceServerName `
                    -DatabaseName $sourceDBname `
                    -CopyResourceGroupName $targetResourceGroupname `
                    -CopyServerName $targetServerName `
                    -CopyDatabaseName $targetDBname
}
```
Now we have a new copy of the PROD database as our TEST database. All we have to do now is to add the AD security group for test purposes (Owner and Reader) and delete the AD security groups used for the PROD database (Owner and Reader).\
First we need a token to access the SQL server.

```powershell
#Get token to access the SQL server
$token = (Get-AzAccessToken -ResourceUrl https://database.windows.net).Token
```

And finally we define the query where we add and delete the security groups and run it.

```powershell
#Set queries
$Params = @"    
                CREATE USER [SQL-Owner-Users-Test] FROM EXTERNAL PROVIDER;
                ALTER ROLE db_owner ADD MEMBER [SQL-Owner-Users-Test];
                CREATE USER [SQL-Reader-Users-Test] FROM EXTERNAL PROVIDER;
                ALTER ROLE db_datareader ADD MEMBER [SQL-Reader-Users-Test];
                DROP USER [SQL-Owner-Users-Prod];
                DROP USER [SQL-Reader-Users-Prod]
"@
```

```powershell
#Execute queries on target database
Invoke-Sqlcmd -ServerInstance $ServerFQDN -Database $targetDBname -AccessToken $token -Username $Username -Password $Password -Query $Params
```
Save, publish and test the runbook. Make sure the service account has the right permissions to enter and alter databases on the SQL server.

To top it all of you could add a schedule so you have a fresh test database on the moment that you prefer.
![Schedule the runbook](/AZURE-automate-testdb-sql/schedule1.jpg)

That's all there is to it!

Full runbook Powershell script:
```Powershell
#Connect to Azure
Connect-AzAccount -Identity

#Set subscrition
Set-AzContext -SubscriptionId $subscriptionId

#Importing module
Import-Module Az.Sql -Force
Import-Module sqlserver

#Check if database is available. If not make new database. If present delete database and make new one
$OldDatabase = Get-AzSqlDatabase -ResourceGroupName $sourceResourceGroupName -ServerName $sourceServerName -DatabaseName $targetDBname -ErrorAction SilentlyContinue

if ($OldDatabase -eq $null){
    $NewDatabase = New-AzSqlDatabaseCopy -ResourceGroupName $sourceResourceGroupName `
                    -ServerName $sourceServerName `
                    -DatabaseName $sourceDBname `
                    -CopyResourceGroupName $targetResourceGroupname `
                    -CopyServerName $targetServerName `
                    -CopyDatabaseName $targetDBname
}
else{
    $DeleteDatabase = Remove-AzSqlDatabase -ResourceGroupName $sourceResourceGroupName -ServerName $sourceServerName -DatabaseName $targetDBname
    
    Start-Sleep -Seconds 30
    
    $NewDatabase = New-AzSqlDatabaseCopy -ResourceGroupName $sourceResourceGroupName `
                    -ServerName $sourceServerName `
                    -DatabaseName $sourceDBname `
                    -CopyResourceGroupName $targetResourceGroupname `
                    -CopyServerName $targetServerName `
                    -CopyDatabaseName $targetDBname
}

#Get token to access the SQL server
$token = (Get-AzAccessToken -ResourceUrl https://database.windows.net).Token

#Set queries
$Params = @"    
                CREATE USER [SQL-Owner-Users-Test] FROM EXTERNAL PROVIDER;
                ALTER ROLE db_owner ADD MEMBER [SQL-Owner-Users-Test];
                CREATE USER [SQL-Reader-Users-Test] FROM EXTERNAL PROVIDER;
                ALTER ROLE db_datareader ADD MEMBER [SQL-Reader-Users-Test];
                DROP USER [SQL-Owner-Users-Prod];
                DROP USER [SQL-Reader-Users-Prod]
"@

#Execute queries on target database
Invoke-Sqlcmd -ServerInstance $ServerFQDN -Database $targetDBname -AccessToken $token -Username $Username -Password $Password -Query $Params
```