## Set up an Application Gateway in front of Web App that has Azure AD authentication enabled   ##

Extension of this [blog](http://thewindowsupdate.com/2019/04/01/setting-up-application-gateway-with-an-app-service-that-uses-azure-active-directory-authentication/), following steps need to be executed in this github:

- 1. Configure web app
- 2. Configure application gateway

It is assumed the web app and application gateway are already deployed, in which the application gateway uses the web app as its backend. See this [link](https://docs.microsoft.com/en-us/azure/application-gateway/configure-web-app-portal) how to do this. Make also sure that PowerShell is installed with the Az modules.

### 1. Configure web app ###

The following steps need to be executed:

- 1a. Assign custom domain to web app
- 1b. Add SSL certificate to application gateway
- 1c. Turn on AAD authentication of web app
- 1d. Change callback URL in app registration

#### 1a. Assign custom domain to web app ####

In this [link](https://docs.microsoft.com/en-us/azure/app-service/manage-custom-dns-buy-domain), it is explained how a custom domain can be bought from an app service. Two remarks:

- GoDaddy is used as provider, domain name cost 12E per year and cannot be revoked.
- Once domain is added, it is marked as insecure. This will be fixed in next step.

#### 1b. Add SSL certificate to application gateway ####

In step 1a, a custom hostname was assigend to the web app. In step 1b the custom domain name will be secured using an SSL certificate. Execute the steps in this [link](https://docs.microsoft.com/nl-nl/azure/app-service/configure-ssl-certificate#create-a-free-certificate-preview). One remark:

- Make secure that a CNAME record is added in DNS zone in which www refers to link of web app

When all steps are executed successfully, the configuration looks as follows:

![1b. web app with custom domain secured with SSL](https://github.com/rebremer/application-gateway-aadauth/blob/master/images/webapp_customdomain_ssl.png "1b. web app with custom domain secured with SSL")

#### 1c. Turn on AAD authentication of web app ####

In this [link](https://docs.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad#-configure-with-express-settings), it is explained how to set up AAD authentication for a web app. One remark:
- To automate AAD auth with advanced settings, see this [script](https://github.com/rebremer/managed_identity_authentication/blob/master/AAD_auth_ADFv2_MI_to_Azure_Function.ps1).

#### 1d. Change callback URL in app registration ####

In step 1c, an app registration was created that is linked to the web app. In this web app, the callback URL of the custom domain name needs to be added. Look up the app registration in Azure Active Directory (using the same name as the web app) and add the domain, see also below.

![1d. Add callback URL of custom domain to app registration](https://github.com/rebremer/application-gateway-aadauth/blob/master/images/appregistation_callback_customdomain.png "1d. Add callback URL of custom domain to app registration")


### 2. Configure application gateway ###

The following steps need to be executed:

- 2a. Add SSL certificate to application gateway
- 2b. Add custom domain to application gateway settings
- 2c. Point DNS zone to application gateway
- 2d. Lock down web app

#### 2a. Add SSL certificate to application gateway ####

In this step, an SSL certificate is created and added as Listener in the application gateway. This enables the user to use the custom domain name using HTTPS and is also required since the callback URL in step 1d only supports HTTPS. See this [link](https://docs.microsoft.com/en-us/azure/application-gateway/create-ssl-portal#create-a-self-signed-certificate) how to create an self-signed SSL certificate and to add it as listener to web app. See also below when successfully configured:

![2a. Self-signed certificate as listener in gateway](https://github.com/rebremer/application-gateway-aadauth/blob/master/images/selfsigned_ssl_listener.png "2a. Self-signed certificate as listener in gateway")

#### 2b. Add custom domain to application gateway settings ####

In this step, the application gateway is changed such that the custom domain is used when referred to the web app. The PowerShell scripts in the blog will be used, however, since the AzureRM are still used in that [blog](http://thewindowsupdate.com/2019/04/01/setting-up-application-gateway-with-an-app-service-that-uses-azure-active-directory-authentication/), the Az commands can be found below.

```PowerShell

$webappFQDN = "<<your webapp link, e.g. test-flaskrbr-app.azurewebsites.net>>" 
$gw = Get-AzApplicationGateway -Name "<<your webapp name>>" -ResourceGroupName "<<your rg name>>" 
$match=New-AzApplicationGatewayProbeHealthResponseMatch -StatusCode 200-401 
Add-AzApplicationGatewayProbeConfig -name AppGatewayAADProbe -ApplicationGateway $gw -Protocol Https -Path / -Interval 30 -Timeout 120 -UnhealthyThreshold 3 -PickHostNameFromBackendHttpSettings -Match $match 
$probe = Get-AzApplicationGatewayProbeConfig -name AppGatewayAADProbe -ApplicationGateway $gw 
Set-AzApplicationGatewayBackendHttpSettings -Name "<<your http setting name>>" -ApplicationGateway $gw -HostName "<<your hostname, e.g. www.rbrpdomains.com>>" -Port 443 -Protocol https -CookieBasedAffinity Disabled -RequestTimeout 30 -Probe $probe 
Set-AzApplicationGatewayBackendAddressPool -Name "<<your backendpool name>>" -ApplicationGateway $gw -BackendFqdns $webappFQDN 
Set-AzApplicationGateway -ApplicationGateway $gw 

```

When run successfully, the httpsettings will look as follows:

![2b. Httpsettings with custom domain and https](https://github.com/rebremer/application-gateway-aadauth/blob/master/images/httpsettings_https_customdomain.png "2b. Httpsettings with custom domain and https")

#### 2c. Point DNS zone to application gateway ####

In this step, the DNS zone is pointed to the application gateway. Go to your DNS zone, remove the A record and point the CNAME with WWW to you application gateway URL, see also below.

![2c. DNS zone pointing to application gateway URL](https://github.com/rebremer/application-gateway-aadauth/blob/master/images/dnszone_applicationgateway_url.png "2c. DNS zone pointing to application gateway URL")

#### 2d. Lock down web app ####

Finally, the web app can be locked down for the public internet and shall be only opened for the application gateway. Go to access restrictions and add the VNET of the application gateway as only resource that can acccess web app, see also below.

![2d1. Access restrictions web app only allowing VNET application gateway](https://github.com/rebremer/application-gateway-aadauth/blob/master/images/lockdown_webapp_vnet_gateway.png "2d1. Access restrictions web app only allowing VNET application gateway")

When everything is configured successfully, you can visit to https://<<your custom domain>> and authentication using AAD without being redirected to web app, see also below.

![2d2. Application gateway AAD auth using custom domain](https://github.com/rebremer/application-gateway-aadauth/blob/master/images/aadauth_customdomain_application_gateway.png "2d2. Application gateway AAD auth using custom domain")