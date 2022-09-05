# AADPasswordExpiryNotifier
Azure Logic App that identifies when Azure AD User Accounts' passwords are set to expire, and notifies them via email using the _mail_ property in AzureAD.

In some cases, users may have multiple user accounts in different Azure AD tenants. These accounts may be different than what they use to login to their machines locally for example, and as such won't be notified about pending password expiry.

Without using SSPR (which requires certain licensing, and involves extra costs) this solution notifies the user about their pending password reset via email (through SendGrid, O365, or whatever).

## How the solution works
The Logic App performs a number of steps, which can be reduced to the following pseudo-code:
1.	A request is received via HTTP to run the app
2.	Extract the configuration parameters provided in the body of the request
3.	Get all users in the directory (specified in the DirectoryId variable)
4.	Iterate through all the users, and build a list of actions to be taken:<br>
4.1.	Calculate each user’s password’s expiry date from the date it was last set, and the DaysToExpiryFromLastSet configuration parameter<br>
4.2.	If this date falls on FirstWarningDays, add an action item to send the First email warning<br>
4.3.	If this date falls on SecondWarningDays, add an action to send the Second email warning<br>
4.4.	If this date falls on ThirdWarningDays, add an action to send the Third email warning<br>
> This approach expects that the Logic App will run daily. If not, certain users may not receive certain notifications. This approach was chosen as it provides the simplest solution to not repeatedly email users.
> A possible improvement would involve saving the state of the notifications to a storage account

5.	Iterate through the resulting action list, and:
a.	Check if the user’s mail property is not null
b.	Send an email to the user
> The action list is built first and then executed, to ensure that a single step was created containing the email body to be sent. If the email body needs to change (which may be frequently), it needs to only change in one place as a result

## Deploying the solution automatically

[![Deploy To Azure](https://docs.microsoft.com/en-us/azure/templates/media/deploy-to-azure.svg)](https://portal.azure.com/#blade/Microsoft_Azure_CreateUIDef/CustomDeploymentBlade/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fgrimstoner%2FAADPasswordExpiryNotifier%2Fmaster%2Fazuredeploy.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2Fgrimstoner%2FAADPasswordExpiryNotifier%2Fmaster%2FazuredeployUI.json)

and follow the prompts to configure the solution.

## Deploy the solution manually

Alternatively, you can manually deploy the solution using the below:

### Create a Logic App

Create a Logic App in the target subscription by clicking the Add button on the Logic Apps Service in the Azure Portal:

![image](https://user-images.githubusercontent.com/3426823/188476849-1a844888-5ec3-4a63-86ea-99eacaa903fd.png)
<p align="center">Figure 1: Creating a Logic App</p>

When configuring the Logic App, set the following parameters:

- **Subscription**: The subscription to deploy to.
> Ensure that the Logic App is deployed to the same subscription as the one containing the Azure Active Directory resource
- **Resource group**: The resource group to deploy to.
- **Type**: Consumption
> A Consumption based model is used due to the low number of executions, as well as the low requirement on resources that are required for this Logic App
- **Logic App name**: The name of the Logic App
- **Publish**: Workflow
- **Region**: The Region to deploy to.
- **Enable Log Analytics**: Enable Log Analytics to analyze and/or automate based on the results of Logic App executions.
- **Log Analytics Workspace**: The Log Analytics Workspace to save logs in.
> Enabling Log Analytics Analytics allows access to a number of reports, such as *Triggered failures count* and *Total billable executions*

- **Tags**: Tag the resource as per the applicable policy

### Create a System Assigned Managed Identity

The Logic App needs to be able to query information in Azure Active Directory through the [Microsoft Graph API](https://developer.microsoft.com/en-us/graph). The AAD Password Expiry Notified solution makes use of a System Assigned Managed Identity (as opposed to a [User Assigned Managed Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)).

To create a System Assigned Managed Identify, perform the following steps:
- Open the Logic App you created in Create a Logic App
- Navigate to Identity under Settings
- Enable the System assigned managed identity by setting Status to On, and saving the changes:

![image](https://user-images.githubusercontent.com/3426823/188477036-0003ac0e-6509-4eca-b8dc-10bcbbbe1f90.png)
<p align="center">Figure 2: Enabling the System Assigned Managed Identity</p>

### Assign permissions
Next, you’ll need to register the Logic App as an owned application in order to provide it with the relevant access permissions. To do this, navigate to the Azure Active Directory service in the Azure Portal, click the App registrations tab, and click *+New Registration*.
Provide a Name, and leave the default Supported account types as *Accounts in this organizational directory only*, and click *Register* (you do not need to set the *Redirect URL*).

> This solution has not been tested in a cross-tenant environment

Once created, navigate to the App’s API permissions, and add the following permissions under the Microsoft Graph API:
- GroupMember.Read.All
- People.Read.All
- User.Read.All
 
![image](https://user-images.githubusercontent.com/3426823/188477336-ea1100cf-83f5-4265-ab74-15803c05f228.png)
<p align="center">Figure 3: API permissions required</p>

> Applying the principle of least priviledge has not yet been tested for this solution. It may function without granting some of these permissions
 
On the Overview tab, make a note of the *Application (client) ID* value.

### Creating an Application secret
On the *Certificates & secrets* tab, click the *+New client secret button*, and create a secret to be used by the Logic App. Set the *Expires* value to your requirement.
After creating it, take note of the value: this will be used in the Logic App’s configuration.

## Deploy the solution
Logic Apps can be designed using the Logic app designer in Azure Portal. This designer generates the necessary code, which is viewable in the Logic app code view:

![image](https://user-images.githubusercontent.com/3426823/188477464-417e41a2-4678-48f5-bc97-967c3e9b967c.png)

The easiest way to deploy this solution, is by simply copying and pasting the source code into the Logic app code view editor:
- Navigate to the Logic App’s code view
- Copy the source code from AADPasswordExpiryNotifier-LogicApp.json, and paste it into the editor
- Save the changes
- Navigate to the designer to confirm it looks as follows, and no errors are being presented:

![image](https://user-images.githubusercontent.com/3426823/188477554-2db591c4-c45b-4bdb-9dd1-3d6d95578fbb.png)
<p align="center">Figure 4: The Logic App designer view</p>

### Setting the DirectoryId, ApplicationId and Secret variables
In the Logic App designer, configure the options for *Initialize DirectoryId*, *Initialize ApplicationId* and *Initialize secret* with the values recorded earlier.

#### Configuring email sending

By default, the AAD Password Expiry Notifier sends emails via a [SendGrid API](https://sendgrid.com/). Once the Logic App has been created, and the source code saved, you will need to configure this step in the Logic App:
 
![image](https://user-images.githubusercontent.com/3426823/188477697-14089442-1b87-41ef-8e31-a94d6e0d48e4.png)
<p align="center">Figure 5: The SendGrid email sending step</p>

This step also contains the contents of the email that will be received by the user. Configure it as required.
> The Logic App sends emails to the Azure AD user’s mail property. If this property does not contain a value, the user will be ignored.

You could replace this step with another way of notifying your users, i.e., through an Office365 email.

## Triggering the Logic App
The AAD Password Expiry Notifier Logic App is triggered by an HTTP request with the relevant configuration options. 

These configuration options should be encoded as JSON, and embedded into the request body with the following schema:
```
{
    "properties": {
        "DaysToExpiryFromLastSet": {
            "type": "integer"
        },
        "FirstWarningDays": {
            "type": "integer"
        },
        "SecondWarningDays": {
            "type": "integer"
        },
        "ThirdWarningDays": {
            "type": "integer"
        }
    },
    "type": "object"
}
```
> *DaysToExpiryFromLastSet* is used to calculate when the user’s password will expire, using the *lastPasswordChangeDateTime* property returned from the Microsoft Graph API. By default on Azure AD Tenants, this is set at 90 days (see [here](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-sspr-policy#password-policies-that-only-apply-to-cloud-user-accounts)).

### Triggering via another Logic App
The simplest way to trigger the solution is via another Logic App. By following the same steps as for the main solution, deploy another Logic App, but use the source code in AADPasswordExpiryNotifier-Trigger.json.

> Note that using a second Logic App will effectively double the amount of executions, which may be billable dependent on your subscription. However, since the solution is intended to run once daily, this will result in a maximum number of *60* billable executions in a month

This will create a Logic App that triggers automatically based on a schedule, and then calls the AAD Password Expiry Notifier. The schedule can be configured to run as per requirement, and because of the tight integration between Logic Apps, even allow the configuration options to be configured via the front-end:

![image](https://user-images.githubusercontent.com/3426823/188478072-d3c7a6fe-5d00-4d6b-b389-35de1fe61009.png)
<p align="center">Figure 6: Setting configuration options via the designer.</p>

Configure the options as required, and Save the Logic App.

> The AAD Password Expiry Notifier Logic App makes use of asynchronous HTTP calls. Once you’ve created the Trigger app, ensure that the call to the main Logic App is set to use the Asynchronous Pattern by checking the step’s Settings:
> ![image](https://user-images.githubusercontent.com/3426823/188478174-b6cefef8-5c3a-4d59-8b55-2c61e22f8f97.png)
> 
> If this setting is not enabled, you may encounter timeout exceptions in the Logic App’s run history.

 
### Triggering via custom HTTP request
Since the Logic App exposes an HTTP endpoint, it can be invoked from a custom client, such as an Azure Function, PowerShell, PowerApps, or custom code.
See these resources for information:
- [Call or trigger logic apps by using Azure Functions and Azure Service Bus](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-scenario-function-sb-trigger)
- [Call logic apps from Power Automate and Power Apps](https://docs.microsoft.com/en-us/azure/logic-apps/call-from-power-automate-power-apps)
- [Create custom APIs you can call from Azure Logic Apps](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-create-api-app)


 
# Using the solution
Basic monitoring and administration of the solution involves checking the Run History of both the AAD Password Expiry Notifier Logic App, and if so configured, the Trigger app. 
By checking the Overview tab on the Logic Apps, success and failures in runs can be idenfitied:

![image](https://user-images.githubusercontent.com/3426823/188478537-0fe00e2e-356d-493f-bc9a-19159792980b.png)
<p align="center">Figure 7: Viewing the Logic App's run history</p>

Opening a specific run shows the steps performed, and whether the steps passed or failed:
 
![image](https://user-images.githubusercontent.com/3426823/188478662-fdf003a2-e422-4e76-99d3-32d9155e2a03.png)
<p align="center">Figure 8: Viewing specific steps' success or failure</p>

The Response body of the HTTP request will contain a debug log of the actions taken/not taken. To view these, open the Response step (as shown in Figure 7). It will contain readable text (an array of strings), resembling the following:
```
[
  "Ignore xxxxxx, as their password only expires in 66 days.",
  "Ignore xxxxxx, as their password only expires in 66 days.",
  "Send first warning email to xxxxxx, as their password expires in 53 days.",
  "Send second warning email to xxxxxx, as their password expires in 59 days.",
  "Could not send email to xxxxxx since the email address is not set.",
  "Sent email to xxxxxx at xxxxxx@xxxxxx."
]
```

By consulting the run history, and the debug output, most problems can be identified and resolved.
 
> If Log Analytics were enabled during the creation of the Logic App, specific monitors, runbooks, and other automations can be configured to aid in troubleshooting. 
