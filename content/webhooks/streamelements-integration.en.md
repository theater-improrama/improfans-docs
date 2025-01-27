---
title: 'StreamElements Integration'
date: 2024-12-21T16:51:10+01:00
---

[StreamElements](https://streamelements.com/) offers an [API interface for incoming donations](https://dev.streamelements.com/docs/api-docs/7e632a4cecfe1-channel) from third-party providers - for example from ImproFans. This means donations made through ImproFans can also be displayed live via StreamElements.

The following shows how to set up our webhook across different platforms.

## Via [Pipedream](https://pipedream.com/)

[Pipedream](https://pipedream.com/) is a (free) web event automation platform that allows you to easily receive our webhooks for donation notifications and forward them to StreamElements.

### Obtaining StreamElements Credentials

To use the StreamElements API, you need your StreamElements "Account ID" and "JWT Token" (hereafter just "Token"). Here's how to get them:

1. Log in to StreamElements [dashboard](https://streamelements.com/dashboard).
2. Open the [channel overview](https://streamelements.com/dashboard/account/channels).
3. Here you can see your Account ID and Token:
    ![image](/images/webhooks/streamelements-integration/01_streamelements-credentials.en.jpg)
    Both will be needed in the following steps.

### Configuring Pipedream

1. Log in to Pipedream and open the [project overview](https://pipedream.com/projects).
2. Create a new workflow by clicking *New Workflow* in the top right:
    ![image](/images/webhooks/streamelements-integration/02_new-workflow.jpg)
3. Enter a name for the workflow project (e.g. `ImproFans Webhook`) and click *Create project and continue*:
    ![image](/images/webhooks/streamelements-integration/03_create-project.jpg)
    {{< callout type="info" >}}
    A Pipedream project can contain multiple workflows.
    {{< /callout >}}
4. Now enter a name for the actual workflow (e.g. `StreamElements`) and click *Create Workflow*:
    ![image](/images/webhooks/streamelements-integration/04_create-workflow.jpg)
5. Now add a trigger for the webhook. The trigger gives you the webhook link that you'll need to add to ImproFans later. Click the blue *Add Trigger* button:
    ![image](/images/webhooks/streamelements-integration/05_add-trigger.jpg)
6. Select *HTTP / Webhook*:
    ![image](/images/webhooks/streamelements-integration/06_select-trigger-1.jpg)
7. ... and then *New Requests*:
    ![image](/images/webhooks/streamelements-integration/07_select-trigger-2.jpg)
8. Confirm adding the trigger by clicking *Save and continue*:
    ![image](/images/webhooks/streamelements-integration/08_configure-trigger.jpg)
9. Now copy the link in the red marked box:
    ![image](/images/webhooks/streamelements-integration/09_copy-endpoint-url.jpg)
10. Log in to your ImproFans account and open the [webhook overview](https://improfans.de/u/webhooks). Add the link from the previous step here and then click *Add Webhook*:
    ![image](/images/webhooks/streamelements-integration/10_add-improfans-webhook.en.jpg)
11. Back to Pipedream. Click the *+* symbol to add a new action - in this case the code for the StreamElements connection:
    ![image](/images/webhooks/streamelements-integration/11_add-action.jpg)
12. Select *Node*:
    ![image](/images/webhooks/streamelements-integration/12_select-action-1.jpg)
13. ... and then *Run Node code*:
    ![image](/images/webhooks/streamelements-integration/13_select-action-2.jpg)
14. First enter a new name for the action (e.g. `streamelements_tip`). To do this, simply click on the name of the action (*code* by default) and then enter a new name:
    ![image](/images/webhooks/streamelements-integration/14_configure-action-1.jpg)
15. Delete the existing standard code from the text field (red rectangle in the image) and then insert the code below the image:
    ![image](/images/webhooks/streamelements-integration/15_configure-action-2.jpg)
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
            user:
            {
                username: `${steps.trigger.event.body.data.name ?? 'Anonymous'}`,
                userId: steps.trigger.event.body.data.donor_id ?? '00000000-0000-0000-0000-000000000000',
                email: 'no@email.no',
            },
            provider: "improfans",
            message: `${steps.trigger.event.body.data.message}`,
            amount: steps.trigger.event.body.data.amount,
            currency: steps.trigger.event.body.data.currency,
            imported: "true",
            }, {
            headers: {
            'Authorization': `Bearer ${this.streamelementsAccessToken}`,
            }
        });
        },
    })
    ```
16. After inserting the code, click *Refresh fields* to display the fields for your StreamElements Account ID and Token:
    ![image](/images/webhooks/streamelements-integration/16_configure-action-3.jpg)
17. Now add your StreamElements Account ID and Token from earlier (see above) in the designated fields (*StreamElements Account ID* and *StreamElements Access Token*):
    ![image](/images/webhooks/streamelements-integration/17_configure-action-4.jpg)
18. Now click *Deploy* to start the workflow so you can receive our webhook events:
    ![image](/images/webhooks/streamelements-integration/18_deploy.jpg)

Now you're ready to receive donation notifications on StreamElements.
