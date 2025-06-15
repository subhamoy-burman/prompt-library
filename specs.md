Of course. This is a fantastic, real-world use case that perfectly illustrates the power of combining structured logic with LLMs. Building a detailed specification is the first step to success.

Here is a detailed project specification for the **iTero Intelligent Troubleshooting Assistant**.

---

### **Project Specification: iTero Intelligent Troubleshooting Assistant**

**Version:** 1.0
**Date:** June 13, 2025

#### **1. Overview & Vision**

This document outlines the design and architecture for an "Intelligent Troubleshooting Assistant" to be integrated into the iTero scanner's "Troubleshoot & Report" software module.

The vision is to create a dynamic, conversational AI agent that guides dental professionals and technicians through complex troubleshooting scenarios in real-time. By analyzing scanner-generated logs and leveraging a comprehensive knowledge base, the assistant will provide contextual, step-by-step remedies. If the automated process fails to resolve the issue, the system will seamlessly escalate by creating a formal support ticket and providing the user with immediate confirmation and follow-up details. This will reduce support call volume, decrease user downtime, and improve the overall customer experience.

#### **2. Key Features & Functionality**

*   **Log-Driven Analysis:** The workflow initiates by ingesting and analyzing logs directly from the iTero scanner application.
*   **Conversational Interaction:** The user interacts with the assistant via a chat interface, receiving one suggestion at a time and providing feedback on its success.
*   **RAG from Vector DB:** The assistant's suggestions are grounded in a knowledge base stored in a **Pinecone** vector database, ensuring answers are accurate and based on official documentation.
*   **Stateful Conversation:** The assistant maintains the context of the conversation, tracking which remedies have been suggested and their outcomes.
*   **Automated Escalation:** When all knowledge base remedies are exhausted and the issue persists, the system automatically creates a support ticket via a service API call.
*   **Confirmation & Closure:** Upon successful ticket creation, the user is immediately provided with a ticket number and a support callback phone number, concluding the automated interaction.

#### **3. System Architecture**

The solution will be built within the **Azure AI Foundry** ecosystem, primarily using **Prompt Flow** as the orchestration engine.

**Components:**

1.  **Scanner Application Module (Client):** The existing "Troubleshoot & Report" module on the iTero scanner. This is the user-facing component.
2.  **Azure AI Hub Project:** The central development and hosting environment for the Prompt Flow.
3.  **Prompt Flow (Chat Flow):** The core orchestration engine that defines the entire troubleshooting logic.
4.  **Azure OpenAI Service:** Provides the LLMs for reasoning, analysis, and natural language generation.
    *   **Embedding Model:** `subhamoy-text-embeddings` (for vectorizing logs and queries).
    *   **Reasoning/Chat Model:** `gpt-4o` (for complex analysis, planning, and response generation).
5.  **Pinecone Vector Database (Knowledge Base):** The long-term memory of the system.
    *   **Index:** An index named `itero-kb` containing vector embeddings of chunked troubleshooting manuals, error code descriptions, technical notes, and past resolved tickets.
    *   **Metadata:** Each vector will have associated metadata, including `source_document`, `document_section`, and a list of `error_codes` it pertains to.
6.  **Internal Ticketing System (API):** A hypothetical but representative internal service for creating support tickets.

**Diagram:**
```
[Scanner App UI] <--> [Azure AI Hub: Prompt Flow Endpoint]
      |                                  |
      |                                  |---[Azure OpenAI Service (GPT-4o, Embeddings)]
      |                                  |
      |                                  |---[Pinecone Vector DB (Knowledge Base)]
      |                                  |
      |                                  |---[Ticketing System API]
      |
(User Interaction)
```

#### **4. Data & Logic Flow**

The entire process will be orchestrated as a **Chat Flow** within Prompt Flow to manage conversational history automatically.

1.  **Initiation:** The user clicks "Troubleshoot" in the scanner app. The app sends the recent scanner logs as the initial input to the Prompt Flow endpoint.
2.  **Log Analysis:** The flow's first step is a Python tool that pre-processes the raw logs. It extracts critical error codes, timestamps, and key status messages, summarizing them into a clean, structured "Problem Statement".
3.  **Knowledge Retrieval:** A second Python tool takes the extracted error codes and problem statement, generates an embedding using Azure OpenAI, and queries the Pinecone vector database to retrieve a list of the top 3-5 most relevant troubleshooting guide chunks.
4.  **Stateful Interaction Loop (Core of the Chat Flow):**
    a.  An LLM tool (`State_Manager_LLM`) analyzes the current `chat_history`, the original problem statement, and the full list of retrieved knowledge base steps.
    b.  Its primary job is to decide the `next_action`. The prompt will instruct it to output a JSON object like `{"action": "ACTION_TYPE", "details": "..."}`.
    c.  Possible `ACTION_TYPE` values are:
        *   `SUGGEST_REMEDY`: If there are untried steps in the knowledge base. The `details` will contain the text of the next remedy to suggest.
        *   `AWAITING_USER_FEEDBACK`: After a remedy has been suggested.
        *   `ALL_STEPS_EXHAUSTED`: If the user has confirmed that all relevant remedies from the knowledge base have failed.
        *   `CLARIFY`: If the user's response is ambiguous.
    d.  This decision is passed to the next nodes.
5.  **Automated Escalation (Conditional Logic):**
    a.  A Python tool (`Create_Support_Ticket_Python`) is configured with a conditional `Activate config`.
    b.  **Condition:** `when ${State_Manager_LLM.output.action} == "ALL_STEPS_EXHAUSTED"`
    c.  This tool makes a `POST` request to the internal ticketing system API, sending the problem statement and conversation history.
6.  **Response Generation:**
    a.  A final Python tool (`Format_Final_Response_Python`) constructs the message to send back to the user.
    b.  If the next action is `SUGGEST_REMEDY`, it formats the suggestion clearly.
    c.  If a ticket was created (i.e., the `Create_Support_Ticket_Python` node ran and returned a ticket ID), it formats the confirmation message with the ticket number and callback details.
7.  **Output:** The flow's `chat_output` is the formatted message, which is displayed to the user in the scanner's chat interface.

---

#### **5. Prompt Flow Design: Nodes and Code**

**Flow Type:** Chat Flow

**Inputs:**
*   `chat_history`: (Managed by Prompt Flow)
*   `scanner_logs`: (string, initial input)
*   `user_response`: (string, subsequent inputs)

**Nodes:**

**1. `Analyze_Logs_Python` (Python Tool)**
*   **Inputs:** `logs: ${inputs.scanner_logs}`
*   **Code:**
    ```python
    import re
    from promptflow.core import tool

    @tool
    def analyze_scanner_logs(logs: str) -> dict:
        # This function would contain sophisticated parsing logic.
        # For this spec, we'll simulate it with regex.
        error_codes = re.findall(r"ERROR_CODE:\s*(\d+)", logs)
        critical_warnings = re.findall(r"WARN:\s*(Critical.*)", logs)
        
        problem_summary = f"The user's scanner reported the following error codes: {', '.join(error_codes)}. "
        if critical_warnings:
            problem_summary += f"It also logged critical warnings: {'; '.join(critical_warnings)}."
            
        return {
            "summary": problem_summary,
            "error_codes": list(set(error_codes)) # Unique error codes
        }
    ```

**2. `Retrieve_KB_from_Pinecone_Python` (Python Tool)**
*   **Inputs:** `problem_data: ${Analyze_Logs_Python.output}`
*   **Code:**
    ```python
    import os
    from promptflow.core import tool, Connection
    from openai import AzureOpenAI
    from pinecone import Pinecone # Assumes pinecone-client is installed

    @tool
    def retrieve_from_pinecone(problem_data: dict, open_ai_connection: Connection) -> list:
        # Initialize clients
        pinecone_client = Pinecone(api_key=os.environ.get("PINECONE_API_KEY"))
        openai_client = AzureOpenAI(api_key=open_ai_connection.api_key, api_version="2023-07-01-preview", azure_endpoint=open_ai_connection.api_base)
        
        # Get query embedding
        embedding_model = os.environ.get("AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME", "subhamoy-text-embeddings")
        response = openai_client.embeddings.create(input=problem_data["summary"], model=embedding_model)
        query_vector = response.data[0].embedding
        
        # Query Pinecone
        index = pinecone_client.Index("itero-kb")
        results = index.query(
            vector=query_vector,
            top_k=5, # Get top 5 relevant chunks
            include_metadata=True
        )
        
        # Format results
        retrieved_docs = []
        for match in results['matches']:
            retrieved_docs.append({
                "source": match['metadata']['source_document'],
                "text": match['metadata']['chunk_text']
            })
            
        return retrieved_docs
    ```

**3. `State_Manager_LLM` (LLM Tool)**
*   **Inputs:**
    *   `problem_summary: ${Analyze_Logs_Python.output.summary}`
    *   `retrieved_docs: ${Retrieve_KB_from_Pinecone_Python.output}`
    *   `chat_history: ${inputs.chat_history}`
*   **Prompt (Jinja):**
    ```jinja
    system:
    You are a sophisticated troubleshooting state manager for iTero scanners. Your role is to decide the next action in a troubleshooting conversation.
    Analyze the user's original problem, the conversation history, and the provided knowledge base documents.
    Based on this analysis, determine if you should suggest the next untried remedy, if all remedies have been exhausted, or if you need to ask a clarifying question.
    Your output MUST be a single, valid JSON object with the keys "action" and "details".
    Possible values for "action" are: "SUGGEST_REMEDY", "ALL_STEPS_EXHAUSTED", "CLARIFY".
    - If suggesting a remedy, "details" should contain the exact text of the remedy to provide to the user.
    - If all steps are exhausted, "details" should be "All knowledge base steps have been attempted.".
    - If clarifying, "details" should contain the question to ask the user.

    user:
    **Original Problem:**
    {{problem_summary}}

    **Conversation History:**
    {% for item in chat_history %}
    User said: '{{item.inputs.user_response}}'
    Assistant suggested: '{{item.outputs.chat_output}}'
    {% endfor %}

    **Available Knowledge Base Steps:**
    {{retrieved_docs}}

    **Decision in JSON format:**
    ```
*   **Deployment:** `gpt-4o`

**4. `Create_Support_Ticket_Python` (Python Tool)**
*   **Activate config:** `when ${State_Manager_LLM.output.action} == "ALL_STEPS_EXHAUSTED"`
*   **Inputs:** `full_history: ${inputs.chat_history}`
*   **Code:**
    ```python
    import os
    import json
    import requests
    from promptflow.core import tool

    @tool
    def create_ticket(full_history: list) -> dict:
        ticketing_api_url = os.environ.get("TICKETING_SYSTEM_ENDPOINT")
        api_key = os.environ.get("TICKETING_SYSTEM_API_KEY")
        
        payload = {
            "subject": "iTero Automated Troubleshooting Escalation",
            "description": "User was unable to resolve issue with AI assistant.",
            "transcript": json.dumps(full_history, indent=2)
        }
        
        headers = {"Authorization": f"Bearer {api_key}"}
        
        try:
            response = requests.post(ticketing_api_url, json=payload, headers=headers)
            response.raise_for_status() # Raise an exception for bad status codes
            ticket_data = response.json()
            return {
                "ticket_id": ticket_data.get("id"),
                "callback_number": ticket_data.get("support_phone")
            }
        except requests.exceptions.RequestException as e:
            # Handle API call failure
            return {"ticket_id": "ERROR", "callback_number": "N/A"}
    ```

**5. `Format_Final_Response_Python` (Python Tool)**
*   **Inputs:**
    *   `state_decision: ${State_Manager_LLM.output}`
    *   `ticket_info: ${Create_Support_Ticket_Python.output}` (This will be null if the node was skipped)
*   **Code:**
    ```python
    from promptflow.core import tool

    @tool
    def format_response(state_decision: dict, ticket_info: dict = None) -> str:
        action = state_decision.get("action")
        details = state_decision.get("details")

        if ticket_info and ticket_info.get("ticket_id") != "ERROR":
            return (f"I've exhausted all the steps from our knowledge base. I have created a support ticket for you. "
                    f"Your ticket number is {ticket_info['ticket_id']}. A support agent will call you back at {ticket_info['callback_number']}. "
                    f"Is there anything else I can help you with today?")
        elif action == "ALL_STEPS_EXHAUSTED":
             return "I see we've tried all available steps. I will now create a support ticket for you. Please wait a moment."
        elif action == "SUGGEST_REMEDY":
            return f"{details} Please try this and let me know if it resolves the issue."
        elif action == "CLARIFY":
            return details
        else:
            return "I'm sorry, I'm having trouble determining the next step. Could you please rephrase your last response?"
    ```

**Outputs:**
*   `chat_output`: `${Format_Final_Response_Python.output}`

---

#### **6. External Systems & Dependencies**

*   **Pinecone:** Requires an active account, API key, and a pre-populated index.
*   **Internal Ticketing System:** Requires a stable, documented REST API endpoint (`POST /api/tickets`) and an authentication mechanism (e.g., API Key/Bearer Token).
*   **Scanner Application:** Must be capable of making HTTPS requests to the deployed Prompt Flow endpoint and rendering a chat UI.