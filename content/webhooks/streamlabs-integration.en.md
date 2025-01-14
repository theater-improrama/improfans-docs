---
title: 'Streamlabs Integration'
date: 2025-01-12T21:34:00+01:00
---

[Streamlabs](https://streamlabs.com/) provides an [API interface for incoming donations](https://dev.streamlabs.com/v1/reference/donations-1) from third-party providers - for example from ImproFans. This means donations made through ImproFans can also be displayed live via Streamlabs.

Below we'll show how to set up our webhook through different platforms.

## Via [Pipedream](https://pipedream.com/)

[Pipedream](https://pipedream.com/) is a (free) platform for automating web events that allows you to receive our webhooks for donation notifications and forward them to Streamlabs without much effort.

1. Log in to Pipedream and open the [Projects overview](https://pipedream.com/projects).
2. Create a new workflow by clicking *New Workflow* in the top right:
    ![image](/images/webhooks/streamlabs-integration/01_new-workflow.jpg)
3. Enter a name for the workflow project (e.g. `ImproFans Webhook`) and click *Create project and continue*:
    ![image](/images/webhooks/streamlabs-integration/02_create-project.jpg)
    {{< callout type="info" >}}
    A Pipedream project can contain multiple workflows.
    {{< /callout >}}
4. Now enter a name for the actual workflow (e.g. `Streamlabs`) and click *Create Workflow*:
    ![image](/images/webhooks/streamlabs-integration/03_create-workflow.jpg)
5. Now add a trigger for the webhook. The trigger gives you the webhook link that you'll need to add to ImproFans later. Click the blue *Add Trigger* button:
    ![image](/images/webhooks/streamlabs-integration/04_add-trigger.jpg)
6. Select *HTTP / Webhook* here:
    ![image](/images/webhooks/streamlabs-integration/05_select-trigger-1.jpg)
7. ... and then *New Requests*:
    ![image](/images/webhooks/streamlabs-integration/06_select-trigger-2.jpg)
8. Confirm adding the trigger by clicking *Save and continue*:
    ![image](/images/webhooks/streamlabs-integration/07_configure-trigger.jpg)
9. Now copy the link in the red marked box:
    ![image](/images/webhooks/streamlabs-integration/08_copy-endpoint-url.jpg)
10. Log in to your ImproFans account and open the [Webhook overview](https://improfans.de/u/webhooks). Paste the link from the previous step here and then click *Add webhook*:
    ![image](/images/webhooks/streamlabs-integration/09_add-improfans-webhook.de.jpg)
11. Back to Pipedream. Click the *+* symbol to add a new action - in this case the code for the StreamElements link:
    ![image](/images/webhooks/streamlabs-integration/10_add-action.jpg)
12. Enter *Streamlabs* (1) in the actions search, press enter and then click the *Streamlabs* button (2):
    ![image](/images/webhooks/streamlabs-integration/11_select-action-1.jpg)
13. Then click the *Build any Streamlabs API request* button:
    ![image](/images/webhooks/streamlabs-integration/12_select-action-2.jpg)
14. First assign a new name for the action (e.g. `streamlabs_tip`). Simply click on the action name (default is *streamlabs*) and enter a new name:
    ![image](/images/webhooks/streamlabs-integration/13_configure-action-1.jpg)
15. Insert the code below the image in the red marked box:
    ![image](/images/webhooks/streamlabs-integration/14_configure-action-2.jpg)
    ```js
    import axios from "axios"

    export default defineComponent({
      props: {
        streamlabs: {
          type: "app",
          app: "streamlabs",
        }
      },
      async run({steps, $}) {
        return await axios.post(
          `https://streamlabs.com/api/v1.0/donations`, // v1.0 vs v2.0 API endpoint uses different authentication system
          {
            name: steps.trigger.event.body.data.name ?? 'Anonymous',
            message: steps.trigger.event.body.data.message,
            identifier: steps.trigger.event.body.data.donor_id ?? '00000000-0000-0000-0000-000000000000',
            amount: steps.trigger.event.body.data.amount,
            currency: steps.trigger.event.body.data.currency,
            skip_alert: "no",
            access_token: this.streamlabs.$auth.oauth_access_token
          }
        );
      },
    })
    ```
16. Now connect your Streamlabs account with Pipedream by opening the *Connect a Streamlabs account* menu and clicking *Connect new account*. A separate window will open where you'll be prompted to log in to Streamlabs and grant Pipedream access to your Streamlabs account's API:
    ![image](/images/webhooks/streamlabs-integration/15_configure-action-3.jpg)
17. Now click *Deploy* to start the workflow so you can receive our webhook events:
    ![image](/images/webhooks/streamlabs-integration/16_deploy.jpg)

Now you're ready to receive donation notifications on Streamlabs.
