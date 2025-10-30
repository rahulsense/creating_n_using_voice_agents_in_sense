# Creating and using voice agents in Sense

## Overview

This guide provides step-by-step instructions for creating and deploying voice agents for post-application candidate screening. You'll learn how to configure a Retell AI voice agent, import it into the Sense platform, and set up automated **Submittal Workflows** (workflows that trigger when a candidate applies to a job) for candidate interaction.

### Prerequisites

- Access to Retell AI Dashboard
- Access to Sense Internal Hub (requires AWS VPN connection)
- Multi-entity agency credentials (Sense dashboard)

### Workflow Summary

```
Create Voice Agent (Retell) → Import to Sense → Create Test Data → Configure Workflow → Activate
```

---

## 1. Creating a Voice Agent in Retell AI

### 1.1 Access the Dashboard

Navigate to the [Retell AI Dashboard](https://dashboard.retellai.com).

### 1.2 Initialize New Agent

1. Click **Create Agent**
2. Select **Voice Agent**
   
<img width="3456" height="674" alt="image" src="https://github.com/user-attachments/assets/a9677cbf-2684-46b8-a968-66124080b697" />

3. Choose **Multi-Prompt Agent**
4. Select **Start from Blank** template
   
<img width="3456" height="1910" alt="image" src="https://github.com/user-attachments/assets/e8542c4a-52fa-4194-89b8-b7ae553b4554" />

### 1.3 Configure Base Settings

Configure the following core settings:

| Setting | Value | Description |
|---------|-------|-------------|
| Universal Prompt | Custom prompt text | General instructions for the agent (overarching behavior) |
| Welcome Message | "AI speaks first" | Recommended for better UX |

<img width="3456" height="1542" alt="image" src="https://github.com/user-attachments/assets/0c633cc5-8bb6-4564-937a-2c96f37f1c59" />

> **Note:** The universal prompt provides high-level guidance, while prompt tree prompts handle specific conversation states.

### 1.4 Design Prompt Tree

Click **Edit Prompt Tree** to define the conversation flow. The agent operates in two distinct states:

#### State 1: Dynamic Questions

Generates role-specific questions based on the job description (JD). These questions are **not known to the agent creator beforehand**, but the **criteria for asking these questions is defined**.

**Purpose:** Assess candidate fit against unique role requirements

**Characteristics:**
- Questions dynamically generated from the specific job description
- Agent creator provides criteria, not exact questions
- Adapts to each job's unique requirements

**Example Criteria:**
- Ask 3 questions around the skills section in the Job Description
- Ask 2 questions around past experience mentioned in the Job Description
- Assess cultural fit based on company values in the JD

#### State 2: Static Questions

Asks standardized questions that are **already provided at the time of agent creation**. These are predetermined, role-agnostic questions.

**Purpose:** Collect mandatory information common to all roles

**Characteristics:**
- Questions are predefined and provided to the agent creator
- Same questions asked across all job applications
- Focus on logistical and compliance requirements

**Examples:**
- Work authorization status
- Compensation expectations
- Earliest available start date
- Location preferences
- Willingness to relocate

<img width="3450" height="830" alt="image" src="https://github.com/user-attachments/assets/7cca9e6e-23c3-4b9d-9d41-00ef228befea" />

### 1.5 Configure Functions

Functions extend the agent's capabilities beyond conversation. Configure these on the right panel of your agent page.

#### Function 1: End Call (Default)

Creates a graceful call termination when the conversation concludes.

**Setup:** Use default end call function provided by Retell.

#### Function 2: Reschedule Call

Enables candidates to reschedule the screening call.

<img width="3456" height="1882" alt="image" src="https://github.com/user-attachments/assets/ec471a55-aba6-4b32-8162-2a0781c36602" />

**Configuration Steps:**

1. Click **Add Function** → **Custom Function**
2. Enter the following parameters:

| Parameter | Value |
|-----------|-------|
| Name | `reschedule_call` |
| Description | Function to reschedule calls with candidates |
| API Endpoint | `https://business.sensehq.com/api/v1/voice-ai/trigger` |
| Timeout | Default |

3. Add JSON Schema:

```json
{
  "type": "object",
  "properties": {
    "agency_id": {
      "enum": [
        "agency_id"
      ]
    },
    "task_type": {
      "enum": [
        "reschedule_call"
      ]
    },
    "try_after": {
      "type": "string"
    },
    "current_time": {
      "type": "string"
    }
  },
  "required": [
    "task_type",
    "agency_id",
    "try_after",
    "current_time"
  ]
}
```

4. Click **Save**

---

## 2. Importing Voice Agent to Sense

### 2.1 Connect to VPN

<img width="856" height="548" alt="image" src="https://github.com/user-attachments/assets/7da90186-a715-4405-8364-9161e1c872c4" />

> ⚠️ **Required:** Connect to AWS VPN before proceeding.

### 2.2 Access Lowcoder

Navigate to **Lowcoder** ([Internal Hub](https://internalhub.prod.sensehq.co/apps/6753f53d273afe4a77cd1012/view))

> ℹ️ **Note:** Lowcoder is an open-source project hosted on Sense servers behind the internalhub domain.

### 2.3 Configure Import Settings

Complete the import form with the following values:

<img width="3456" height="1912" alt="image" src="https://github.com/user-attachments/assets/47151f24-6ecf-40cc-9ea1-5fa58b14f421" />


| Field | Value |
|-------|-------|
| Environment | `prod` |
| Current Agency | `multientity (bullhorn)` |
| Retell Dashboard Bots | Select your newly created bot |
| Entity | `bh_job_submission` |
| Phone Number | Select default phone number |

<img width="3456" height="1908" alt="image" src="https://github.com/user-attachments/assets/ce3b6c5e-86e4-4bd7-8c32-91cdb5823d40" />

Click **Import** to complete the process.

---

## 3. Creating Test Environment

This section demonstrates how to programmatically simulate a candidate applying to a job, which will trigger the **Submittal Workflow**.

### Understanding Submittals and the Workflow

**What is a Submittal?**

A **Submittal** (also called a **Job Submission**) represents a candidate's application to a specific job. It's the core entity that links a candidate to a job opening.

**Workflow Context:**
- **Workflow Type:** Submittal Workflow (triggers when a candidate applies to a job)
- **ATS:** Bullhorn (used with Multi-Entity agency)
- **Submittal Entity Name:** `bh_job_submission` (Bullhorn's ATS name for the Submittal entity)
- **Required Entities:** To create a Submittal, two entities must exist:
  - A **Candidate** (the person applying)
  - A **Job** (the position being applied to)

**What We're Doing:**

We will programmatically simulate a candidate application by:
1. Creating a test candidate record
2. Creating a job submission that links this candidate to a specific job (Job ID: 132)
3. Adding this submission to a list that will be attached to our workflow

This simulates the real-world scenario where a candidate applies to a job through your ATS.

### 3.1 Create Test Candidate

Execute the following cURL command to create a candidate record (use Postman).

> **Important:** Update these values before executing:
> - `external_source_id` (must be unique)
> - `phone` (your test phone number)
> - `name` and `email` (your test details)

```bash
curl --location 'https://system-me.internal-me.sensehq.co/api/v1/ems/bh_candidate/3384364471276985052/create' \
--header 'x-sense-app: dummy' \
--header 'Content-Type: application/json' \
--header 'X-Api-Key: 036e2118-28db-41e8-a755-1fc62a33dfcf' \
--data-raw '{
  "external_source_type": "bullhorn",
  "external_source_id": "UNIQUE_ID_HERE",
  "data": {
    "phone": "+1YOUR_PHONE_NUMBER",
    "name": "Your Name",
    "email": "your.email@example.com",
    "status": "New Lead",
    "firstname": "YourFirstName",
    "lastname": "YourLastName",
    "personsubtype": "Candidate",
    "preferredcontact": "Phone",
    "smsoptin": true,
    "workphone": "+1YOUR_PHONE_NUMBER",
    "address.countryid": 1,
    "address.countryname": "United States",
    "address.countrycode": "US",
    "owner.id": "2842",
    "category.id": 2000000,
    "category.name": "Other Area(s)",
    "isdeleted": false,
    "iseditable": true,
    "dateadded": "2025-10-30"
  }
}'
```

**Response:** Save the returned candidate ID for the next step.

### 3.2 Create Job Submission

This step creates the **Submittal** (the application itself), linking your test candidate to a specific job. **This is the programmatic simulation of a candidate applying to Job ID 132.**

> **Critical:** The `candidate.id` value must match the `external_source_id` from Step 3.1.

> ℹ️ **Note:** We're using **Job ID 132** for this test. This job must already exist in the system.

```bash
curl --location 'https://system-me.internal-me.sensehq.co/api/v1/ems/bh_job_submission/3384364471276985052/create' \
--header 'x-sense-app: dummy' \
--header 'Content-Type: application/json' \
--header 'X-Api-Key: b9ed9c50-fead-4255-80b8-46c52d015a58' \
--data '{
  "external_source_type": "bullhorn",
  "external_source_id": "UNIQUE_SUBMISSION_ID",
  "data": {
    "dateadded": "2025-10-30",
    "sendinguser.id": "44",
    "datewebresponse": "2025-10-30",
    "isdeleted": false,
    "joborder.id": "132",
    "candidate.id": "CANDIDATE_ID_FROM_STEP_3.1",
    "status": "New Lead",
    "datelastmodified": "2025-10-30",
    "joborderowner.id": "174008",
    "clientcorporation.id": "133"
  }
}'
```

**Next Step:** Add this submission to a list that will be attached to the workflow in the next section.

---

## 4. Configuring Automation Workflows (also called J2)

> ℹ️ **Terminology:** Workflows in Sense are also referred to as **J2** (what it was called before). Both terms refer to the same automation system.

### 4.1 Access Workflows

Navigate to [Automation Workflows](https://multientity.sensehq.com/automation-workflows)

### 4.2 Create Workflow

1. Click **Create Workflow**
2. Select **Submittal** as the focus entity

<img width="3456" height="1020" alt="image" src="https://github.com/user-attachments/assets/553493ad-e2ee-427c-bae1-c079f44ddaa4" />

### 4.3 Configure Audience

1. Click **Add Audience List from Scratch**

<img width="3456" height="794" alt="image" src="https://github.com/user-attachments/assets/d5e2e50e-f76f-42ee-b4e4-dc55ce0c846c" />

2. Under **Add Variable**, add `candidate.id`
3. Enter the candidate ID created in Step 3.1

<img width="3456" height="1912" alt="image" src="https://github.com/user-attachments/assets/877fa727-25b5-4153-b20c-1ca07a7214bd" />
  

> ℹ️ **Note:** This list will contain the submission we created, making it the target audience for this workflow.

### 4.4 Add Voice Flow Node

1. Click the **+** button below the trigger
2. Select **Voice Flow**'

<img width="3456" height="1910" alt="image" src="https://github.com/user-attachments/assets/b52571a2-a584-4700-b7ae-d963d7065214" />

3. Choose your imported Retell voice bot from the dropdown

> ℹ️ **Troubleshooting:** If your bot doesn't appear, verify the import in Step 2.

### 4.5 Configure Voice Call Settings

Map the following variables for the voice call:

| Setting | Variable Mapping |
|---------|-----------------|
| Call To | `candidate/phone` |
| Recipient Name | `candidate/first_name` |

<img width="3456" height="902" alt="image" src="https://github.com/user-attachments/assets/ced158dc-878c-47a1-9bf5-0b7db9da19e7" />

### 4.6 Activate Workflow

1. Review your workflow configuration
2. Click **Activate** to enable the workflow
   
<img width="3456" height="1242" alt="image" src="https://github.com/user-attachments/assets/78ea1e68-d754-440e-af27-ca2765d6d60a" />

3. The test call will be initiated to your configured phone number

---

## Troubleshooting

### Voice Bot Not Appearing in Workflow

**Symptoms:** Bot doesn't show in Voice Flow selection dropdown

**Solutions:**
- Verify successful import in Internal Hub (Step 2.3)
- Confirm environment is set to `prod`
- Ensure agency is `multientity (bullhorn)`
- Check entity type is `bh_job_submission`
- Try refreshing the workflow page

### Test Call Not Received

**Symptoms:** No call received after activation

**Solutions:**
- Verify phone number format in candidate data (must include country code)
- Confirm workflow is in **Active** state
- Check that candidate ID matches in all references
- Verify phone service is operational
- Check Retell AI dashboard for call logs

### Import Failures

**Symptoms:** Error during bot import to Sense

**Solutions:**
- Confirm AWS VPN connection is active
- Verify API credentials are current
- Check bot configuration is complete in Retell
- Ensure all required functions are properly configured
- Verify you're accessing Lowcoder (Internal Hub) with proper permissions

### API Request Errors

**Symptoms:** cURL commands return errors

**Solutions:**
- Verify all placeholder values are replaced with actual data
- Ensure `external_source_id` values are unique
- Check API keys are valid and not expired
- Confirm request payload matches schema requirements

---

## Appendix

### API Endpoints Reference

| Service | Endpoint | Purpose |
|---------|----------|---------|
| Voice AI Trigger | `https://business.sensehq.com/api/v1/voice-ai/trigger` | Reschedule calls |
| Candidate Creation | `https://system-me.internal-me.sensehq.co/api/v1/ems/bh_candidate/{id}/create` | Create candidate records |
| Submission Creation | `https://system-me.internal-me.sensehq.co/api/v1/ems/bh_job_submission/{id}/create` | Create job submissions |

### Key Terminology

| Term | Definition |
|------|------------|
| **Submittal** | A candidate's application to a specific job (also called Job Submission) |
| **Job Submission** | Technical entity name for a submittal; in Bullhorn ATS, this is `bh_job_submission` |
| **J2** | Alternative name for Workflows in Sense (both terms refer to the same system) |
| **Submittal Workflow** | An automation workflow that triggers when a candidate applies to a job |
| **Prompt Tree** | Conversational flow logic defining agent behavior |
| **Universal Prompt** | High-level instructions governing overall agent behavior |
| **Multi-Entity** | Internal testing agency environment using Bullhorn ATS |
| **Lowcoder** | Open-source project hosted on Sense servers (accessed via Internal Hub domain) |
| **Dynamic Questions** | Questions generated based on specific job descriptions; criteria provided, not exact questions |
| **Static Questions** | Predefined questions provided at agent creation time; same across all jobs |

