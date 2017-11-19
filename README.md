# azure-datacenter-ip-ranges-api
This project contains a lighweight manner to create an API that return the Azure Datacenter IP address ranges 
Get Azure Datacenter IP ranges via API

Introduction
One of the struggles that we may face when dealing with Azure services, is the network filtering. Today, we consume a lot of services provided by Azure, and some Azure services can consume services from our infrastructure. Microsoft is publishing an ‘xml’ file that contains the exhaustive list of the Azure Datacenter IP ranges for all its public regions (No gov). This ‘xml’ file is regularly updated by Microsoft to reflect the list updates as they can add or remove ranges.
The problem is that consuming an ‘xml’ file is not very convenient as we need to make many transformations to consume the content. Many of you have requested that Microsoft at least publishes this list via an API so it can be consumed by many sources, using Rest requests. Until Microsoft make it, I will show you today how to create a very lightweight web app on Azure (using Azure Functions) that will ‘magically’ do the job.
NB : The Azure Datacenter IP ranges include all the address space used by the Azure datacenters, including the customers address space
1-	Why do we need these address ranges ?
If you want to consume Azure services or some Azure services want to consume your services, and you don’t want to allow all the “Internet” space, you can ‘reduce’ the allowed ranges to only the Azure DC address space. In addition, you can go further and select the address ranges by Azure region in case the Azure services are from a particular region. 
Examples :	
o	Some applications in your servers need to consume Azure Web Apps hosted on Azure. In a world without the Azure DC address space, you should allow them to access  internet, which is a bad idea. You can configure your firewalls to only permit access to the Azure DC IP ranges
o	If you are using Network Virtual Appliances on Azure, and you want to allow the VM’s Azure agent to access the Azure services (storage accounts) in order to function properly, you can allow access only to the Azure DC IPs instead of internet.

2-	The Solution
In order to consume the Azure Datacenter IPs via an API, I used the powerful and simple Azure functions to provide a very light weight ‘File’ to ‘JSON’ converter. The Azure function will do the following:
o	Accept only a POST request
o	Download the Azure Datacenter IP ranges xml file
o	Convert it to a JSON format
o	Return an output based on the request: 
o	A POST request can accept a Body of the following format : {   "region": "regionname", "request": "requesttype" }. 
	The <request> parameter can have the value of :
•	dcip : This will return the list of the Azure Datacenter IP ranges, depending on the <regionname> parameter. <regionname> can be :
o	all : This will  return a JSON output of all the Azure Datacenter IP ranges of all regions
o	regionname : This will  return a JSON output of the regionname’s Azure Datacenter IP ranges
•	dcnames : This will return al list of the Azure Datacenter region’s names. The <region> parameter will be ignored in this case
o	In case of a bad region name or request value, an error will be returned

3-	Try it before deploying it
If you want to see the result, you can make the following requests using your favorite tool, against an Azure function hosted on my platform. In my case, I’m using powershell and my function Uri is https://azuredcip.azurewebsites.net/api/azuredcipranges
3.1- Get the address ranges of all the Azure DCs regions
#Powershell code
$body =  @{"region"="all";"request"="dcip"} | ConvertTo-Json

$webrequest  = Invoke-WebRequest -Method "POST" -uri https://azuredcip.azurewebsites.net/api/azuredcipranges -Body $body

ConvertFrom-Json -InputObject $webrequest.Content  

 

3.1- Get the address ranges of the North Europe region
In this case, note that we must use europnorth instead of northeurope
#Powershell code
$body =  @{"region"="europenorth";"request"="dcip"} | ConvertTo-Json

$webrequest  = Invoke-WebRequest -Method "POST" -uri https://azuredcip.azurewebsites.net/api/azuredcipranges -Body $body

ConvertFrom-Json -InputObject $webrequest.Content  

 

3.1- Get the region names
$body =  @{"request"="dcnames"} | ConvertTo-Json
#or
#$body =  @{"region"="anything";"request"="dcip"} | ConvertTo-Json 

$webrequest  = Invoke-WebRequest -Method "POST" -uri https://azuredcip.azurewebsites.net/api/azuredcipranges -Body $body

ConvertFrom-Json -InputObject $webrequest.Content  

 

4-	How to deploy it on your system ?
In order to deploy this function within your infrastructure, you will need to create a Azure function (Section : Create a function app) within your infrastructure. You can use an existing Function App or App Service Plan.
NB : The App Service Plan OS must be Windows
After creating the Funtion App, do the following:
Step	Screenshot
Go to your Function App and click the Create new (+)	 
In Language chose Powersell then select HttpTrigger - Powershell	 
Give a Name to your function and choose an Authorization level.
In my case, i set the Authorization to anonymous in order to make the steps simpler. We will see later how to secure the access to the function http trigger	 

Copy paste the following code on the function tab, then click Save

# POST method: $req
$requestBody = Get-Content $req -Raw | ConvertFrom-Json
$region = $requestBody.region
$request = $requestBody.request
#Main
if (-not$region) {$region='all'}
$URi = "https://www.microsoft.com/en-us/download/confirmation.aspx?id=41653"
$downloadPage =     Invoke-WebRequest -Uri $URi  -usebasicparsing
$xmlFileUri =  ($downloadPage.RawContent.Split('"') -like "https://*PublicIps*")[0]
$response =   Invoke-WebRequest -Uri $xmlFileUri -usebasicparsing
[xml]$xmlResponse =     [System.Text.Encoding]::UTF8.GetString($response.Content)

$AzDcIpTab = @{}
if ($request -eq 'dcip')
{
foreach ($location in $xmlResponse.AzurePublicIpAddresses.Region)
{
if ($region -eq 'all') {$AzDcIpTab.Add($location.Name,$location.IpRange.Subnet)}
elseif ($region -eq $location.Name) {$AzDcIpTab.Add($location.Name,$location.IpRange.Subnet)}
}
if ($AzDcIpTab.Count -eq '0') {$AzDcIpTab.Add("error","the requested region does not exist")}
}
elseif ($request -eq 'dcnames')
{
$AzDcIpTab =  $xmlResponse.AzurePublicIpAddresses.Region.name
}
else
{$AzDcIpTab.Add("error","the request parameter is not valid")}

$AzDcIpJson =  $AzDcIpTab | ConvertTo-Json
Out-File -Encoding Ascii -FilePath $res -inputObject $AzDcIpJson 

	 
Go to the integrate tab, and choose the following:
o	Allowed HTTP methods : Selected methods
o	Selected HTTP methods : Keep only POST
Click Save	 
It’s done !
To test the function, go back to the main blade and develop the Test tab	 
On the request body, type :
{
    "region" : "europeewest",
    "request": "dcnames"
}
Then click Run

You should see the results on the output, and the http Status 200 OK	 
Click Get function URL to get the URL of your function in order to query it via an external tool	 
 

5-	Securing the access to the function URL
There are 3 options that let you secure the access to the Function URL:
1-	Network IP Restrictions : I personally think that this is best option to secure the access to Function URL. IP Restrictions allows you allow only a set of Public IP addresses to access the Function URL. For example, if you have an automation script that requests the API and update a Firewall object or a database, you can whitelist only the Public IP address used by this automation workflow, which is the outbound IP address. This feature is available for Basic Tier App Service Plans and greater. It’s not supported for free and shared sku. Start using it by following this tutorial : https://docs.microsoft.com/en-us/azure/app-service/app-service-ip-restrictions.
NB : The Networking blade for a Function App can be found by clicking on the Function Name  Platform features  Networking
 
2-	Function Key and Host Key : You can restrict the access to the Function URL by leveraging an Authorization feature, by protecting the URL requesting via a ‘Key’. There are two Key types : The Function Key which is defined per Funtion and a Hot key which is defined and the same for all the functions within a Function App. In order to protect a function via a Function key, do the following :
Step	Screenshot
Go to the Integrate blade, and change the Authorization level to Function	
 

Go to the Manage blade. You can use the default generated key or Add a new function key. Generating a new key is provided for keys rotation	 
Now, you can access the ‘protected’ URL by  clicking on the Get function URL on the main blade. You can select which key to use.	 

3-	You can alternatively use the Authentication and authorization feature in Azure App Service. This feature allows you to secure your Function App by requesting the caller to authenticate to an Identity Provider and provide a token. You can use it against Azure Active Directory or Facebook for example. I will not detail the steps in this post, but here’s some materials : 
o	Overview : https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/app-service/app-service-authentication-overview.md
o	How to configure your App Service application to use Azure Active Directory login : https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/app-service/app-service-mobile-how-to-configure-active-directory-authentication.md 



	



