# Agent-workflow: triage
Incident Severity Triage Agent
Agentic workflow use case for an incident triage system

The notebook defines a full agent workflow for an incident-severity triage use case, combines schema, instructions, tools, validation, and retry logic in one system.

The first major piece is the model backend selection in build_model(). It reads the PROVIDER environment variable and switches between AWS Bedrock and Azure OpenAI without changing the rest of the agent logic.

For AWS, it creates a Bedrock runtime client and uses BedrockConverseModel with BedrockProvider. For Azure, it uses OpenAIChatModel with AzureProvider.

The SeverityResult class is the output schema. It is a Pydantic BaseModel with fields like level, summary, and rationale, which forces the LLM to return structured, validated output instead of free-form text.

IncidentDeps is a dependency container. It holds runtime context such as known_systems, which is a dictionary mapping each system name to its owning team. This is passed into tools and instructions through RunContext.
The Agent object is created with three important settings: the model, deps_type=IncidentDeps, output_type=SeverityResult, and retries=2. This means the agent has a typed dependency structure and can retry when needed.

The @agent.instructions decorator defines the system prompt dynamically. The function receives a RunContext[IncidentDeps], so it can use information from dependencies at runtime, such as the number of systems on record.

ctx is the key runtime object in this code. It is a RunContext instance that gives the agent access to dependency objects and runtime state inside tools and instructions.

incident_tools = FunctionToolset[IncidentDeps]() creates a toolset container. This is a collection of tool functions that the agent can call during execution.

The first tool, lookup_system_owner, uses ctx.deps.known_systems to find the owning team for a system. It is a simple lookup tool, but it is important because the system prompt says the agent must never guess the team and must call this tool first.

The second tool, check_recent_incidents, simulates an incident-management lookup. It returns recent incident titles for a system in the last N hours and uses ModelRetry if the system is unknown.

The third tool, page_oncall, simulates paging the on-call engineer. It validates the severity input and raises ModelRetry if the severity is invalid or the system is unknown, which is a classic self-repair pattern.

ModelRetry is not an application error; it is a structured retry signal for the agent framework. The model gets a corrective hint and can try again with better tool arguments or improved reasoning.

The @agent.output_validator function is the second self-repair mechanism. It validates the final structured response after the model produces it, not just the tool call arguments.

In the validator, if the output says P1 but the rationale does not include evidence like “outage” or “down,” the validator raises ModelRetry and asks the model to revise the answer.

This is a very interview-worthy pattern because it shows that the agent can self-correct both during tool usage and after final output generation, which is essential in enterprise-grade systems.

The relevant_tools_only() function implements “tool RAG” or retrieval over tools. Instead of exposing all tools to the model, it filters them based on the incoming user query.

The filtering is simple here: it tokenizes the query and scores each tool description by keyword overlap. In production, this would often be replaced by embedding-based retrieval over tool descriptions. 
incident_tools.filtered(...) returns a scoped toolset that only includes the most relevant tools. This reduces prompt size and makes the model less likely to choose the wrong tool.

The final execution step, agent.run_sync(query, deps=deps, toolsets=[scoped_toolset]), wires everything together: the query goes in, the filtered toolset is available, the model uses tools and retries when needed, and the final result.output is a validated SeverityResult.

ctx = runtime context object passed into tools/instructions
RunContext = wrapper around dependencies and runtime state
IncidentDeps = dependency container holding business data
FunctionToolset = collection of callable tools exposed to the agent
ModelRetry = built-in repair signal that tells the model to retry with better input
output_validator = guardrail for final structured output
