---
title: 'StreamElements Integration'
date: 2024-12-21T16:51:10+01:00
---

[StreamElements](https://streamelements.com/) offers an [API interface for incoming donations](https://dev.streamelements.com/docs/api-docs/7e632a4cecfe1-channel) from third-party providers - such as [ImproFans](https://improfans.de/). This means that donations made via ImproFans can also be displayed live on StreamElements.

Below, you’ll learn how to set up our webhook on various platforms.

## Via [Pipedream](https://pipedream.com/)

[Pipedream](https://pipedream.com/) is a (free) platform for automating web events. It lets you easily receive our donation notification webhooks and forward them to StreamElements.

### Obtaining StreamElements Credentials

To use the StreamElements API, you'll need your StreamElements "Account ID" and "JWT Token" (simply referred to as "Token" in the following sections). Here’s how to obtain them:

1. Log in to StreamElements at [this link](https://streamelements.com/dashboard).
2. Open the [Channel Overview](https://streamelements.com/dashboard/account/channels).
3. You’ll see your Account ID and Token here:
   ![image](/images/webhooks/streamelements-integration/01_streamelements-credentials.en.jpg)

   You’ll need both of these later.

### Configuring Pipedream

1. Log in to [Pipedream](https://pipedream.com/) and open the [Project Overview](https://pipedream.com/projects).
2. Create a new workflow by clicking on *New Workflow*:
   ![image](/images/webhooks/streamelements-integration/02_new-workflow.jpg)
3. Enter a name for the project workflow (e.g., `ImproFans Webhook`) and click on *Create project and continue*:
   ![image](/images/webhooks/streamelements-integration/03_create-project.jpg)

   {{< callout type="info" >}}
   A Pipedream project can contain multiple workflows.
   {{< /callout >}}

4. Now name the actual workflow (e.g., `StreamElements`) and click on *Create Workflow*:
   ![image](/images/webhooks/streamelements-integration/04_create-workflow.jpg)
5. Add a trigger for the webhook. This trigger will give you the webhook link that you’ll later add to ImproFans:
   ![image](/images/webhooks/streamelements-integration/05_add-trigger.jpg)
6. Select *HTTP / Webhook*:
   ![image](/images/webhooks/streamelements-integration/06_select-trigger-1.jpg)
7. Next, select *New Requests*:
   ![image](/images/webhooks/streamelements-integration/07_select-trigger-2.jpg)
8. Confirm adding the trigger by clicking *Save and continue*:
   ![image](/images/webhooks/streamelements-integration/08_configure-trigger.jpg)
9.   1. The link in the red box is the webhook endpoint. You’ll add this below on [ImproFans](https://improfans.de/) in your webhooks.  
     2. Click on the *+* symbol to add a new action—in this case, the code to connect with StreamElements:

     ![image](/images/webhooks/streamelements-integration/09_add-action.jpg)
10. Select *Node*:
    ![image](/images/webhooks/streamelements-integration/10_select-action-1.jpg)
11. Next, choose *Run Node code*:
    ![image](/images/webhooks/streamelements-integration/11_select-action-2.jpg)
12.   1. First, give this action a new name (e.g., `streamelements_tip`) by clicking on *code* within the action.
      2. Insert the following code into the red-marked box next to the number 2:
         
         ```js
         import axios from "axios"

         export default defineComponent({
           props: {
             streamelementsAccountId: {
               type: "string",
               label: "StreamElements Account ID"
             },
             streamelementsAccessToken: {
               type: "string",
               label: "StreamElements Access Token"
             }
           },
           async run({ steps, $ }) {
             await axios.post(`https://api.streamelements.com/kappa/v2/tips/${this.streamelementsAccountId}`, {
               user: {
                 username: `${steps.trigger.event.data.name ?? 'Anonymous'}`,
                 userId: steps.trigger.event.data.donor_id ?? '00000000-0000-0000-0000-000000000000',
                 email: 'no@email.no',
               },
               provider: "improfans",
               message: `${steps.trigger.event.data.message}`,
               amount: steps.trigger.event.data.amount,
               currency: steps.trigger.event.data.currency,
               imported: "true",
             }, {
               headers: {
                 'Authorization': `Bearer ${this.streamelementsAccessToken}`,
               }
             });
           },
         })
         ```
      3. After you’ve inserted the code, click on *Refresh fields* so the fields for your StreamElements Account ID and Token appear:
         ![image](/images/webhooks/streamelements-integration/12_configure-action-1.jpg)
13. Now enter your StreamElements Account ID and Token from earlier (see above) in the fields for *StreamElements Account ID* and *StreamElements Access Token*:
    ![image](/images/webhooks/streamelements-integration/13_configure-action-2.jpg)
14. Click on *Deploy* to start the workflow so you can receive our webhook events:
    ![image](/images/webhooks/streamelements-integration/14_deploy-1.jpg)
15. Copy the link in the red-marked box:
    ![image](/images/webhooks/streamelements-integration/15_deploy-2.jpg)
16. Log in to your [ImproFans](https://improfans.de/) account and open the [Webhook Overview](https://improfans.de/u/webhooks). Insert the link from the Pipedream endpoint here, then click on *Add webhook*:
    ![image](/images/webhooks/streamelements-integration/16_add-improfans-webhook.en.jpg)

Now you’re all set to receive donation notifications on StreamElements.