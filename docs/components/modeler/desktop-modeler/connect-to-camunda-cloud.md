---
id: connect-to-camunda-cloud
title: Deploy your first diagram
---

Desktop Modeler can directly deploy diagrams and start process instances in Camunda Platform 8. Follow the steps below to deploy a diagram:

1. Click the deployment icon:

![deployment icon](./img/deploy-icon.png)

1. Click **Camunda Platform 8 SaaS**, or alternatively, select **Camunda Platform 8 Self-Managed** if you want to deploy to a [local installation](../../../self-managed/platform-deployment/overview.md), for example:

![deployment configuration](./img/deploy-diagram-camunda-cloud.png)

1. Input the `Cluster URL` and the credentials (`Client ID`, `Client Secret`) of your [API client](../../console/manage-clusters/manage-api-clients.md):

![deployment via Camunda Platform 8](./img/deploy-diagram-camunda-cloud-remember.png)

4. Select the **Remember** checkbox if you want to locally store the connection information.

5. Click **Deploy** to perform the actual deployment.

![deployment successful](./img/deploy-diagram-camunda-cloud-success.png)
