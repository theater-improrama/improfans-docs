---
title: 'Streamlabs Integration'
date: 2025-01-12T21:34:00+01:00
---

[Streamlabs](https://streamlabs.com/) bietet eine [Schnittstelle (API) für eingehende Spenden](https://dev.streamlabs.com/v1/reference/donations-1) von Drittanbietern an - zum Beispiel von ImproFans. D.h. Spenden, die über ImproFans getätigt werden, können auch live über Streamlabs angezeigt werden.

Im Folgenden wird gezeigt, wie man über verschiedene Plattformen unseren Webhook einrichtet.

## Via [Pipedream](https://pipedream.com/)

[Pipedream](https://pipedream.com/) ist eine (kostenlose) Plattform zur Automatisierung von Web-Events, die es dir ermöglicht, ohne großen Aufwand unsere Webhooks für Spendenbenachrichtigungen empfangen und an Streamlabs weiterzugeben.

1. Logge dich bei Pipedream ein und öffne die [Projektübersicht](https://pipedream.com/projects).
2. Erstelle einen neuen Workflow, indem du oben rechts auf *New Workflow* klickst:
    ![image](/images/webhooks/streamlabs-integration/01_new-workflow.jpg)
3. Gib einen Namen für das Projekt des Workflows an (bspw. `ImproFans Webhook`) und klicke auf *Create project and continue*:
    ![image](/images/webhooks/streamlabs-integration/02_create-project.jpg)
    {{< callout type="info" >}}
    Ein Pipedream-Projekt kann mehrere Workflows enthalten.
    {{< /callout >}}
4. Gib nun einen Namen für den eigentlichen Workflow an (bspw. `Streamlabs`) und klicke auf *Create Workflow*:
    ![image](/images/webhooks/streamlabs-integration/03_create-workflow.jpg)
5. Füg' nun einen Trigger für den Webhook hinzu. Der Trigger gibt dir den Webhook-Link, den du später bei ImproFans hinzufügen musst. Klicke dazu auf den blauen *Add Trigger*-Button:
    ![image](/images/webhooks/streamlabs-integration/04_add-trigger.jpg)
6. Wähle hierzu *HTTP / Webhook* aus:
    ![image](/images/webhooks/streamlabs-integration/05_select-trigger-1.jpg)
7. ... und danach *New Requests*:
    ![image](/images/webhooks/streamlabs-integration/06_select-trigger-2.jpg)
8. Bestätige das Hinzufügen des Triggers, indem du auf *Save and continue* klickst:
    ![image](/images/webhooks/streamlabs-integration/07_configure-trigger.jpg)
9. Kopiere nun den Link in der rot markierten Box:
    ![image](/images/webhooks/streamlabs-integration/08_copy-endpoint-url.jpg)
10. Logge dich in deinen ImproFans-Account ein und öffne die [Webhook-Übersicht](https://improfans.de/u/webhooks). Füge den Link vom vorherigen Schritt nun hier ein und drücke anschließend auf *Webhook hinzufügen*:
    ![image](/images/webhooks/streamlabs-integration/09_add-improfans-webhook.de.jpg)
11. Zurück zu Pipedream. Klicke auf das *+*-Symbol, um eine neue Action hinzuzufügen - in diesem Falle den Code für die StreamElements-Verknüpfung:
    ![image](/images/webhooks/streamlabs-integration/10_add-action.jpg)
12. Gib hierzu in der Suche für Actions *Streamlabs* (1) ein, drücke die Eingabetaste und klicke anschließend auf den Button *Streamlabs* (2):
    ![image](/images/webhooks/streamlabs-integration/11_select-action-1.jpg)
13. Danach klickst du auf den Button *Use any Streamlabs API in Node.js*:
    ![image](/images/webhooks/streamlabs-integration/12_select-action-2.jpg)
14. Vergebe hier zunächst einen neuen Namen für die Action (bspw. `streamlabs_tip`). Klicke dazu einfach auf den Namen der Action (standardmäßig *streamlabs*) und gib dann einen neuen Namen ein:
    ![image](/images/webhooks/streamlabs-integration/13_configure-action-1.jpg)
15. Füge den Code unter dem Bild in der rot markierten Box ein:
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
16. Verbinde nun dein Streamlabs-Konto mit Pipedream, indem du das Menü *Connect a Streamlabs account* öffnest und auf *Connect new account* klickst. Danach öffnet sich ein separates Fenster indem du dazu aufgefordert wirst, dich bei Streamlabs anzumelden und Pipedream Zugriff auf die Streamlabs API deines Streamlabs-Kontos zu geben:
    ![image](/images/webhooks/streamlabs-integration/15_configure-action-3.jpg)
17. Klicke nun auf *Deploy*, um den Workflow zu starten, sodass du unsere Webhook-Events empfangen kannst:
    ![image](/images/webhooks/streamlabs-integration/16_deploy.jpg)

Jetzt bist du startklar, um Spendenbenachrichtigungen bei Streamlabs zu empfangen.
