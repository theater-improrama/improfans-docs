---
title: 'StreamElements Integration'
date: 2024-12-21T16:51:10+01:00
---

[StreamElements](https://streamelements.com/) bietet eine [Schnittstelle (API) für eingehende Spenden](https://dev.streamelements.com/docs/api-docs/7e632a4cecfe1-channel) von Drittanbietern an - zum Beispiel von ImproFans. D. h. Spenden, die über ImproFans getätigt werden, können auch live über StreamElements angezeigt werden.

Im Folgenden wird gezeigt, wie man über verschiedene Plattformen unseren Webhook einrichtet.

## Via [Pipedream](https://pipedream.com/)

[Pipedream](https://pipedream.com/) ist eine (kostenlose) Plattform zur Automatisierung von Web-Events, die es dir ermöglicht, ohne großen Aufwand unsere Webhooks für Spendenbenachrichtigungen empfangen und an StreamElements weiterzugeben.

### StreamElements Zugangsdaten erhalten

Damit du die API von StreamElements nutzen kannst, benötigst du deine StreamElements "Konto-ID" und -"JWT Token" (folgend nur "Token"). Das machst du so:

1. Bei StreamElements [einloggen](https://streamelements.com/dashboard).
2. Öffne die [Kanalübersicht](https://streamelements.com/dashboard/account/channels).
3. Hier siehst du deine Konto-ID und den Token:
    ![image](/images/webhooks/streamelements-integration/01_streamelements-credentials.de.jpg)
    Beide werden im Folgenden benötigt.

### Pipedream konfigurieren

1. Logge dich bei Pipedream ein und öffne die [Projektübersicht](https://pipedream.com/projects).
2. Erstelle einen neuen Workflow, indem du auf *New Workflow* klickst:
    ![image](/images/webhooks/streamelements-integration/02_new-workflow.jpg)
3. Gib einen Namen für das Projekt des Workflows an (bspw. `ImproFans Webhook`) und klicke auf *Create project and continue*:
    ![image](/images/webhooks/streamelements-integration/03_create-project.jpg)
    {{< callout type="info" >}}
    Ein Pipedream-Projekt kann mehrere Workflows enthalten.
    {{< /callout >}}
4. Gib nun einen Namen für den eigentlichen Workflow an (bspw. `StreamElements`) und klicke auf *Create Workflow*:
    ![image](/images/webhooks/streamelements-integration/04_create-workflow.jpg)
5. Füg' nun einen Trigger für den Webhook hinzu. Der Trigger gibt dir den Webhook-Link, den du später bei ImproFans hinzufügen musst:
    ![image](/images/webhooks/streamelements-integration/05_add-trigger.jpg)
6. Wähle hierzu *HTTP / Webhook* aus:
    ![image](/images/webhooks/streamelements-integration/06_select-trigger-1.jpg)
7. ... und danach *New Requests*:
    ![image](/images/webhooks/streamelements-integration/07_select-trigger-2.jpg)
8. Bestätige das Hinzufügen des Triggers, indem du auf *Save and continue* klickst:
    ![image](/images/webhooks/streamelements-integration/08_configure-trigger.jpg)
9.  1. Der Link in der roten Box ist der Webhook-Endpoint. Diesen fügst du weiter unten auf ImproFans unter deinen Webhooks hinzu;
    2. Klicke auf das *+*-Symbol, um eine neue Action hinzuzufügen - in diesem Falle den Code für die StreamElements-Verknüpfung:

    ![image](/images/webhooks/streamelements-integration/09_add-action.jpg)
10. Wähle hierzu *Node* aus:
    ![image](/images/webhooks/streamelements-integration/10_select-action-1.jpg)
11. ... und danach *Run Node code*:
    ![image](/images/webhooks/streamelements-integration/11_select-action-2.jpg)
12. 1. Vergebe hier zunächst einen neuen Namen für die Action (bspw. `streamelements_tip`). Dazu klickst du einfach auf *code* innehralb der Action;
    2. Füge Folgenden Code in der rot markierten Box neben der Zahl 2 ein:
        
        ```js
        import axios from "axios"

        export default defineComponent({
          props: {
            streamelementsAccountId: {
              type: "string",
              label: "StreamElements Konto-ID"
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
    3. Nachdem du den Code eingefügt hast, klicke auf *Refresh fields*, damit die Felder für deine StreamElements Konto-ID und -Token angezeigt werden:
    
    ![image](/images/webhooks/streamelements-integration/12_configure-action-1.jpg)
13. Füge nun deine StreamElements Konto-ID und -Token von vorhin (s. oben) in die dafür vorgesehenen Felder (*StreamElements Konto-ID* und *StreamElements Access Token*):
    ![image](/images/webhooks/streamelements-integration/13_configure-action-2.jpg)
14. Klicke nun auf *Deploy*, um den Workflow zu starten, sodass du unsere Webhook-Events empfangen kannst:
    ![image](/images/webhooks/streamelements-integration/14_deploy-1.jpg)
15. Kopiere nun den Link in der rot markierten Box:
    ![image](/images/webhooks/streamelements-integration/15_deploy-2.jpg)
16. Logge dich in deinen ImproFans-Account ein und öffne die [Webhook-Übersicht](https://improfans.de/u/webhooks). Füge den Link vom Pipedream-Endpunkt nun hier ein und drücke anschließend auf *Webhook hinzufügen*:
    ![image](/images/webhooks/streamelements-integration/16_add-improfans-webhook.de.jpg)

Jetzt bist du startklar, um Spendenbenachrichtigungen bei StreamElements zu empfangen. 