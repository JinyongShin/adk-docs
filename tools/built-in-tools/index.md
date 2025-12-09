# Built-in tools

These built-in tools provide ready-to-use functionality such as Google Search or code executors that provide agents with common capabilities. For instance, an agent that needs to retrieve information from the web can directly use the **google_search** tool without any additional setup.

## How to Use

1. **Import:** Import the desired tool from the tools module. This is `agents.tools` in Python, `google.golang.org/adk/tool/geminitool` in Go, or `com.google.adk.tools` in Java.
1. **Configure:** Initialize the tool, providing required parameters if any.
1. **Register:** Add the initialized tool to the **tools** list of your Agent.

Once added to an agent, the agent can decide to use the tool based on the **user prompt** and its **instructions**. The framework handles the execution of the tool when the agent calls it. Important: check the ***Limitations*** section of this page.

## Available Built-in tools

Note: Go supports the Google Search tool and other built-in tools via the `geminitool` package. Note: Java only supports Google Search and Code Execution tools currently.

### Google Search

Supported in ADKPython v0.1.0Go v0.1.0Java v0.2.0

The `google_search` tool allows the agent to perform web searches using Google Search. The `google_search` tool is only compatible with Gemini 2 models. For further details of the tool, see [Understanding Google Search grounding](https://google.github.io/adk-docs/grounding/google_search_grounding/index.md).

Additional requirements when using the `google_search` tool

When you use grounding with Google Search, and you receive Search suggestions in your response, you must display the Search suggestions in production and in your applications. For more information on grounding with Google Search, see Grounding with Google Search documentation for [Google AI Studio](https://ai.google.dev/gemini-api/docs/grounding/search-suggestions) or [Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs/grounding/grounding-search-suggestions). The UI code (HTML) is returned in the Gemini response as `renderedContent`, and you will need to show the HTML in your app, in accordance with the policy.

```py
# Copyright 2025 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools import google_search
from google.genai import types

APP_NAME="google_search_agent"
USER_ID="user1234"
SESSION_ID="1234"


root_agent = Agent(
    name="basic_search_agent",
    model="gemini-2.0-flash",
    description="Agent to answer questions using Google Search.",
    instruction="I can answer your questions by searching the internet. Just ask me anything!",
    # google_search is a pre-built tool which allows the agent to perform Google searches.
    tools=[google_search]
)

# Session and Runner
async def setup_session_and_runner():
    session_service = InMemorySessionService()
    session = await session_service.create_session(app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID)
    runner = Runner(agent=root_agent, app_name=APP_NAME, session_service=session_service)
    return session, runner

# Agent Interaction
async def call_agent_async(query):
    content = types.Content(role='user', parts=[types.Part(text=query)])
    session, runner = await setup_session_and_runner()
    events = runner.run_async(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    async for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("Agent Response: ", final_response)

# Note: In Colab, you can directly use 'await' at the top level.
# If running this code as a standalone Python script, you'll need to use asyncio.run() or manage the event loop.
await call_agent_async("what's the latest ai news?")
```

```go
// Copyright 2025 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

package main

import (
    "context"
    "fmt"
    "log"

    "google.golang.org/adk/agent"
    "google.golang.org/adk/agent/llmagent"
    "google.golang.org/adk/model/gemini"
    "google.golang.org/adk/runner"
    "google.golang.org/adk/session"
    "google.golang.org/adk/tool"
    "google.golang.org/adk/tool/geminitool"
    "google.golang.org/genai"
)

func createSearchAgent(ctx context.Context) (agent.Agent, error) {
    model, err := gemini.NewModel(ctx, "gemini-2.5-flash", &genai.ClientConfig{})
    if err != nil {
        return nil, fmt.Errorf("failed to create model: %v", err)
    }

    return llmagent.New(llmagent.Config{
        Name:        "basic_search_agent",
        Model:       model,
        Description: "Agent to answer questions using Google Search.",
        Instruction: "I can answer your questions by searching the web. Just ask me anything!",
        Tools:       []tool.Tool{geminitool.GoogleSearch{}},
    })
}

const (
    userID  = "user1234"
    appName = "Google Search_agent"
)

func callAgent(ctx context.Context, a agent.Agent, prompt string) error {
    sessionService := session.InMemoryService()
    session, err := sessionService.Create(ctx, &session.CreateRequest{
        AppName: appName,
        UserID:  userID,
    })
    if err != nil {
        return fmt.Errorf("failed to create the session service: %v", err)
    }

    config := runner.Config{
        AppName:        appName,
        Agent:          a,
        SessionService: sessionService,
    }
    r, err := runner.New(config)
    if err != nil {
        return fmt.Errorf("failed to create the runner: %v", err)
    }

    sessionID := session.Session.ID()
    userMsg := &genai.Content{
        Parts: []*genai.Part{{Text: prompt}},
        Role:  string(genai.RoleUser),
    }

    // The r.Run method streams events and errors.
    // The loop iterates over the results, handling them as they arrive.
    for event, err := range r.Run(ctx, userID, sessionID, userMsg, agent.RunConfig{
        StreamingMode: agent.StreamingModeSSE,
    }) {
        if err != nil {
            fmt.Printf("\nAGENT_ERROR: %v\n", err)
        } else if event.Partial {
            for _, p := range event.LLMResponse.Content.Parts {
                fmt.Print(p.Text)
            }
        }
    }
    return nil
}

func main() {
    agent, err := createSearchAgent(context.Background())
    if err != nil {
        log.Fatalf("Failed to create agent: %v", err)
    }
    fmt.Println("Agent created:", agent.Name())
    prompt := "what's the latest ai news?"
    fmt.Printf("\nPrompt: %s\nResponse: ", prompt)
    if err := callAgent(context.Background(), agent, prompt); err != nil {
        log.Fatalf("Error calling agent: %v", err)
    }
    fmt.Println("\n---")
}
```

```java
import com.google.adk.agents.BaseAgent;
import com.google.adk.agents.LlmAgent;
import com.google.adk.runner.Runner;
import com.google.adk.sessions.InMemorySessionService;
import com.google.adk.sessions.Session;
import com.google.adk.tools.GoogleSearchTool;
import com.google.common.collect.ImmutableList;
import com.google.genai.types.Content;
import com.google.genai.types.Part;

public class GoogleSearchAgentApp {

  private static final String APP_NAME = "Google Search_agent";
  private static final String USER_ID = "user1234";
  private static final String SESSION_ID = "1234";

  /**
   * Calls the agent with the given query and prints the final response.
   *
   * @param runner The runner to use.
   * @param query The query to send to the agent.
   */
  public static void callAgent(Runner runner, String query) {
    Content content =
        Content.fromParts(Part.fromText(query));

    InMemorySessionService sessionService = (InMemorySessionService) runner.sessionService();
    Session session =
        sessionService
            .createSession(APP_NAME, USER_ID, /* state= */ null, SESSION_ID)
            .blockingGet();

    runner
        .runAsync(session.userId(), session.id(), content)
        .forEach(
            event -> {
              if (event.finalResponse()
                  && event.content().isPresent()
                  && event.content().get().parts().isPresent()
                  && !event.content().get().parts().get().isEmpty()
                  && event.content().get().parts().get().get(0).text().isPresent()) {
                String finalResponse = event.content().get().parts().get().get(0).text().get();
                System.out.println("Agent Response: " + finalResponse);
              }
            });
  }

  public static void main(String[] args) {
    // Google Search is a pre-built tool which allows the agent to perform Google searches.
    GoogleSearchTool googleSearchTool = new GoogleSearchTool();

    BaseAgent rootAgent =
        LlmAgent.builder()
            .name("basic_search_agent")
            .model("gemini-2.0-flash") // Ensure to use a Gemini 2.0 model for Google Search Tool
            .description("Agent to answer questions using Google Search.")
            .instruction(
                "I can answer your questions by searching the internet. Just ask me anything!")
            .tools(ImmutableList.of(googleSearchTool))
            .build();

    // Session and Runner
    InMemorySessionService sessionService = new InMemorySessionService();
    Runner runner = new Runner(rootAgent, APP_NAME, null, sessionService);

    // Agent Interaction
    callAgent(runner, "what's the latest ai news?");
  }
}
```

### Code Execution

Supported in ADKPython v0.1.0Java v0.2.0

The `built_in_code_execution` tool enables the agent to execute code, specifically when using Gemini 2 models. This allows the model to perform tasks like calculations, data manipulation, or running small scripts.

````py
# Copyright 2025 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import asyncio
from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.code_executors import BuiltInCodeExecutor
from google.genai import types

AGENT_NAME = "calculator_agent"
APP_NAME = "calculator"
USER_ID = "user1234"
SESSION_ID = "session_code_exec_async"
GEMINI_MODEL = "gemini-2.0-flash"

# Agent Definition
code_agent = LlmAgent(
    name=AGENT_NAME,
    model=GEMINI_MODEL,
    code_executor=BuiltInCodeExecutor(),
    instruction="""You are a calculator agent.
    When given a mathematical expression, write and execute Python code to calculate the result.
    Return only the final numerical result as plain text, without markdown or code blocks.
    """,
    description="Executes Python code to perform calculations.",
)

# Session and Runner
session_service = InMemorySessionService()
session = asyncio.run(session_service.create_session(
    app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
))
runner = Runner(agent=code_agent, app_name=APP_NAME,
                session_service=session_service)

# Agent Interaction (Async)
async def call_agent_async(query):
    content = types.Content(role="user", parts=[types.Part(text=query)])
    print(f"\n--- Running Query: {query} ---")
    final_response_text = "No final text response captured."
    try:
        # Use run_async
        async for event in runner.run_async(
            user_id=USER_ID, session_id=SESSION_ID, new_message=content
        ):
            print(f"Event ID: {event.id}, Author: {event.author}")

            # --- Check for specific parts FIRST ---
            has_specific_part = False
            if event.content and event.content.parts:
                for part in event.content.parts:  # Iterate through all parts
                    if part.executable_code:
                        # Access the actual code string via .code
                        print(
                            f"  Debug: Agent generated code:\n```python\n{part.executable_code.code}\n```"
                        )
                        has_specific_part = True
                    elif part.code_execution_result:
                        # Access outcome and output correctly
                        print(
                            f"  Debug: Code Execution Result: {part.code_execution_result.outcome} - Output:\n{part.code_execution_result.output}"
                        )
                        has_specific_part = True
                    # Also print any text parts found in any event for debugging
                    elif part.text and not part.text.isspace():
                        print(f"  Text: '{part.text.strip()}'")
                        # Do not set has_specific_part=True here, as we want the final response logic below

            # --- Check for final response AFTER specific parts ---
            # Only consider it final if it doesn't have the specific code parts we just handled
            if not has_specific_part and event.is_final_response():
                if (
                    event.content
                    and event.content.parts
                    and event.content.parts[0].text
                ):
                    final_response_text = event.content.parts[0].text.strip()
                    print(f"==> Final Agent Response: {final_response_text}")
                else:
                    print(
                        "==> Final Agent Response: [No text content in final event]")

    except Exception as e:
        print(f"ERROR during agent run: {e}")
    print("-" * 30)


# Main async function to run the examples
async def main():
    await call_agent_async("Calculate the value of (5 + 7) * 3")
    await call_agent_async("What is 10 factorial?")


# Execute the main async function
try:
    asyncio.run(main())
except RuntimeError as e:
    # Handle specific error when running asyncio.run in an already running loop (like Jupyter/Colab)
    if "cannot be called from a running event loop" in str(e):
        print("\nRunning in an existing event loop (like Colab/Jupyter).")
        print("Please run `await main()` in a notebook cell instead.")
        # If in an interactive environment like a notebook, you might need to run:
        # await main()
    else:
        raise e  # Re-raise other runtime errors
````

````java
import com.google.adk.agents.BaseAgent;
import com.google.adk.agents.LlmAgent;
import com.google.adk.runner.Runner;
import com.google.adk.sessions.InMemorySessionService;
import com.google.adk.sessions.Session;
import com.google.adk.tools.BuiltInCodeExecutionTool;
import com.google.common.collect.ImmutableList;
import com.google.genai.types.Content;
import com.google.genai.types.Part;

public class CodeExecutionAgentApp {

  private static final String AGENT_NAME = "calculator_agent";
  private static final String APP_NAME = "calculator";
  private static final String USER_ID = "user1234";
  private static final String SESSION_ID = "session_code_exec_sync";
  private static final String GEMINI_MODEL = "gemini-2.0-flash";

  /**
   * Calls the agent with a query and prints the interaction events and final response.
   *
   * @param runner The runner instance for the agent.
   * @param query The query to send to the agent.
   */
  public static void callAgent(Runner runner, String query) {
    Content content =
        Content.builder().role("user").parts(ImmutableList.of(Part.fromText(query))).build();

    InMemorySessionService sessionService = (InMemorySessionService) runner.sessionService();
    Session session =
        sessionService
            .createSession(APP_NAME, USER_ID, /* state= */ null, SESSION_ID)
            .blockingGet();

    System.out.println("\n--- Running Query: " + query + " ---");
    final String[] finalResponseText = {"No final text response captured."};

    try {
      runner
          .runAsync(session.userId(), session.id(), content)
          .forEach(
              event -> {
                System.out.println("Event ID: " + event.id() + ", Author: " + event.author());

                boolean hasSpecificPart = false;
                if (event.content().isPresent() && event.content().get().parts().isPresent()) {
                  for (Part part : event.content().get().parts().get()) {
                    if (part.executableCode().isPresent()) {
                      System.out.println(
                          "  Debug: Agent generated code:\n```python\n"
                              + part.executableCode().get().code()
                              + "\n```");
                      hasSpecificPart = true;
                    } else if (part.codeExecutionResult().isPresent()) {
                      System.out.println(
                          "  Debug: Code Execution Result: "
                              + part.codeExecutionResult().get().outcome()
                              + " - Output:\n"
                              + part.codeExecutionResult().get().output());
                      hasSpecificPart = true;
                    } else if (part.text().isPresent() && !part.text().get().trim().isEmpty()) {
                      System.out.println("  Text: '" + part.text().get().trim() + "'");
                    }
                  }
                }

                if (!hasSpecificPart && event.finalResponse()) {
                  if (event.content().isPresent()
                      && event.content().get().parts().isPresent()
                      && !event.content().get().parts().get().isEmpty()
                      && event.content().get().parts().get().get(0).text().isPresent()) {
                    finalResponseText[0] =
                        event.content().get().parts().get().get(0).text().get().trim();
                    System.out.println("==> Final Agent Response: " + finalResponseText[0]);
                  } else {
                    System.out.println(
                        "==> Final Agent Response: [No text content in final event]");
                  }
                }
              });
    } catch (Exception e) {
      System.err.println("ERROR during agent run: " + e.getMessage());
      e.printStackTrace();
    }
    System.out.println("------------------------------");
  }

  public static void main(String[] args) {
    BuiltInCodeExecutionTool codeExecutionTool = new BuiltInCodeExecutionTool();

    BaseAgent codeAgent =
        LlmAgent.builder()
            .name(AGENT_NAME)
            .model(GEMINI_MODEL)
            .tools(ImmutableList.of(codeExecutionTool))
            .instruction(
                """
                                You are a calculator agent.
                                When given a mathematical expression, write and execute Python code to calculate the result.
                                Return only the final numerical result as plain text, without markdown or code blocks.
                                """)
            .description("Executes Python code to perform calculations.")
            .build();

    InMemorySessionService sessionService = new InMemorySessionService();
    Runner runner = new Runner(codeAgent, APP_NAME, null, sessionService);

    callAgent(runner, "Calculate the value of (5 + 7) * 3");
    callAgent(runner, "What is 10 factorial?");
  }
}
````

### GKE Code Executor

Supported in ADKPython v1.14.0

The GKE Code Executor (`GkeCodeExecutor`) provides a secure and scalable method for running LLM-generated code by leveraging the GKE (Google Kubernetes Engine) Sandbox environment, which uses gVisor for workload isolation. For each code execution request, it dynamically creates an ephemeral, sandboxed Kubernetes Job with a hardened Pod configuration. You should use this executor for production environments on GKE where security and isolation are critical.

#### How it Works

When a request to execute code is made, the `GkeCodeExecutor` performs the following steps:

1. **Creates a ConfigMap:** A Kubernetes ConfigMap is created to store the Python code that needs to be executed.
1. **Creates a Sandboxed Pod:** A new Kubernetes Job is created, which in turn creates a Pod with a hardened security context and the gVisor runtime enabled. The code from the ConfigMap is mounted into this Pod.
1. **Executes the Code:** The code is executed within the sandboxed Pod, isolated from the underlying node and other workloads.
1. **Retrieves the Result:** The standard output and error streams from the execution are captured from the Pod's logs.
1. **Cleans Up Resources:** Once the execution is complete, the Job and the associated ConfigMap are automatically deleted, ensuring that no artifacts are left behind.

#### Key Benefits

- **Enhanced Security:** Code is executed in a gVisor-sandboxed environment with kernel-level isolation.
- **Ephemeral Environments:** Each code execution runs in its own ephemeral Pod, to prevent state transfer between executions.
- **Resource Control:** You can configure CPU and memory limits for the execution Pods to prevent resource abuse.
- **Scalability:** Allows you to run a large number of code executions in parallel, with GKE handling the scheduling and scaling of the underlying nodes.

#### System requirements

The following requirements must be met to successfully deploy your ADK project with the GKE Code Executor tool:

- GKE cluster with a **gVisor-enabled node pool**.
- Agent's service account requires specific **RBAC permissions**, which allow it to:
  - Create, watch, and delete **Jobs** for each execution request.
  - Manage **ConfigMaps** to inject code into the Job's pod.
  - List **Pods** and read their **logs** to retrieve the execution result
- Install the client library with GKE extras: `pip install google-adk[gke]`

For a complete, ready-to-use configuration, see the [deployment_rbac.yaml](https://github.com/google/adk-python/blob/main/contributing/samples/gke_agent_sandbox/deployment_rbac.yaml) sample. For more information on deploying ADK workflows to GKE, see [Deploy to Google Kubernetes Engine (GKE)](/adk-docs/deploy/gke/).

```python
from google.adk.agents import LlmAgent
from google.adk.code_executors import GkeCodeExecutor

# Initialize the executor, targeting the namespace where its ServiceAccount
# has the required RBAC permissions.
# This example also sets a custom timeout and resource limits.
gke_executor = GkeCodeExecutor(
    namespace="agent-sandbox",
    timeout_seconds=600,
    cpu_limit="1000m",  # 1 CPU core
    mem_limit="1Gi",
)

# The agent now uses this executor for any code it generates.
gke_agent = LlmAgent(
    name="gke_coding_agent",
    model="gemini-2.0-flash",
    instruction="You are a helpful AI agent that writes and executes Python code.",
    code_executor=gke_executor,
)
```

#### Configuration parameters

The `GkeCodeExecutor` can be configured with the following parameters:

| Parameter            | Type  | Description                                                                                                           |
| -------------------- | ----- | --------------------------------------------------------------------------------------------------------------------- |
| `namespace`          | `str` | Kubernetes namespace where the execution Jobs will be created. Defaults to `"default"`.                               |
| `image`              | `str` | Container image to use for the execution Pod. Defaults to `"python:3.11-slim"`.                                       |
| `timeout_seconds`    | `int` | Timeout in seconds for the code execution. Defaults to `300`.                                                         |
| `cpu_requested`      | `str` | Amount of CPU to request for the execution Pod. Defaults to `"200m"`.                                                 |
| `mem_requested`      | `str` | Amount of memory to request for the execution Pod. Defaults to `"256Mi"`.                                             |
| `cpu_limit`          | `str` | Maximum amount of CPU the execution Pod can use. Defaults to `"500m"`.                                                |
| `mem_limit`          | `str` | Maximum amount of memory the execution Pod can use. Defaults to `"512Mi"`.                                            |
| `kubeconfig_path`    | `str` | Path to a kubeconfig file to use for authentication. Falls back to in-cluster config or the default local kubeconfig. |
| `kubeconfig_context` | `str` | The `kubeconfig` context to use.                                                                                      |

### Vertex AI RAG Engine

Supported in ADKPython v0.1.0Java v0.2.0

The `vertex_ai_rag_retrieval` tool allows the agent to perform private data retrieval using Vertex AI RAG Engine.

When you use grounding with Vertex AI RAG Engine, you need to prepare a RAG corpus before hand. Please refer to the [RAG ADK agent sample](https://github.com/google/adk-samples/blob/main/python/agents/RAG/rag/shared_libraries/prepare_corpus_and_data.py) or [Vertex AI RAG Engine page](https://cloud.google.com/vertex-ai/generative-ai/docs/rag-engine/rag-quickstart) for setting it up.

```py
# Copyright 2025 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import os

from google.adk.agents import Agent
from google.adk.tools.retrieval.vertex_ai_rag_retrieval import VertexAiRagRetrieval
from vertexai.preview import rag

from dotenv import load_dotenv
from .prompts import return_instructions_root

load_dotenv()

ask_vertex_retrieval = VertexAiRagRetrieval(
    name='retrieve_rag_documentation',
    description=(
        'Use this tool to retrieve documentation and reference materials for the question from the RAG corpus,'
    ),
    rag_resources=[
        rag.RagResource(
            # please fill in your own rag corpus
            # here is a sample rag corpus for testing purpose
            # e.g. projects/123/locations/us-central1/ragCorpora/456
            rag_corpus=os.environ.get("RAG_CORPUS")
        )
    ],
    similarity_top_k=10,
    vector_distance_threshold=0.6,
)

root_agent = Agent(
    model='gemini-2.0-flash-001',
    name='ask_rag_agent',
    instruction=return_instructions_root(),
    tools=[
        ask_vertex_retrieval,
    ]
)
```

### Vertex AI Search

Supported in ADKPython v0.1.0

The `vertex_ai_search_tool` uses Google Cloud Vertex AI Search, enabling the agent to search across your private, configured data stores (e.g., internal documents, company policies, knowledge bases). This built-in tool requires you to provide the specific data store ID during configuration. For further details of the tool, see [Understanding Vertex AI Search grounding](https://google.github.io/adk-docs/grounding/vertex_ai_search_grounding/index.md).

```py
# Copyright 2024 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import asyncio

from google.adk.agents import LlmAgent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.genai import types
from google.adk.tools import VertexAiSearchTool

# Replace with your Vertex AI Search Datastore ID, and respective region (e.g. us-central1 or global).
# Format: projects/<PROJECT_ID>/locations/<REGION>/collections/default_collection/dataStores/<DATASTORE_ID>
DATASTORE_PATH = "DATASTORE_PATH_HERE"

# Constants
APP_NAME_VSEARCH = "vertex_search_app"
USER_ID_VSEARCH = "user_vsearch_1"
SESSION_ID_VSEARCH = "session_vsearch_1"
AGENT_NAME_VSEARCH = "doc_qa_agent"
GEMINI_2_FLASH = "gemini-2.0-flash"

# Tool Instantiation
# You MUST provide your datastore ID here.
vertex_search_tool = VertexAiSearchTool(data_store_id=DATASTORE_PATH)

# Agent Definition
doc_qa_agent = LlmAgent(
    name=AGENT_NAME_VSEARCH,
    model=GEMINI_2_FLASH, # Requires Gemini model
    tools=[vertex_search_tool],
    instruction=f"""You are a helpful assistant that answers questions based on information found in the document store: {DATASTORE_PATH}.
    Use the search tool to find relevant information before answering.
    If the answer isn't in the documents, say that you couldn't find the information.
    """,
    description="Answers questions using a specific Vertex AI Search datastore.",
)

# Session and Runner Setup
session_service_vsearch = InMemorySessionService()
runner_vsearch = Runner(
    agent=doc_qa_agent, app_name=APP_NAME_VSEARCH, session_service=session_service_vsearch
)
session_vsearch = session_service_vsearch.create_session(
    app_name=APP_NAME_VSEARCH, user_id=USER_ID_VSEARCH, session_id=SESSION_ID_VSEARCH
)

# Agent Interaction Function
async def call_vsearch_agent_async(query):
    print("\n--- Running Vertex AI Search Agent ---")
    print(f"Query: {query}")
    if "DATASTORE_PATH_HERE" in DATASTORE_PATH:
        print("Skipping execution: Please replace DATASTORE_PATH_HERE with your actual datastore ID.")
        print("-" * 30)
        return

    content = types.Content(role='user', parts=[types.Part(text=query)])
    final_response_text = "No response received."
    try:
        async for event in runner_vsearch.run_async(
            user_id=USER_ID_VSEARCH, session_id=SESSION_ID_VSEARCH, new_message=content
        ):
            # Like Google Search, results are often embedded in the model's response.
            if event.is_final_response() and event.content and event.content.parts:
                final_response_text = event.content.parts[0].text.strip()
                print(f"Agent Response: {final_response_text}")
                # You can inspect event.grounding_metadata for source citations
                if event.grounding_metadata:
                    print(f"  (Grounding metadata found with {len(event.grounding_metadata.grounding_attributions)} attributions)")

    except Exception as e:
        print(f"An error occurred: {e}")
        print("Ensure your datastore ID is correct and the service account has permissions.")
    print("-" * 30)

# --- Run Example ---
async def run_vsearch_example():
    # Replace with a question relevant to YOUR datastore content
    await call_vsearch_agent_async("Summarize the main points about the Q2 strategy document.")
    await call_vsearch_agent_async("What safety procedures are mentioned for lab X?")

# Execute the example
# await run_vsearch_example()

# Running locally due to potential colab asyncio issues with multiple awaits
try:
    asyncio.run(run_vsearch_example())
except RuntimeError as e:
    if "cannot be called from a running event loop" in str(e):
        print("Skipping execution in running event loop (like Colab/Jupyter). Run locally.")
    else:
        raise e
```

### BigQuery

Supported in ADKPython v1.1.0

These are a set of tools aimed to provide integration with BigQuery, namely:

- **`list_dataset_ids`**: Fetches BigQuery dataset ids present in a GCP project.
- **`get_dataset_info`**: Fetches metadata about a BigQuery dataset.
- **`list_table_ids`**: Fetches table ids present in a BigQuery dataset.
- **`get_table_info`**: Fetches metadata about a BigQuery table.
- **`execute_sql`**: Runs a SQL query in BigQuery and fetch the result.
- **`forecast`**: Runs a BigQuery AI time series forecast using the `AI.FORECAST` function.
- **`ask_data_insights`**: Answers questions about data in BigQuery tables using natural language.

They are packaged in the toolset `BigQueryToolset`.

```py
# Copyright 2025 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import asyncio

from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools.bigquery import BigQueryCredentialsConfig
from google.adk.tools.bigquery import BigQueryToolset
from google.adk.tools.bigquery.config import BigQueryToolConfig
from google.adk.tools.bigquery.config import WriteMode
from google.genai import types
import google.auth

# Define constants for this example agent
AGENT_NAME = "bigquery_agent"
APP_NAME = "bigquery_app"
USER_ID = "user1234"
SESSION_ID = "1234"
GEMINI_MODEL = "gemini-2.0-flash"

# Define a tool configuration to block any write operations
tool_config = BigQueryToolConfig(write_mode=WriteMode.BLOCKED)

# Uses externally-managed Application Default Credentials (ADC) by default.
# This decouples authentication from the agent / tool lifecycle.
# https://cloud.google.com/docs/authentication/provide-credentials-adc
credentials_config = BigQueryCredentialsConfig()

# Instantiate a BigQuery toolset
bigquery_toolset = BigQueryToolset(
    credentials_config=credentials_config, bigquery_tool_config=tool_config
)

# Agent Definition
bigquery_agent = Agent(
    model=GEMINI_MODEL,
    name=AGENT_NAME,
    description=(
        "Agent to answer questions about BigQuery data and models and execute"
        " SQL queries."
    ),
    instruction="""\
        You are a data science agent with access to several BigQuery tools.
        Make use of those tools to answer the user's questions.
    """,
    tools=[bigquery_toolset],
)

# Session and Runner
session_service = InMemorySessionService()
session = asyncio.run(
    session_service.create_session(
        app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
    )
)
runner = Runner(
    agent=bigquery_agent, app_name=APP_NAME, session_service=session_service
)


# Agent Interaction
def call_agent(query):
    """
    Helper function to call the agent with a query.
    """
    content = types.Content(role="user", parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    print("USER:", query)
    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("AGENT:", final_response)


call_agent("Are there any ml datasets in bigquery-public-data project?")
call_agent("Tell me more about ml_datasets.")
call_agent("Which all tables does it have?")
call_agent("Tell me more about the census_adult_income table.")
call_agent("How many rows are there per income bracket?")
call_agent(
    "What is the statistical correlation between education_num, age, and the income_bracket?"
)
```

### Spanner

Supported in ADKPython v1.11.0

These are a set of tools aimed to provide integration with Spanner, namely:

- **`list_table_names`**: Fetches table names present in a GCP Spanner database.
- **`list_table_indexes`**: Fetches table indexes present in a GCP Spanner database.
- **`list_table_index_columns`**: Fetches table index columns present in a GCP Spanner database.
- **`list_named_schemas`**: Fetches named schema for a Spanner database.
- **`get_table_schema`**: Fetches Spanner database table schema and metadata information.
- **`execute_sql`**: Runs a SQL query in Spanner database and fetch the result.
- **`similarity_search`**: Similarity search in Spanner using a text query.

They are packaged in the toolset `SpannerToolset`.

```py
# Copyright 2025 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import asyncio

from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
# from google.adk.sessions import DatabaseSessionService
from google.adk.tools.google_tool import GoogleTool
from google.adk.tools.spanner import query_tool
from google.adk.tools.spanner.settings import SpannerToolSettings
from google.adk.tools.spanner.settings import Capabilities
from google.adk.tools.spanner.spanner_credentials import SpannerCredentialsConfig
from google.adk.tools.spanner.spanner_toolset import SpannerToolset
from google.genai import types
from google.adk.tools.tool_context import ToolContext
import google.auth
from google.auth.credentials import Credentials

# Define constants for this example agent
AGENT_NAME = "spanner_agent"
APP_NAME = "spanner_app"
USER_ID = "user1234"
SESSION_ID = "1234"
GEMINI_MODEL = "gemini-2.5-flash"

# Define Spanner tool config with read capability set to allowed.
tool_settings = SpannerToolSettings(capabilities=[Capabilities.DATA_READ])

# Define a credentials config - in this example we are using application default
# credentials
# https://cloud.google.com/docs/authentication/provide-credentials-adc
application_default_credentials, _ = google.auth.default()
credentials_config = SpannerCredentialsConfig(
    credentials=application_default_credentials
)

# Instantiate a Spanner toolset
spanner_toolset = SpannerToolset(
    credentials_config=credentials_config, spanner_tool_settings=tool_settings
)

# Optional
# Create a wrapped function tool for the agent on top of the built-in
# `execute_sql` tool in the Spanner toolset.
# For example, this customized tool can perform a dynamically-built query.
def count_rows_tool(
    table_name: str,
    credentials: Credentials,  # GoogleTool handles `credentials`
    settings: SpannerToolSettings,  # GoogleTool handles `settings`
    tool_context: ToolContext,  # GoogleTool handles `tool_context`
):
  """Counts the total number of rows for a specified table.

  Args:
    table_name: The name of the table for which to count rows.

  Returns:
      The total number of rows in the table.
  """

  # Replace the following settings for a specific Spanner database.
  PROJECT_ID = "<PROJECT_ID>"
  INSTANCE_ID = "<INSTANCE_ID>"
  DATABASE_ID = "<DATABASE_ID>"

  query = f"""
  SELECT count(*) FROM {table_name}
    """

  return query_tool.execute_sql(
      project_id=PROJECT_ID,
      instance_id=INSTANCE_ID,
      database_id=DATABASE_ID,
      query=query,
      credentials=credentials,
      settings=settings,
      tool_context=tool_context,
  )

# Agent Definition
spanner_agent = Agent(
    model=GEMINI_MODEL,
    name=AGENT_NAME,
    description=(
        "Agent to answer questions about Spanner database and execute SQL queries."
    ),
    instruction="""\
        You are a data assistant agent with access to several Spanner tools.
        Make use of those tools to answer the user's questions.
    """,
    tools=[
        spanner_toolset,
        # Add customized Spanner tool based on the built-in Spanner toolset.
        GoogleTool(
            func=count_rows_tool,
            credentials_config=credentials_config,
            tool_settings=tool_settings,
        ),
    ],
)


# Session and Runner
session_service = InMemorySessionService()

# Optionally, Spanner can be used as the Database Session Service for production.
# Note that it's suggested to use a dedicated instance/database for storing sessions.
# session_service_spanner_db_url = "spanner+spanner:///projects/PROJECT_ID/instances/INSTANCE_ID/databases/my-adk-session"
# session_service = DatabaseSessionService(db_url=session_service_spanner_db_url)

session = asyncio.run(
    session_service.create_session(
        app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
    )
)
runner = Runner(
    agent=spanner_agent, app_name=APP_NAME, session_service=session_service
)


# Agent Interaction
def call_agent(query):
    """
    Helper function to call the agent with a query.
    """
    content = types.Content(role="user", parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    print("USER:", query)
    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("AGENT:", final_response)

# Replace the Spanner database and table names below with your own.
call_agent("List all tables in projects/<PROJECT_ID>/instances/<INSTANCE_ID>/databases/<DATABASE_ID>")
call_agent("Describe the schema of <TABLE_NAME>")
call_agent("List the top 5 rows in <TABLE_NAME>")
```

### Bigtable

Supported in ADKPython v1.12.0

These are a set of tools aimed to provide integration with Bigtable, namely:

- **`list_instances`**: Fetches Bigtable instances in a Google Cloud project.
- **`get_instance_info`**: Fetches metadata instance information in a Google Cloud project.
- **`list_tables`**: Fetches tables in a GCP Bigtable instance.
- **`get_table_info`**: Fetches metadata table information in a GCP Bigtable.
- **`execute_sql`**: Runs a SQL query in Bigtable table and fetch the result.

They are packaged in the toolset `BigtableToolset`.

```py
# Copyright 2025 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import asyncio

from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.tools.google_tool import GoogleTool
from google.adk.tools.bigtable import query_tool
from google.adk.tools.bigtable.settings import BigtableToolSettings
from google.adk.tools.bigtable.bigtable_credentials import BigtableCredentialsConfig
from google.adk.tools.bigtable.bigtable_toolset import BigtableToolset
from google.genai import types
from google.adk.tools.tool_context import ToolContext
import google.auth
from google.auth.credentials import Credentials

# Define constants for this example agent
AGENT_NAME = "bigtable_agent"
APP_NAME = "bigtable_app"
USER_ID = "user1234"
SESSION_ID = "1234"
GEMINI_MODEL = "gemini-2.5-flash"

# Define Bigtable tool config with read capability set to allowed.
tool_settings = BigtableToolSettings()

# Define a credentials config - in this example we are using application default
# credentials
# https://cloud.google.com/docs/authentication/provide-credentials-adc
application_default_credentials, _ = google.auth.default()
credentials_config = BigtableCredentialsConfig(
    credentials=application_default_credentials
)

# Instantiate a Bigtable toolset
bigtable_toolset = BigtableToolset(
    credentials_config=credentials_config, bigtable_tool_settings=tool_settings
)

# Optional
# Create a wrapped function tool for the agent on top of the built-in
# `execute_sql` tool in the bigtable toolset.
# For example, this customized tool can perform a dynamically-built query.
def count_rows_tool(
    table_name: str,
    credentials: Credentials,  # GoogleTool handles `credentials`
    settings: BigtableToolSettings,  # GoogleTool handles `settings`
    tool_context: ToolContext,  # GoogleTool handles `tool_context`
):
  """Counts the total number of rows for a specified table.

  Args:
    table_name: The name of the table for which to count rows.

  Returns:
      The total number of rows in the table.
  """

  # Replace the following settings for a specific bigtable database.
  PROJECT_ID = "<PROJECT_ID>"
  INSTANCE_ID = "<INSTANCE_ID>"

  query = f"""
  SELECT count(*) FROM {table_name}
    """

  return query_tool.execute_sql(
      project_id=PROJECT_ID,
      instance_id=INSTANCE_ID,
      query=query,
      credentials=credentials,
      settings=settings,
      tool_context=tool_context,
  )

# Agent Definition
bigtable_agent = Agent(
    model=GEMINI_MODEL,
    name=AGENT_NAME,
    description=(
        "Agent to answer questions about bigtable database and execute SQL queries."
    ),
    instruction="""\
        You are a data assistant agent with access to several bigtable tools.
        Make use of those tools to answer the user's questions.
    """,
    tools=[
        bigtable_toolset,
        # Add customized bigtable tool based on the built-in bigtable toolset.
        GoogleTool(
            func=count_rows_tool,
            credentials_config=credentials_config,
            tool_settings=tool_settings,
        ),
    ],
)


# Session and Runner
session_service = InMemorySessionService()

session = asyncio.run(
    session_service.create_session(
        app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
    )
)
runner = Runner(
    agent=bigtable_agent, app_name=APP_NAME, session_service=session_service
)


# Agent Interaction
def call_agent(query):
    """
    Helper function to call the agent with a query.
    """
    content = types.Content(role="user", parts=[types.Part(text=query)])
    events = runner.run(user_id=USER_ID, session_id=SESSION_ID, new_message=content)

    print("USER:", query)
    for event in events:
        if event.is_final_response():
            final_response = event.content.parts[0].text
            print("AGENT:", final_response)

# Replace the bigtable instance and table names below with your own.
call_agent("List all tables in projects/<PROJECT_ID>/instances/<INSTANCE_ID>")
call_agent("List the top 5 rows in <TABLE_NAME>")
```

## Use Built-in tools with other tools

Supported in ADKPythonJava

The following code sample demonstrates how to use multiple built-in tools or how to use built-in tools with other tools by using multiple agents:

```py
from google.adk.tools.agent_tool import AgentTool
from google.adk.agents import Agent
from google.adk.tools import google_search
from google.adk.code_executors import BuiltInCodeExecutor


search_agent = Agent(
    model='gemini-2.0-flash',
    name='SearchAgent',
    instruction="""
    You're a specialist in Google Search
    """,
    tools=[google_search],
)
coding_agent = Agent(
    model='gemini-2.0-flash',
    name='CodeAgent',
    instruction="""
    You're a specialist in Code Execution
    """,
    code_executor=BuiltInCodeExecutor(),
)
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    tools=[AgentTool(agent=search_agent), AgentTool(agent=coding_agent)],
)
```

```java
import com.google.adk.agents.BaseAgent;
import com.google.adk.agents.LlmAgent;
import com.google.adk.tools.AgentTool;
import com.google.adk.tools.BuiltInCodeExecutionTool;
import com.google.adk.tools.GoogleSearchTool;
import com.google.common.collect.ImmutableList;

public class NestedAgentApp {

  private static final String MODEL_ID = "gemini-2.0-flash";

  public static void main(String[] args) {

    // Define the SearchAgent
    LlmAgent searchAgent =
        LlmAgent.builder()
            .model(MODEL_ID)
            .name("SearchAgent")
            .instruction("You're a specialist in Google Search")
            .tools(new GoogleSearchTool()) // Instantiate GoogleSearchTool
            .build();


    // Define the CodingAgent
    LlmAgent codingAgent =
        LlmAgent.builder()
            .model(MODEL_ID)
            .name("CodeAgent")
            .instruction("You're a specialist in Code Execution")
            .tools(new BuiltInCodeExecutionTool()) // Instantiate BuiltInCodeExecutionTool
            .build();

    // Define the RootAgent, which uses AgentTool.create() to wrap SearchAgent and CodingAgent
    BaseAgent rootAgent =
        LlmAgent.builder()
            .name("RootAgent")
            .model(MODEL_ID)
            .description("Root Agent")
            .tools(
                AgentTool.create(searchAgent), // Use create method
                AgentTool.create(codingAgent)   // Use create method
             )
            .build();

    // Note: This sample only demonstrates the agent definitions.
    // To run these agents, you'd need to integrate them with a Runner and SessionService,
    // similar to the previous examples.
    System.out.println("Agents defined successfully:");
    System.out.println("  Root Agent: " + rootAgent.name());
    System.out.println("  Search Agent (nested): " + searchAgent.name());
    System.out.println("  Code Agent (nested): " + codingAgent.name());
  }
}
```

### Limitations

Warning

Currently, for each root agent or single agent, only one built-in tool is supported. No other tools of any type can be used in the same agent.

For example, the following approach that uses ***a built-in tool along with other tools*** within a single agent is **not** currently supported:

```py
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    tools=[custom_function], 
    code_executor=BuiltInCodeExecutor() # <-- not supported when used with tools
)
```

```java
 LlmAgent searchAgent =
        LlmAgent.builder()
            .model(MODEL_ID)
            .name("SearchAgent")
            .instruction("You're a specialist in Google Search")
            .tools(new GoogleSearchTool(), new YourCustomTool()) // <-- not supported
            .build();
```

ADK Python has a built-in workaroud which bypasses this limitation for `GoogleSearchTool` and `VertexAiSearchTool` (use `bypass_multi_tools_limit=True` to enable it), e.g. [sample agent](https://github.com/google/adk-python/tree/main/contributing/samples/built_in_multi_tools).

Warning

Built-in tools cannot be used within a sub-agent, with the exception of `GoogleSearchTool` and `VertexAiSearchTool` in ADK Python because of the workaround mentioned above.

For example, the following approach that uses built-in tools within sub-agents is **not** currently supported:

```py
url_context_agent = Agent(
    model='gemini-2.0-flash',
    name='UrlContextAgent',
    instruction="""
    You're a specialist in URL Context
    """,
    tools=[url_context],
)
coding_agent = Agent(
    model='gemini-2.0-flash',
    name='CodeAgent',
    instruction="""
    You're a specialist in Code Execution
    """,
    code_executor=BuiltInCodeExecutor(),
)
root_agent = Agent(
    name="RootAgent",
    model="gemini-2.0-flash",
    description="Root Agent",
    sub_agents=[
        url_context_agent,
        coding_agent
    ],
)
```

```java
LlmAgent searchAgent =
    LlmAgent.builder()
        .model("gemini-2.0-flash")
        .name("SearchAgent")
        .instruction("You're a specialist in Google Search")
        .tools(new GoogleSearchTool())
        .build();

LlmAgent codingAgent =
    LlmAgent.builder()
        .model("gemini-2.0-flash")
        .name("CodeAgent")
        .instruction("You're a specialist in Code Execution")
        .tools(new BuiltInCodeExecutionTool())
        .build();


LlmAgent rootAgent =
    LlmAgent.builder()
        .name("RootAgent")
        .model("gemini-2.0-flash")
        .description("Root Agent")
        .subAgents(searchAgent, codingAgent) // Not supported, as the sub agents use built in tools.
        .build();
```
