# Google Sheets Content Calendar Template

This document explains the required structure for the Google Sheets document used by the n8n Agentic Workflow for Social Media Automation. 
Please create a new Google Spreadsheet and add the following 3 sheets (tabs) with the exact column headers specified below.

## Sheet 1: Topic Queue
This sheet is populated by the **Topic Research Agent**.

| ID | Date Added | Topic | Angle | Target Audience | Status | Approved By | Approved Date |
|:---|:-----------|:------|:------|:----------------|:-------|:------------|:--------------|
| (Empty, will be generated) | (e.g., 2023-10-27) | (The topic idea) | (The angle/hook) | (e.g., Software Engineers) | `Pending` (Initial) | (User email) | (Approval Date) |

## Sheet 2: Content Queue
This sheet is populated by the **Content Generation Agent** and updated by the **HITL Approval Workflow** and **Publishing Agent**.

| ID | Topic ID | Platform | Content | Hashtags | Status | Generated At | Approved At | Published At | Post URL | Error Log |
|:---|:---------|:---------|:--------|:---------|:-------|:-------------|:------------|:-------------|:---------|:----------|
| (Content ID) | (Topic ID from Sheet 1) | `LinkedIn` or `Twitter` | (The actual post text) | (e.g. #Tech #AI) | `Awaiting Approval` (Initial) | (Timestamp) | (Timestamp) | (Timestamp) | (URL of post) | (Any error messages) |

## Sheet 3: Workflow Logs
This sheet is populated by the **Monitoring & Alerting Agent** or used by custom logging steps.

| Timestamp | Workflow | Step | Status | Error Message | Duration (ms) |
|:----------|:---------|:-----|:-------|:--------------|:--------------|
| (Timestamp) | (e.g., 01_topic_research) | (e.g., OpenAI Node) | `Success` or `Failed` | (Error text if any) | (e.g., 1250) |

## Setting up in n8n
1. Create a Google Service Account in Google Cloud Console.
2. Enable the Google Sheets API.
3. Share the Google Spreadsheet with the service account email (giving Editor access).
4. Use the Service Account JSON credentials in your n8n `.env` or UI.
