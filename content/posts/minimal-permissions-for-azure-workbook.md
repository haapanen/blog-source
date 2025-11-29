---
title: "Minimal permissions for Azure Workbook, Log Analytics Workspace & Application Insights"
date: 2025-11-29T20:00:00+03:00
draft: false
---

## The problem

I recently wanted to test Azure Workbook as a tool for monitoring dashboards. One challenge that I ran into is Workbook uses Azure permissions to grant access to both the Workbook itself and any logs / metrics it's trying to show. It wasn't exactly obvious what built-in Azure roles are needed to provide access to a Workbook that uses Application Insights metrics/logs stored in Log Analytics Workspace.

## Solution

### Azure Workbook

Grant `Workbook Reader` (`b279062a-9be3-42a0-92ae-8b3cf002ec4d`) scoped to the Workbook to the user / group. This grants view access to the workbook.

### Application Insights

Grant `Monitoring Reader` (`43d0d8ad-25c7-4714-9337-8ba259a9fe05`) scoped to the Application Insights to the user / group. This grants reader access to Application Insights.

### Log Analytics Workspace

This depends on how Log Analytics is used. Log Analytics supports both access to all logs within the workspace or access to logs scoped to the resource user has read access to.

If you wish to grant access to all logs within the workspace, grant `Log Analytics Data Reader` (`3b03c2da-16b3-4a49-8834-0f8130efdd3b`). This grants access to query & search all logs within the workspace.

If you wish to grant access to logs from resources user has access to, you don't need to grant anything at the log analytics workspace level. To grant access to specific resource's logs, grant `Monitoring Reader` (`43d0d8ad-25c7-4714-9337-8ba259a9fe05`) access to the resource and ensure the log analytics workspace access control mode is set to `Use resource or workspace permissions`.
