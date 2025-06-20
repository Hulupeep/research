

# **A Detailed Technical Plan for the Integration of Jira Cloud with a Local OpenHands AI Agent**

## **Executive Summary**

This document presents a comprehensive technical plan for the bidirectional integration of Atlassian's Jira Cloud with a locally deployed instance of the OpenHands AI agent. The primary objective is to establish an automated software development workflow where the assignment of a Jira ticket to a designated AI agent user triggers the local OpenHands instance to execute the specified task, with the agent subsequently updating the Jira ticket upon completion. This creates a closed-loop system that offloads development toil, accelerates task resolution, and enhances engineering productivity.1

The proposed solution is a robust, three-component architecture designed for security, scalability, and enhanced agent performance. This architecture consists of **Jira Cloud** as the system of record, a custom-developed **Integration Middleware Service** acting as the central orchestrator, and the **Local OpenHands Instance** as the execution engine. This decoupled design is a strategic necessity, bridging the public-facing Jira Cloud with the private, local OpenHands deployment.

The core workflow proceeds as follows: a Jira webhook, precisely filtered using Jira Query Language (JQL), notifies the middleware of a ticket assignment. The middleware, a publicly accessible service, validates this trigger, enriches the task with historical and contextual data from Jira and its own persistent state store, and then securely invokes the local OpenHands agent via its API. The agent executes the task within its secure Docker sandbox.2 The middleware monitors this long-running, asynchronous process. Upon completion, it retrieves the results and uses the Jira REST API to update the original ticket with a summary of the work and transition its status, completing the automation cycle. This architecture not only provides a secure and reliable integration but also lays the groundwork for a more context-aware "learning apprentice" agent that improves its performance over time by leveraging a knowledge base of previously completed tasks.

## **Section 1: End-to-End System Architecture**

This section establishes the foundational blueprint for the entire system, detailing the distinct roles of each component and the orchestrated interactions between them. A decoupled, three-part architecture is proposed to address the inherent complexities of bridging a public cloud service with a private, local application.

### **1.1 Architectural Overview**

The integration is composed of three primary components, each with a clearly defined responsibility. This separation of concerns ensures security, modularity, and scalability.

* Component 1: Jira Cloud  
  Jira Cloud serves as the authoritative system of record for all development tasks. Its role within this integration is twofold: to initiate the workflow by firing an event-driven webhook and to be the final destination for status updates, comments, and other feedback from the AI agent. It is the user-facing interface where tasks are defined and assigned.4  
* Component 2: Integration Middleware Service  
  This service is the central nervous system of the integration and represents the primary engineering effort. It is a custom-built application that must be deployed to a publicly accessible environment (e.g., a cloud platform like AWS, Google Cloud, or Azure) to receive webhooks from Jira. Its critical responsibilities include:  
  * **Receiving and Authenticating:** Acting as the secure endpoint for incoming Jira webhooks and validating their authenticity.5  
  * **Context Enrichment:** Upon receiving a trigger, it makes API calls back to Jira to fetch the complete ticket details, as webhook payloads are often truncated.4  
  * **State Management:** Maintaining a persistent database to track ongoing tasks and build a historical knowledge base of completed work.  
  * **Orchestration:** Securely constructing a detailed prompt and invoking the local OpenHands agent via its API.  
  * **Asynchronous Monitoring:** Tracking the progress of long-running agent tasks.  
  * **Feedback Loop:** Posting the agent's results and status changes back to the corresponding Jira ticket.  
* Component 3: Local OpenHands Instance  
  The OpenHands agent is the worker component, responsible for executing the development task. As specified, this instance runs locally on a developer workstation or a private server, operating within a secure, sandboxed environment provided by Docker.7 This sandboxing is a core feature of OpenHands, providing isolation and security by preventing agent actions from affecting the host system.2 The agent interacts with the middleware through a local API, receiving its tasks and reporting back upon completion. It leverages a configured Large Language Model (LLM) as its cognitive engine to understand and execute tasks ranging from writing code to running shell commands and interacting with files.3

### **1.2 Data Flow Diagram**

The end-to-end process involves a precise sequence of interactions between the three components, orchestrated by the middleware. The flow is designed to be asynchronous to handle the potentially long duration of AI-driven development tasks.

1. **Trigger:** A user in the Jira Cloud UI assigns a ticket to a specific user account designated for the AI agent (e.g., "openhands-bot").  
2. **Webhook Event:** Jira fires an issue\_updated webhook containing a payload with information about the change. This webhook is sent to the public URL of the Integration Middleware Service.9  
3. **Validation and Ingestion:** The middleware receives the webhook, validates its authenticity, and parses the payload to confirm the trigger condition (i.e., assignment to the agent) has been met. It immediately returns a 200 OK status to Jira to acknowledge receipt.  
4. **Context Fetching:** The middleware extracts the issue key (e.g., "PROJ-123") from the payload and makes an authenticated REST API call to Jira to retrieve the full details of the ticket, including the summary, description, labels, and any custom fields.6  
5. **Historical Context Query:** The middleware queries its internal database for historical context related to the project, such as summaries of previously completed tickets with similar labels or keywords.  
6. **Agent Invocation:** The middleware constructs a detailed, structured prompt containing the full task description and the retrieved historical context. It then makes a POST request to the local OpenHands instance's API to start a new conversation, passing the prompt and any relevant repository information.10  
7. **Task Execution:** The OpenHands backend receives the request and dispatches the task to its sandboxed runtime environment. The agent begins executing the task, which may involve cloning a repository, modifying files, and running tests.2  
8. **Asynchronous Polling:** The middleware, having received a conversation\_id from OpenHands, begins periodically polling the agent's status endpoint (GET /api/conversations/{conversation\_id}) to check for completion.12  
9. **Result Retrieval:** Once the polling mechanism detects that the agent's task status has changed to a terminal state (e.g., "completed"), the middleware retrieves the final results, such as a summary of changes or a link to a pull request.  
10. **Jira Update:** The middleware makes a final authenticated POST request to the Jira REST API's transitions endpoint. This single call updates the ticket's status (e.g., to "In Review" or "Done") and adds a comment containing the agent's summary of the completed work.13

### **1.3 Middleware as a Strategic Necessity**

The selection of a three-component architecture, with the middleware as its centerpiece, is not an arbitrary design choice but a direct consequence of the system's core constraints. The fundamental challenge is bridging a public cloud service (Jira) with a private, local application (OpenHands). Jira Cloud's webhook mechanism can only send HTTP POST requests to publicly accessible HTTPS endpoints.5 The user's OpenHands instance, running on a local machine or a server within a private network, lacks a public IP address and is therefore unreachable by Jira.

This creates an architectural gap that the middleware is designed to fill. It acts as a secure, publicly-facing bridge. However, its role extends far beyond that of a simple proxy. Because it sits at the nexus of all communication, it is the logical place to house the system's core orchestration logic, state management, and context enrichment capabilities. Attempting a direct connection would be insecure and technically infeasible without complex and brittle solutions like NAT traversal or VPN tunnels. By embracing the middleware pattern, the architecture becomes inherently more secure, manageable, and feature-rich. It isolates the local agent from the public internet and centralizes the complex logic of task orchestration, error handling, and context management, making it the most critical component in this integration.

## **Section 2: The Jira Trigger: Configuring and Securing Webhooks**

The entire automated workflow is initiated by an event in Jira. Configuring this trigger correctly is paramount to ensure the system is reliable, efficient, and secure. This involves creating a specific webhook, using JQL to filter events precisely, understanding the data payload, and securing the receiving endpoint.

### **2.1 Webhook Registration and Configuration**

A webhook must be registered within Jira Cloud to listen for the specific event that will trigger the OpenHands agent. This can be accomplished through two primary methods:

* **Jira Administration Console:** A user with Administer Jira global permission can navigate to Settings \> System \> WebHooks and manually create a new webhook.5 The configuration requires providing a name, the URL of the middleware's public endpoint, and selecting the events to subscribe to.  
* **REST API:** For automated or scripted setups, a webhook can be registered programmatically by making a POST request to the /rest/api/2/webhook endpoint. This requires an authenticated session and is the preferred method for integration within a third-party application or deployment script.15

Regardless of the method, the webhook must be configured to subscribe to the jira:issue\_updated event. This event fires whenever an issue's fields are modified, which includes changes to the assignee.9

### **2.2 Precision Targeting with JQL Filtering**

Subscribing to all issue\_updated events for a project would generate a high volume of unnecessary traffic to the middleware, as it would trigger on every comment, label change, or status update. To prevent this, Jira's webhook configuration allows for a JQL (Jira Query Language) filter to be applied.9 This powerful feature ensures that Jira only sends a notification when highly specific conditions are met.

An effective JQL filter for this use case would be:  
project \= "PROJ" AND assignee \= "60a1b2c3d4e5f67890123456" AND status \= "To Do"  
This filter instructs Jira to fire the webhook *only* if the updated issue:

1. Belongs to the project with the key "PROJ".  
2. Is now assigned to the specific user account for the OpenHands agent (using the user's Atlassian Account ID, which is more stable than the display name).  
3. Is in the "To Do" status, indicating it is ready to be worked on.

This pre-filtering at the source is a critical optimization that makes the entire system more efficient and reduces the processing load on the middleware.

### **2.3 Deconstructing the Webhook Payload**

When the jira:issue\_updated event fires, Jira sends a JSON payload to the configured URL. The middleware must parse this payload to extract the necessary information. The payload contains several key objects, but the most crucial for this workflow is the changelog.17

The changelog object details exactly which fields were modified in the transaction that triggered the event. It contains an items array, and each object in the array represents a single field change. The middleware logic must iterate through this array and look for an object where the field property has a value of "assignee". This confirms that the trigger was indeed an assignment change. The to property of that item will contain the account ID of the new assignee, which can be cross-referenced to confirm it is the OpenHands agent's account.

The payload also includes the issue object, which provides the issue's id and key. These identifiers are essential for the middleware to use in subsequent API calls back to Jira to fetch full details or to post updates.

### **2.4 Securing the Webhook Endpoint**

Since the middleware's webhook endpoint is publicly accessible, it must be secured to prevent unauthorized or malicious requests. Jira provides a mechanism to include a secret token in the webhook configuration. This secret can then be used to generate a signature (e.g., a HMAC hash of the request body) that is sent as a request header. The middleware can then compute its own signature using the same secret and compare it to the one received, thus verifying the request's authenticity.14

Alternatively, or in addition, if Atlassian provides a stable list of IP addresses from which webhooks originate, the middleware's infrastructure can be configured with an IP whitelist to only accept requests from those addresses. A shared secret mechanism remains the more flexible and common approach.

### **2.5 The Power of the changelog Object**

Relying on the changelog object is a more robust and reliable design than simply inspecting the issue object's current state. The issue object only shows the final state of the ticket after the update, but it does not reveal *what* changed to cause the event. An issue\_updated event could be triggered by a user adding a comment while the ticket is already assigned to the agent. If the middleware only checked the current assignee, it might incorrectly trigger a new agent task on a ticket that is already in progress or has been handled.

The changelog object provides the necessary delta. By inspecting it, the middleware can definitively answer the question: "Was the assignee field just changed to the OpenHands agent in this specific transaction?" This allows the system to react precisely to the intended trigger event, avoiding redundant or erroneous agent invocations and making the overall automation significantly more accurate and dependable.

## **Section 3: The Integration Middleware Service: Design and Implementation**

The Integration Middleware Service is the architectural linchpin of this solution. Its design must prioritize robustness, security, and clarity. This section outlines a recommended technology stack, API design, core processing logic, and best practices for credential management.

### **3.1 Technology Stack Recommendation**

A lightweight, high-performance web framework is ideal for the middleware. **Python with the FastAPI framework** is a highly recommended choice for several reasons:

* **Performance:** FastAPI is built on Starlette and Pydantic, offering performance comparable to NodeJS and Go.  
* **Ease of Use:** Its modern Python syntax and dependency injection system simplify development.  
* **Automatic Documentation:** It automatically generates interactive API documentation (using Swagger UI and ReDoc), which is invaluable for development, testing, and debugging.  
* **Strong Ecosystem:** Python's extensive libraries, such as requests for making HTTP calls and SQLAlchemy for database interaction, are mature and well-supported.

For deployment, a serverless platform like **AWS Lambda** or **Google Cloud Run** is an excellent fit. These platforms automatically manage scaling and provide a simple way to expose a public HTTPS endpoint, eliminating the need to manage underlying server infrastructure. For simpler setups, a small virtual machine (e.g., an AWS EC2 t3.micro instance) would also suffice.

### **3.2 API Endpoint Design**

The middleware needs to expose at least one primary endpoint to interact with Jira.

* **Endpoint:** POST /webhook/jira  
* **Purpose:** This endpoint will be the target URL for the Jira webhook. Its sole responsibility is to receive, validate, and quickly acknowledge the incoming event from Jira.  
* **Implementation:** To ensure a responsive system and prevent Jira from experiencing timeouts, this endpoint should perform its work asynchronously. Upon receiving a valid request, it should immediately add the task to a background processing queue (using a library like Celery with RabbitMQ/Redis for robust solutions, or FastAPI's built-in BackgroundTasks for simpler use cases) and return a 200 OK response to Jira. The actual processing of the event will happen in the background.

### **3.3 Payload Processing and Context Enrichment**

The background task triggered by the /webhook/jira endpoint contains the core logic of the middleware.

1. **Verify Request:** The first step is to verify the request's authenticity, for example, by checking a shared secret signature as discussed in Section 2.4.  
2. **Parse and Validate Payload:** The JSON payload is parsed. The logic checks the webhookEvent to ensure it is jira:issue\_updated and inspects the changelog to confirm that the assignee field was changed to the designated OpenHands agent user. If these conditions are not met, the process terminates.  
3. **Extract Issue Key:** If the trigger is valid, the issue.key (e.g., "PROJ-123") is extracted from the payload.  
4. **Enrich Context from Jira:** The middleware then uses this key to make an authenticated GET request to the Jira REST API endpoint /rest/api/3/issue/{issueKey}.6 This call is crucial because webhook payloads are often summaries and may not contain the full issue description or custom field data. This step ensures the agent receives the complete context required to perform its task.

### **3.4 Secure Credential Management**

The middleware service is a trusted component that handles sensitive credentials. Hardcoding secrets into the application code is a major security vulnerability and must be avoided.

The service will need to manage several secrets:

* **Jira API Token:** For authenticating API calls back to Jira Cloud. This is generated by a user in their Atlassian account settings.18  
* **OpenHands API Key:** For authenticating calls to the local OpenHands instance's API.  
* **Webhook Secret:** The shared secret used to validate incoming webhooks from Jira.  
* **Database Credentials:** If a separate database is used for state management.

The recommended best practice is to use a dedicated secrets management service, such as **AWS Secrets Manager**, **Google Secret Manager**, or **HashiCorp Vault**. These services provide secure storage, access control, and auditing for secrets. For simpler deployments, environment variables can be used, but they should be injected securely by the deployment platform and not stored in version control.

When authenticating with the Jira Cloud API, the Authorization header must be correctly formatted. It uses Basic Authentication, where the value is the Base64-encoded string of user\_email:api\_token.18 The middleware must construct this header for every API call to Jira.

## **Section 4: Invoking the OpenHands Agent and Managing Execution**

Once the middleware has validated a task and enriched its context, the next phase is to invoke the OpenHands agent and manage its execution lifecycle. This involves translating the Jira ticket into a clear prompt for the agent and handling the asynchronous nature of its work.

### **4.1 API-Driven Invocation of OpenHands**

The middleware will communicate with the local OpenHands instance via its REST API. While the documentation for the self-hosted OpenHands API is found within the source code and mirrors the Cloud API, the primary endpoint for initiating a task is modeled after the POST /api/conversations endpoint described in the OpenHands Cloud API documentation.10

The middleware will send a POST request to the local agent's API endpoint (e.g., http://\<local-openhands-ip\>:3000/api/conversations). The body of this request will be a JSON object containing at a minimum:

* initial\_user\_msg: A string containing the detailed, structured prompt for the agent.  
* repository (optional): The full name of the target Git repository (e.g., owner/repo) if the task requires interacting with a specific codebase.

### **4.2 Crafting a High-Quality Agent Prompt**

The performance of the LLM-powered OpenHands agent is critically dependent on the quality and clarity of its initial prompt. Simply forwarding the raw Jira ticket description is suboptimal. The middleware should act as a "prompt engineer," constructing a well-structured message that maximizes the agent's chance of success.

A recommended structure for the initial\_user\_msg is:

* **Role Definition:** Start by defining the agent's role to set the context.  
  * *Example:* "You are an expert software developer tasked with resolving a Jira issue."  
* **Primary Task:** Clearly state the main objective, using the Jira ticket's summary.  
  * *Example:* "Your primary task is to:."  
* **Detailed Description:** Provide the full, unabridged description from the Jira ticket.  
  * *Example:* "The full details and requirements for this task are as follows:."  
* **Contextual Information (from Section 6):** Inject relevant historical data to guide the agent.  
  * *Example:* "For additional context, your team has recently resolved similar issues. Here are their summaries: \[Inject summaries of 1-3 relevant past tasks from the knowledge base\]."  
* **Acceptance Criteria:** Define what "done" means for the agent.  
  * *Example:* "This task is considered complete when you have implemented the required code changes, all existing and new tests pass, and you have prepared the code for a pull request."

This structured approach provides the agent with a clear mission, comprehensive details, valuable context, and a defined goal, significantly improving its reasoning and execution capabilities.3

### **4.3 Managing Asynchronous Agent Execution**

AI agent tasks, especially those involving coding and testing, are not instantaneous. They are inherently long-running and asynchronous. A simple synchronous request-response pattern where the middleware waits for the agent to finish will fail due to HTTP timeouts and inefficiency. The architecture must embrace this asynchronicity.

1. **Initiation and ID Retrieval:** The POST /api/conversations call to OpenHands should be designed to return immediately with a unique conversation\_id for the newly created task.11  
2. **State Persistence:** The middleware must immediately persist this conversation\_id in its state store, linking it to the corresponding jira\_issue\_key. The initial status should be set to "RUNNING".  
3. **Asynchronous Polling:** A separate, persistent process within the middleware (e.g., a scheduled job running every minute, or a dedicated worker process) is responsible for monitoring ongoing tasks. This process queries the state store for all tasks with a "RUNNING" status.  
4. **Status Check:** For each running task, the monitoring process makes a GET request to the OpenHands API at GET /api/conversations/{conversation\_id}.12 It inspects the  
   status field in the response.  
5. **Triggering the Next Step:** If the status remains "RUNNING", the poller moves to the next task. If the status changes to a terminal state like "completed" or "failed", the poller triggers the final feedback loop (detailed in Section 5), passing along the final results and the associated jira\_issue\_key.

This polling-based state machine architecture is resilient to transient failures and is perfectly suited for managing long-running background processes.

### **Table 4.1: Jira-to-OpenHands Prompt Mapping**

This table formalizes the process of constructing the agent's prompt from Jira ticket data, serving as a clear specification for developers implementing the middleware.

| Jira Field | Transformation Logic | Prompt Section | Example |
| :---- | :---- | :---- | :---- |
| summary | Use directly as the main task title. | **Primary Task** | "Your primary task is to: Add user profile pictures." |
| description | Use the full, unmodified text. | **Detailed Description** | "The full details...: Users should be able to upload a JPG or PNG..." |
| labels | Use as keywords to query the knowledge base. | **Contextual Information** | "For additional context...: 'Implement avatar cropping feature'..." |
| custom\_fields | Extract relevant fields (e.g., "Acceptance Criteria"). | **Acceptance Criteria** | "This task is complete when...: The uploaded image is visible..." |
| issue\_key | Use for logging and tracking. | (Internal Metadata) | \[INFO\] Starting agent for PROJ-123. |

## **Section 5: The Feedback Loop: Updating Jira from the Agent**

The final and most critical phase of the automation is closing the loop: reporting the agent's work back to Jira. This provides visibility to the human team, integrates the AI's output into the standard development workflow, and marks the task as complete. This process is triggered by the middleware's detection of task completion.

### **5.1 Detecting Task Completion**

As outlined in Section 4.3, the middleware employs an asynchronous polling mechanism. The background monitoring process continuously checks the status of active OpenHands conversations. When it polls the GET /api/conversations/{conversation\_id} endpoint and finds that the status has transitioned from "RUNNING" to a final state such as "completed," it knows the agent has finished its work. At this point, the middleware retrieves the final output from the conversation data, which should include a summary of the actions taken, any generated artifacts (like a patch file), or a link to a pull request.

### **5.2 Updating the Jira Ticket via REST API**

With the agent's results in hand, the middleware uses its stored Jira API token to perform the final update. It is crucial to use the correct Jira API endpoint for this operation to ensure atomicity.

The correct endpoint is POST /rest/api/3/issue/{issueIdOrKey}/transitions.13

Using this endpoint is superior to making separate calls to update the status and add a comment. The transitions endpoint allows both actions to be performed in a single, atomic API call. This prevents race conditions or partial updates where the status might change but the explanatory comment fails to be added. The PUT /rest/api/3/issue/{issueIdOrKey} endpoint, used for general field editing, explicitly does not support status transitions and will ignore such requests.13

The request body sent to this endpoint will contain:

* transition: An object specifying the target transition, for example, { "id": "5" }. The numeric id for a specific transition (e.g., "Resolve Issue," "Move to In Review") must be discovered dynamically beforehand by making a GET request to the same /transitions endpoint, which lists all available transitions from the issue's current state.  
* update: An object that can contain fields to be updated on the transition screen, most importantly a comment array to add the agent's summary.

### **5.3 Formatting Comments with Atlassian Document Format (ADF)**

Modern Jira Cloud issues do not use plain text or Markdown for rich text fields like the description or comments. Instead, they use the Atlassian Document Format (ADF), which is a JSON-based structure.13 The middleware must therefore construct a valid ADF JSON object to post the comment.

For example, to post a comment that says "Task completed. Pull request created at \[link\]," the middleware would construct an ADF object similar to this:

JSON

{  
  "body": {  
    "type": "doc",  
    "version": 1,  
    "content":  
          }  
        \]  
      }  
    \]  
  }  
}

This allows for the creation of rich comments with links, bold text, lists, and other formatting, providing a much cleaner and more useful summary for the human developers who will review the agent's work.

### **Table 5.1: Jira API Update Actions**

This table provides a quick-reference guide for the key API interactions required to implement the feedback loop.

| Action | HTTP Method | Endpoint | Example Request Body Snippet |
| :---- | :---- | :---- | :---- |
| Get Available Transitions | GET | /rest/api/3/issue/{issueIdOrKey}/transitions | (No body required) |
| Transition Issue & Add Comment | POST | /rest/api/3/issue/{issueIdOrKey}/transitions | {"transition": {"id": "5"}, "update": {"comment": \[{"add": {"body":...}}\]}} |

## **Section 6: Advanced State and Context Management for a Smarter Agent**

To fulfill the forward-looking requirement for an agent that understands project context, including past work and future roadmap items, the middleware must evolve beyond a stateless orchestrator. By incorporating a persistent state store, it can transform the OpenHands agent from a simple tool into a "learning apprentice" that becomes more effective over time.

### **6.1 Designing a Persistent State Store**

The middleware will be connected to a database to manage state. For initial development, a simple file-based database like **SQLite** is sufficient. For production environments, a more robust relational database like **PostgreSQL** is recommended. The core schema will track the lifecycle of each automated task.

A basic tasks table would include columns such as:

* id (Primary Key)  
* jira\_issue\_key (e.g., "PROJ-123")  
* openhands\_conversation\_id  
* status (e.g., "running", "completed", "failed")  
* created\_at  
* completed\_at

This table is essential for the asynchronous polling mechanism and for ensuring that tasks are not processed more than once.

### **6.2 Building a Project Knowledge Base**

The true power of the state store comes from using it to build a long-term memory for the agent. This moves beyond simple task tracking to creating a project-specific knowledge base.

The process is as follows:

1. When a task is successfully completed, the middleware retrieves the initial Jira ticket summary/description (the "problem") and the agent's final summary of work (the "solution").  
2. It then generates a concise, combined summary of this "problem-solution pair." For more advanced implementations, this text can be converted into a vector embedding using a sentence-transformer model.  
3. This summary or embedding is stored in a new knowledge\_base table, tagged with the project key (e.g., "PROJ"), the Jira issue key, and relevant labels from the ticket.

### **6.3 Dynamic Context Injection**

This knowledge base is then leveraged to make the agent smarter on subsequent tasks. When a *new* task is initiated (during the "Context Enrichment" step in the data flow), the middleware performs an additional step:

1. It takes the summary and labels of the new Jira ticket.  
2. It queries the knowledge\_base table to find historical tasks from the same project that are semantically similar. This can be a simple keyword search on the summaries or a more sophisticated vector similarity search if embeddings are used.  
3. It retrieves the top 2-3 most relevant problem-solution summaries from the past.  
4. These summaries are then dynamically injected into the "Contextual Information" section of the prompt sent to the OpenHands agent, as described in Section 4.2.

### **6.4 From Stateless Worker to Learning Apprentice**

This stateful architecture fundamentally changes the nature of the AI agent. A stateless agent approaches every task with no memory of the past. It might solve the same type of problem in slightly different ways each time or fail to recognize established patterns within a codebase. This can lead to inconsistent or suboptimal solutions.

By providing historical context, the middleware equips the agent with a form of institutional knowledge. When presented with a new bug fix, the agent might be reminded of how a similar bug was fixed before, guiding it toward a solution that aligns with the project's existing architecture. When asked to add a new feature, it can see how previous features were implemented, helping it maintain stylistic and structural consistency.

This mechanism directly addresses the user's desire for an agent that understands "what other tickets it has completed." It transforms the agent from a tool that executes isolated commands into an apprentice that learns from experience, progressively improving its alignment with the project's specific needs and conventions. This feedback loop of execution, storage, and retrieval is the key to unlocking higher levels of autonomous performance.

## **Section 7: Implementation Guide and Configuration**

This section provides practical guidance and concrete examples to accelerate the development and deployment of the integration. It covers the setup of the local OpenHands instance and provides code snippets for the middleware service.

### **7.1 OpenHands Local Instance Configuration**

The OpenHands agent is designed to run locally using Docker, which provides a consistent and isolated execution environment.2

**Setup Steps:**

1. **System Requirements:** Ensure the host machine meets the requirements, which typically include Docker Desktop (for Mac/Windows) or Docker Engine (for Linux), and a minimum of 4GB of RAM. For Windows users, WSL 2 is recommended.7  
2. **Docker Installation:** Follow the official OpenHands documentation to run the application using the provided docker run command.8 This command will pull the necessary  
   openhands and runtime images.  
3. **Environment Variables:** The agent's behavior is configured via environment variables passed to the Docker container. The most critical variables for this integration are:  
   * LLM\_MODEL: Specifies the Large Language Model to use (e.g., anthropic/claude-3-5-sonnet-20240620). The agent's performance is highly dependent on the capability of the chosen model.21  
   * LLM\_API\_KEY: The API key for the chosen LLM provider.7  
   * SANDBOX\_VOLUMES: This variable is used to map a directory from the host machine into the agent's sandboxed container (e.g., \-v /path/to/projects:/workspace). This gives the agent access to the project codebase it needs to modify.21  
4. **API Accessibility:** Ensure that the port mapped for the OpenHands UI/API (typically port 3000\) is accessible from the machine hosting the middleware service. Since both are on the local network in this scenario, this usually involves ensuring no firewalls are blocking the connection between the two machines.

### **7.2 Code Snippets for Middleware Service**

The following Python code snippets, using the FastAPI framework, provide a starting point for implementing the middleware's core logic.

**Webhook Receiver Endpoint:**

Python

\# main.py  
from fastapi import FastAPI, Request, BackgroundTasks, HTTPException  
import hmac  
import hashlib  
import os

app \= FastAPI()  
JIRA\_WEBHOOK\_SECRET \= os.environ.get("JIRA\_WEBHOOK\_SECRET")

async def process\_jira\_event(payload: dict):  
    \# Placeholder for the main processing logic  
    print("Processing Jira event in the background...")  
    \# 1\. Extract issue key  
    \# 2\. Call Jira API for full details  
    \# 3\. Query knowledge base  
    \# 4\. Construct prompt  
    \# 5\. Call OpenHands API  
    \# 6\. Store task in DB  
    pass

@app.post("/webhook/jira")  
async def handle\_jira\_webhook(request: Request, background\_tasks: BackgroundTasks):  
    \# 1\. Validate signature  
    signature \= request.headers.get("X-Hub-Signature")  
    if not signature:  
        raise HTTPException(status\_code=400, detail="Missing signature")  
      
    body \= await request.body()  
    expected\_signature \= "sha256=" \+ hmac.new(  
        key=JIRA\_WEBHOOK\_SECRET.encode(),  
        msg=body,  
        digestmod=hashlib.sha256  
    ).hexdigest()

    if not hmac.compare\_digest(expected\_signature, signature):  
        raise HTTPException(status\_code=403, detail="Invalid signature")

    \# 2\. Parse payload and check for assignee change  
    payload \= await request.json()  
    if payload.get("webhookEvent") \== "jira:issue\_updated":  
        changelog \= payload.get("changelog", {})  
        for item in changelog.get("items",):  
            if item.get("field") \== "assignee":  
                \# Add to background tasks for processing  
                background\_tasks.add\_task(process\_jira\_event, payload)  
                break

    return {"status": "ok"}

**Function to Call OpenHands API:**

Python

\# openhands\_client.py  
import requests  
import os

OPENHANDS\_API\_URL \= os.environ.get("OPENHANDS\_API\_URL", "http://localhost:3000/api")  
OPENHANDS\_API\_KEY \= os.environ.get("OPENHANDS\_API\_KEY")

def start\_openhands\_task(prompt: str, repository: str) \-\> str:  
    """Starts a new task in OpenHands and returns the conversation ID."""  
    headers \= {  
        "Authorization": f"Bearer {OPENHANDS\_API\_KEY}",  
        "Content-Type": "application/json"  
    }  
    data \= {  
        "initial\_user\_msg": prompt,  
        "repository": repository  
    }  
    response \= requests.post(f"{OPENHANDS\_API\_URL}/conversations", headers=headers, json=data)  
    response.raise\_for\_status()  
      
    conversation \= response.json()  
    return conversation.get("conversation\_id")

### **7.3 Environment Configuration**

The middleware service will require the following environment variables to be configured in its deployment environment:

* JIRA\_URL: The base URL of the Jira Cloud instance (e.g., https://your-domain.atlassian.net).  
* JIRA\_USER\_EMAIL: The email address of the user account whose API token is being used.  
* JIRA\_API\_TOKEN: The API token for making authenticated calls to Jira.  
* JIRA\_WEBHOOK\_SECRET: The shared secret for validating incoming webhooks.  
* OPENHANDS\_API\_URL: The full URL to the local OpenHands API (e.g., http://192.168.1.100:3000/api).  
* OPENHANDS\_API\_KEY: The API key for the OpenHands service.  
* DATABASE\_URL: The connection string for the persistent state store database.

## **Section 8: Security, Error Handling, and Scalability**

For the integration to be reliable and suitable for a production or team environment, it must be designed with security, robust error handling, and scalability in mind. This section addresses these critical non-functional requirements.

### **8.1 End-to-End Security Checklist**

A multi-layered approach to security is essential to protect the system and the codebases it interacts with.

* **Webhook Security:** The public webhook endpoint must be secured. As detailed previously, this involves validating a signature using a shared secret.14 All communication must use HTTPS.  
* **Credential Storage:** All secrets (API keys, tokens, database passwords) must be stored in a dedicated secrets management system (e.g., AWS Secrets Manager, HashiCorp Vault). They should never be hardcoded in source code or committed to version control.18  
* **API Key Rotation:** Establish a policy for regularly rotating all API keys and tokens. This limits the window of exposure if a key is ever compromised.  
* **Network Security:** The middleware service should be protected by a firewall that restricts traffic to only the necessary ports (e.g., port 443 for HTTPS). The local OpenHands instance should remain on a private network, accessible only by the middleware service, not the public internet. The principle of least privilege should be applied to all network access rules.  
* **Agent Sandbox:** The inherent security of the OpenHands Docker sandbox is a key strength, as it isolates file system access and command execution, preventing the agent from impacting the host system.2

### **8.2 Robust Error Handling and Logging**

A production system must be resilient to failure. The middleware should anticipate and gracefully handle various error conditions.

* **Jira API Unavailability:** Calls to the Jira REST API may fail due to network issues or temporary outages. The middleware should implement a retry mechanism with exponential backoff for these calls to handle transient errors.  
* **OpenHands Agent Failure:** An agent's task may fail for numerous reasons (e.g., code that won't compile, failing tests, inability to understand the prompt). The middleware's polling mechanism should detect a "failed" status from the OpenHands API. Upon detection, it should update the Jira ticket with a comment indicating the failure, including any error logs from the agent, and transition the ticket to a specific "Failed" or "Needs Attention" status. This provides immediate feedback to the human team.  
* **Invalid Payloads:** The webhook endpoint should perform strict validation on incoming payloads and log any malformed or unexpected data before discarding it, preventing crashes due to bad input.  
* **Comprehensive Logging:** Structured logging should be implemented at every significant step of the process (e.g., webhook received, Jira API called, agent invoked, status polled, ticket updated). These logs are indispensable for debugging issues and monitoring the health of the system.

### **8.3 Scalability Considerations**

The initial design using background tasks within a single middleware instance is suitable for low to moderate task volumes. However, as usage grows, this design can become a bottleneck.

* **Low Volume:** For a single team's use, FastAPI's BackgroundTasks or a simple threading model may be sufficient to handle asynchronous processing.  
* **High Volume:** To handle a higher throughput of tasks or to support multiple concurrent agent executions, a more robust architecture is required. A message queuing system like **RabbitMQ** or **Celery** should be introduced.  
  * In this model, the /webhook/jira endpoint's only job is to perform validation and then publish a message containing the task details to a queue.  
  * A separate pool of worker processes subscribes to this queue. Each worker pulls a message, executes the entire task lifecycle (context enrichment, agent invocation, polling, and final update), and then pulls the next message.  
  * This architecture decouples ingestion from processing, allows for horizontal scaling by simply adding more workers, and provides better resilience, as tasks in the queue will not be lost if a worker process crashes.

#### **Works cited**

1. All Hands AI, accessed June 19, 2025, [https://www.all-hands.dev/](https://www.all-hands.dev/)  
2. Runtime Architecture \- All Hands Docs \- OpenHands, accessed June 19, 2025, [https://docs.all-hands.dev/usage/architecture/runtime](https://docs.all-hands.dev/usage/architecture/runtime)  
3. OpenHands: The Open Source Devin AI Alternative \- Apidog, accessed June 19, 2025, [https://apidog.com/blog/openhands-the-open-source-devin-ai-alternative/](https://apidog.com/blog/openhands-the-open-source-devin-ai-alternative/)  
4. Jira REST API examples \- Atlassian Developer, accessed June 19, 2025, [https://developer.atlassian.com/server/jira/platform/jira-rest-api-examples/](https://developer.atlassian.com/server/jira/platform/jira-rest-api-examples/)  
5. Manage webhooks \- Atlassian Support, accessed June 19, 2025, [https://support.atlassian.com/jira-cloud-administration/docs/manage-webhooks/](https://support.atlassian.com/jira-cloud-administration/docs/manage-webhooks/)  
6. Jira Cloud Rest API Create Issue \- Atlassian Developer, accessed June 19, 2025, [https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/)  
7. Getting Started \- All Hands Docs \- OpenHands, accessed June 19, 2025, [https://docs.all-hands.dev/usage/local-setup](https://docs.all-hands.dev/usage/local-setup)  
8. All-Hands-AI/OpenHands: OpenHands: Code Less, Make ... \- GitHub, accessed June 19, 2025, [https://github.com/All-Hands-AI/OpenHands](https://github.com/All-Hands-AI/OpenHands)  
9. Webhook \- Atlassian Developer, accessed June 19, 2025, [https://developer.atlassian.com/cloud/jira/platform/modules/webhook/](https://developer.atlassian.com/cloud/jira/platform/modules/webhook/)  
10. Write documentation for OpenHands Cloud API interface · Issue ..., accessed June 19, 2025, [https://github.com/All-Hands-AI/OpenHands/issues/8126](https://github.com/All-Hands-AI/OpenHands/issues/8126)  
11. Introduction \- All Hands Docs, accessed June 19, 2025, [https://docs.all-hands.dev/swagger-ui/\#/default/newConversation](https://docs.all-hands.dev/swagger-ui/#/default/newConversation)  
12. Backend Architecture \- All Hands Docs, accessed June 19, 2025, [https://docs.all-hands.dev/usage/architecture/backend](https://docs.all-hands.dev/usage/architecture/backend)  
13. Jira Cloud platform REST API documentation \- Atlassian Developer, accessed June 19, 2025, [https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/\#api-rest-api-3-issue-issueidorkey-put](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/#api-rest-api-3-issue-issueidorkey-put)  
14. Webhooks \- Atlassian Developer, accessed June 19, 2025, [https://developer.atlassian.com/cloud/jira/software/webhooks/](https://developer.atlassian.com/cloud/jira/software/webhooks/)  
15. Guide to Webhooks (with Examples from Jira) \- Merge.dev, accessed June 19, 2025, [https://www.merge.dev/blog/guide-to-webhooks-with-examples-from-jira](https://www.merge.dev/blog/guide-to-webhooks-with-examples-from-jira)  
16. The Jira Cloud platform REST API \- developer Atlassian., accessed June 19, 2025, [https://developer.atlassian.com/cloud/jira/platform/rest/v2/api-group-webhooks/](https://developer.atlassian.com/cloud/jira/platform/rest/v2/api-group-webhooks/)  
17. Webhooks \- Atlassian Developer, accessed June 19, 2025, [https://developer.atlassian.com/cloud/jira/platform/webhooks/](https://developer.atlassian.com/cloud/jira/platform/webhooks/)  
18. How can I create a Jira Issue through POST API? \- Atlassian Community, accessed June 19, 2025, [https://community.atlassian.com/forums/Jira-questions/How-can-I-create-a-Jira-Issue-through-POST-API/qaq-p/2763634](https://community.atlassian.com/forums/Jira-questions/How-can-I-create-a-Jira-Issue-through-POST-API/qaq-p/2763634)  
19. Running OpenHands, accessed June 19, 2025, [https://docs.all-hands.dev/modules/usage/installation](https://docs.all-hands.dev/modules/usage/installation)  
20. Build an App with AI in Minutes using OpenHands AI Engineer – Install Locally \- NodeShift, accessed June 19, 2025, [https://nodeshift.com/blog/build-an-app-with-ai-in-minutes-using-openhands-ai-engineer-install-locally](https://nodeshift.com/blog/build-an-app-with-ai-in-minutes-using-openhands-ai-engineer-install-locally)  
21. CLI \- All Hands Docs \- OpenHands, accessed June 19, 2025, [https://docs.all-hands.dev/usage/how-to/cli-mode](https://docs.all-hands.dev/usage/how-to/cli-mode)  
22. The OpenHands CLI: AI-Powered Development in Your Terminal \- All Hands AI, accessed June 19, 2025, [https://www.all-hands.dev/blog/the-openhands-cli-ai-powered-development-in-your-terminal](https://www.all-hands.dev/blog/the-openhands-cli-ai-powered-development-in-your-terminal)