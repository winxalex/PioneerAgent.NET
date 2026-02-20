# 🤖 A Dynamic AI Agent Framework in .NET

## 👋 Introduction

This project was born out of an exploration of building dynamic, AI-powered agentic workflows using .NET, at a time when Microsoft's open-source AI landscape was rapidly evolving with libraries like Semantic Kernel and AutoGen.

## ❓ Why .NET?

While Python dominates the AI landscape, this project takes a different path, leveraging the strengths of C# and the .NET ecosystem. The goal was to combine the robustness of a statically-typed language with the flexibility typically associated with dynamic languages like Python.

Here's how we achieve that:

- 🔌 **Extensibility with Dependency Injection (DI):** We harness the power of .NET's Inversion of Control (IoC) and Dependency Injection to make every component interchangeable. You can easily plug in your own template engine, state transformers, or database services.
- ✨ **Dynamic by Design:** To avoid the rigidity of endless DTOs and models, we mimic Python's dynamism by using `Dictionary<string, object>` and `JsonElement` for agent state management. This allows for flexible, on-the-fly data manipulation.
- 📄 **JSON-Defined Workflows:** Workflows, agents, and their steps are defined declaratively in JSON. This makes it easy to create, modify, and deploy complex agentic systems without recompiling the code.
- 🔀 **Powerful State Transformation:** We use JSONata for querying and transforming the agent's state, enabling complex logic and data manipulation within the workflow itself.

While the initial idea was to use pure C# for state evaluation, the reality is that Large Language Models (LLMs) primarily communicate via JSON. This framework embraces that reality, providing a powerful and flexible way to build LLM-powered applications in .NET.

[![Watch the video](https://img.youtube.com/vi/ufEC0-CeExI/0.jpg)](https://youtu.be/ufEC0-CeExI)

# 🚀 QuickStart

This guide provides a basic overview of how to configure and run agents and workflows using the JSON-defined framework.

## ⚙️ Configuration (`appsettings.json`)

The core of the agent framework is configured through `appsettings.json`. This includes defining LLM providers, workflow steps, and the agents themselves.

### 1. Define LLM Provider

First, define the AI models you want to use in the `Clients` section. Each client has a unique name (e.g., `gemini-pro`) and provider-specific settings.

```json
"AgentsConfiguration": {
  "Clients": {
    "gemini-pro": {
      "Type": "google",
      "ModelId": "gemini-pro",
      "ApiKey": "YOUR_API_KEY_HERE"
    },
    "ollama-kimi-k2.5:cloud": {
        "Log": true,
        "Type": "ollama",
        "ModelId": "kimi-k2.5:cloud",
        "Endpoint": "http://localhost:11434"
    }
  },
  "...": "..."
}
```

### 2. Define a Workflow Step

Steps are the building blocks of an agent. A step is defined in `StepDefinitions` with a name, a description, and an `IOContract` that specifies its inputs and outputs.

```json
"StepDefinitions": {
  "LlmCall": {
    "Description": "Generic step to call any LLM and receive a raw string output (usually JSON).",
    "IOContract": {
      "Inputs": {
        "templateData": "Dictionary<string, object>"
      },
      "Outputs": {
        "result": "string"
      }
    }
  },
  "...": "..."
}
```

### 3. Define a Workflow and Agent

Workflows contain agents, and agents are composed of a series of steps. In the `Workflows` section, you define an agent, its steps, and how data flows between them.

This example shows an agent `QueryValidationAgent` within `DefaultReportFlow`.

```json
"Workflows": {
  "DefaultReportFlow": {
    "Agents": {
      "QueryValidationAgent": {
        "TopicId": "validate query",
        "Description": "Validates the user query for safety and relevance.",
        "Steps": [
          {
            "Type": "LlmCall",
            "Inputs": {
              "templateData.user_query": "user_query"
            },
            "Outputs": {
              "result": {
                "key": "validation_raw_json",
                "scope": "Step"
              }
            },
            "Config": {
              "ClientNames": [ "gemini-pro" ],
              "SystemPrompt": "You are a security analysis expert...",
              "UserPrompt": "file:Prompts/QueryValidationPrompt.md"
            }
          },
          {
            "Type": "DeserializeJson",
            "Inputs": { "source": "validation_raw_json" },
            "Outputs": {
              "deserializedObject": {
                "key": "validation_result",
                "scope": "Agent"
              }
            },
            "Config": { "targetType": "Chat2Report.Models.ValidationResult, Chat2Report" }
          }
        ],
        "Routes": [
          {
            "Condition": "validation_result.is_valid = true",
            "Receivers": [ "NextAgent/default" ]
          },
          {
            "Condition": "validation_result.is_valid != true",
            "Transform": { "Expression": "{'response': validation_result.reason}" }
          }
        ]
      }
    }
  }
}
```

- **`Type`**: Matches a key from `StepDefinitions` (e.g., `LlmCall`).
- **`Inputs`**: Maps data from the workflow state (e.g., the global `user_query`) to the step's input contract.
- **`Outputs`**: Maps the step's output (e.g., `result`) to a variable in the workflow state (`validation_raw_json`).
- **`Config`**: Provides specific settings for this step instance, like which LLM client to use (`gemini-pro`) and the prompt files.
- **`Routes`**: Defines the next agent to call based on the result of the steps.

## 💡 Initialization (`Program.cs`)

The C# code in `Program.cs` loads the configuration and brings the agents to life.

1.  **Add Services**: The application builder is used to add necessary services.
2.  **Register Steps**: Each step defined in JSON must be registered in the service container with a matching key. This links the step name (e.g., "LlmCall") to its C# implementation (e.g., `LlmCallStep`).
3.  **Deploy Agents**: The `DeployAgents` method reads all the workflow configurations and builds the agent network.

```csharp
// In Program.cs

var builder = WebApplication.CreateBuilder(args);

// ... other service configurations

// Create the main agent application builder
AgentsWebBuilder appBuilder = new AgentsWebBuilder(builder);
appBuilder.UseInProcessRuntime();

// 1. Register the default Handlebars template engine.
builder.Services.AddSingleton<ITemplateEngine, HandlebarsTemplateEngine>();

// 2. Register the default services for pausing and terminating the workflow.
builder.Services.AddSingleton<IWorkflowPauseService, DefaultWorkflowPauseService>();
builder.Services.AddSingleton<ITopicTerminationService, DefaultTopicTerminationService>();

// --- Register all workflow step implementations ---
// This tells the framework which C# class to use for a given step type from JSON.
appBuilder.Services.AddKeyedTransient<IBaseWorkflowStep, LlmCallStep>("LlmCall");
appBuilder.Services.AddKeyedTransient<IBaseWorkflowStep, DeserializeJsonStep>("DeserializeJson");
appBuilder.Services.AddKeyedTransient<IBaseWorkflowStep, SqlExecuteStep>("SqlExecute");
// ... register all other steps

// --- Load configuration and deploy agents ---
var allWorkflows = builder.Configuration.GetSection("AgentsConfiguration:Workflows")
    .Get<Dictionary<string, WorkflowDefinition>>() ?? new();

// This dynamically builds all agents and workflows based on your appsettings.json
appBuilder.DeployAgents(allWorkflows);

// ... build and run the application
var app = await appBuilder.BuildAsync();

// ...
app.Run();

```

---

## Quick Start Guide

This guide explains how to configure basic services and start a workflow in the project.

### Explanation: How to Start a Workflow

The following code shows how a workflow is initiated.

```csharp
// 1. Gets IAgentRuntime from the DI container.
var agentRuntime = ServiceProvider.GetService<IAgentRuntime>();

if (agentRuntime != null)
{
    // 2. Uses TopicTerminationService to create a "wait signal".
    TaskCompletionSource<Dictionary<string, object>>? taskCompletionSource =
        await TopicTerminationService.GetOrCreateAsync(new TopicId("workflow-completion", workflowInstanceId));

    // 3. Creates the initial state of the workflow.
    var initialState = new WorkflowState
    {
        WorkflowTopicId = workflowInstanceId,
        Data = new Dictionary<string, object>
        {
            ["user_query"] = userMessage.Text
        }
    };

    // 4. Starts the workflow by publishing a message to the "topic" (`validate query`).
    await agentRuntime.PublishMessageAsync(
        initialState,
        new TopicId("validate query", workflowInstanceId),
        null, null, currentResponseCancellation.Token);

    // 5. Waits for the workflow to finish.
    var result = await taskCompletionSource.Task;

    // 6. Display the final result.

}
```

##### 🏗️ TODO

- MCP support (prompts and tools can be accessed from there with just mcp://url in the config)
- A2UI (read off nasty Blazor)
- Create Angular Node editor
- Agent memory (maybe even separate project)
- Load workflows from files to be reused

##### 🤝 Contributions

You are welcome to contribute!
