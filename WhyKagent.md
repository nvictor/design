The primary advantage of using **kagent** over deploying your own LLM and Model Context Protocol (MCP) servers is that **kagent** provides a complete, Kubernetes-native framework specifically designed to **build, deploy, manage, and observe AI agents** in a cloud-native environment.

It significantly reduces the complexity and effort of stitching together all the necessary components yourself.

---

## 🚀 Key Benefits of Using kagent

| Feature | kagent Approach | Custom Deployment |
| :--- | :--- | :--- |
| **Kubernetes Native** | Built from the ground up to use **Kubernetes custom resources (CRs)** for agents and tools, leveraging native features like scaling and reliability. | Requires significant custom engineering to integrate and manage agents as Kubernetes workloads. |
| **Declarative Agents** | You define agents and their tools in a **simple YAML file** (declarative), and the kagent Controller handles the deployment. | You must write and manage the procedural code and orchestration logic for the agents and LLM/tool communication. |
| **Built-in Tools & Agent** | Comes with an MCP server pre-configured with **tools for Kubernetes, Istio, Helm, Prometheus**, and more, allowing agents to act on your cluster right away. | You must develop, containerize, and deploy every single tool/API wrapper for your agents to use. |
| **LLM Provider Support**| Supports multiple LLM providers out of the box, including **OpenAI, Google Vertex AI, Anthropic, and Ollama**, simplifying configuration via the **ModelConfig** resource. | Integrating new LLMs means writing new API connectors and managing their credentials and configuration manually. |
| **Observability** | Provides **OpenTelemetry tracing** support for monitoring what your agents and tools are doing, which is crucial for debugging non-deterministic AI behavior. | You would need to custom-implement tracing and monitoring across your separate LLM, agent, and MCP server components. |
| **Extensible & Open** | Uses open standards like **MCP** and is built on Microsoft's **AutoGen** framework, making it flexible for adding custom agents and tools. | You are responsible for all architecture and adherence to standards. |

In short, kagent handles the *orchestration and operational boilerplate* of running sophisticated AI agents in Kubernetes, allowing you to focus on the **logic and prompts** that make your agents useful for cloud-native operations and troubleshooting.

Would you like to know more about the **Core Components** of kagent's architecture?