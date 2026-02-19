# QuickStart

This guide provides a basic overview of how to configure and run agents and workflows using the JSON-defined framework.

## Configuration (`appsettings.json`)

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

## Initialization (`Program.cs`)

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

## Quick Start Guide (MK)

Овој водич објаснува како да ги конфигурирате основните сервиси и да стартувате работен тек во проектот.

### 1. Додавање и Конфигурација на Сервиси

За да ги користите `ITemplateEngine`, `ITopicTerminationService` и `IWorkflowPauseService`, прво треба да ги регистрирате во вашиот Dependency Injection (DI) контејнер, што најчесто се прави во `Program.cs`.

**Регистрација во `Program.cs`:**

```csharp
// Во Program.cs, во делот каде што ги конфигурирате сервисите:

// 1. Регистрирај го стандардниот Handlebars template engine.
builder.Services.AddSingleton<ITemplateEngine, HandlebarsTemplateEngine>();

// 2. Регистрирај ги стандардните сервиси за пауза и терминирање на работниот тек.
builder.Services.AddSingleton<IWorkflowPauseService, DefaultWorkflowPauseService>();
builder.Services.AddSingleton<ITopicTerminationService, DefaultTopicTerminationService>();

// ... останати регистрации на сервиси
```

### 2. Како се користи `ITemplateEngine`

`ITemplateEngine` служи за динамично генерирање на содржина (најчесто промпт) од предефиниран темплејт (шаблон).

**Чекор 1: Креирајте темплејт фајл**

Направете нов фајл во `Chat2Report/Prompts/`, на пример `mojot_prompt.md`.

_Содржина на `Chat2Report/Prompts/mojot_prompt.md`_:

```markdown
Ти си асистент кој треба да го анализира следново барање од корисникот: "{{user_query}}".
```

### 3. Објаснување: Како се Стартува Работен Тек

Следниот код покажува како се иницира работен тек.

```csharp
// 1. Го зема IAgentRuntime од DI контејнерот.
var agentRuntime = ServiceProvider.GetService<IAgentRuntime>();

if (agentRuntime != null)
{
    // 2. Користи TopicTerminationService за да креира "сигнал за чекање".
    TaskCompletionSource<Dictionary<string, object>>? taskCompletionSource =
        await TopicTerminationService.GetOrCreateAsync(new TopicId("workflow-completion", workflowInstanceId));

    // 3. Ја креира почетната состојба (initial state) на работниот тек.
    var initialState = new WorkflowState
    {
        WorkflowTopicId = workflowInstanceId,
        Data = new Dictionary<string, object>
        {
            ["user_query"] = userMessage.Text
        }
    };

    // 4. Го стартува работниот тек со објавување порака на "topic" (`validate query`).
    await agentRuntime.PublishMessageAsync(
        initialState,
        new TopicId("validate query", workflowInstanceId),
        null, null, currentResponseCancellation.Token);

    // 5. Чека да заврши работниот тек.
    var result = await taskCompletionSource.Task;

    // 6. Прикажи го финалниот резултат.

}
```
