# **Qiskit MCP Servers** 

## _Comprehensive Documentation_ 

Repository: https://github.com/Qiskit/mcp-servers 

## **1. Introduction** 

The Qiskit MCP Servers repository is a collection of Model Context Protocol (MCP) servers developed by IBM to expose Qiskit and IBM Quantum capabilities to AI assistants, large language models, and agents. 

These servers bridge the gap between conversational AI and real quantum computing. Instead of merely describing quantum concepts, the AI can invoke these servers to perform actual tasks: build circuits, transpile them, search documentation, run simulations, optimize circuits with AI, and submit jobs to IBM Quantum hardware. 

## **2. What is Qiskit?** 

Qiskit is an open-source software development kit (SDK) created by IBM for quantum computing. It provides tools to build, simulate, transpile, and execute quantum circuits. 

Key capabilities of Qiskit include: 

- Building quantum circuits using qubits, quantum gates, and measurements 

- Simulating circuits on classical computers to verify behavior 

- Transpiling and optimizing circuits for target hardware 

- Submitting jobs to real quantum processors via IBM Quantum cloud services 

## **3. What is the Model Context Protocol (MCP)?** 

MCP is an open standard created by Anthropic that allows AI systems to connect to external tools, data sources, and services. 

It functions like a universal connector. An MCP server exposes tools, resources, and prompts that an AI assistant can call. The AI sends a request to the server, the server performs the actual work, and the result is returned to the AI. 

An MCP server typically exposes: 

- Tools: functions that the AI can invoke 

- Resources: structured data the AI can read 

- Prompts: reusable templates for common tasks 

## **4. Repository Overview** 

The repository is a Python monorepo managed with the uv package manager. It contains five independent MCP servers, each published as a separate PyPI package, plus a meta-package that installs all of them. 

|**Server**|**Purpose**|**Requires IBM Token**|
|---|---|---|
|**Qiskit MCP Server**|Local circuit analysis,<br>transpilation, and format<br>conversion|No|
|**Qiskit Docs MCP Server**<br>|Search and retrieve Qiskit<br>documentation|No|
|**Qiskit IBM Runtme MCP**<br>**Server**|Execute circuits on IBM<br>Quantum hardware|Yes|
|**Qiskit IBM Transpiler MCP**<br>**Server**|AI-powered cloud<br>transpilation|Yes|
|**Qiskit Gym MCP Server**|Reinforcement learning for<br>circuit optimization|No|



## **5. Server Details** 

### **5.1 Qiskit MCP Server** 

Package name: qiskit-mcp-server 

Purpose: Provides local Qiskit capabilities for circuit analysis, transpilation, and format conversion. No authentication required. 

Key tools: 

- transpile_circuit_tool: Transpile a circuit for a target basis gate set or topology 

- analyze_circuit_tool: Analyze circuit structure without transpiling 

- compare_optimization_levels_tool: Compare transpilation results across optimization levels 0-3 

- load_circuit_from_qasm_tool: Load and validate a QASM 2.0 or 3.0 circuit 

- export_circuit_to_qasm_tool: Export a circuit to QASM 2.0 or 3.0 

- convert_qpy_to_qasm3_tool: Convert QPY format to QASM 3.0 

- convert_qasm3_to_qpy_tool: Convert QASM 3.0 to QPY format 

Key resources: 

- qiskit://transpiler/info 

- qiskit://transpiler/basis-gates 

- qiskit://transpiler/topologies 

How to start: 

```
cd /Users/omkandpal/mcp-servers
source /Users/omkandpal/.local/bin/env
```

```
uv run --package qiskit-mcp-server qiskit-mcp-server
```

### **5.2 Qiskit Docs MCP Server** 

Package name: qiskit-docs-mcp-server 

Purpose: Allows AI assistants to search and retrieve live Qiskit documentation, guides, API references, and error codes. 

Key tools: 

- search_docs_tool: Search across Qiskit docs, API, learning, and tutorials 

- get_page_tool: Fetch a full documentation page by URL 

- lookup_error_code_tool: Look up Qiskit/IBM Quantum error codes 

Key resources: 

- qiskit-docs://modules 

- qiskit-docs://addons 

- qiskit-docs://guides 

- qiskit-docs://tutorials 

- qiskit-docs://api-packages 

- qiskit-docs://error-codes 

How to start: 

```
cd /Users/omkandpal/mcp-servers
source /Users/omkandpal/.local/bin/env
uv run --package qiskit-docs-mcp-server qiskit-docs-mcp-server
```

### **5.3 Qiskit IBM Runtime MCP Server** 

Package name: qiskit-ibm-runtime-mcp-server 

Purpose: Connects to IBM Quantum cloud services to list backends, manage accounts, submit jobs, and retrieve results from real quantum hardware. 

Authentication required: Yes, requires an IBM Quantum API token set as QISKIT_IBM_TOKEN. 

Key tools: 

- setup_ibm_quantum_account_tool: Save IBM Quantum credentials 

- list_backends_tool: List available IBM Quantum backends 

- least_busy_backend_tool: Find the least busy backend 

- get_backend_properties_tool / get_backend_calibration_tool: Retrieve backend metadata 

- get_coupling_map_tool / find_optimal_qubit_chains_tool: Hardware topology analysis 

- run_sampler_tool / run_estimator_tool: Execute circuits on IBM hardware 

- list_my_jobs_tool / get_job_status_tool / get_job_results_tool / cancel_job_tool: Job management 

How to start: 

```
cd /Users/omkandpal/mcp-servers
source /Users/omkandpal/.local/bin/env
QISKIT_IBM_TOKEN=<your-token> uv run --package qiskit-ibm-
runtime-mcp-server qiskit-ibm-runtime-mcp-server
```

### **5.4 Qiskit IBM Transpiler MCP Server** 

Package name: qiskit-ibm-transpiler-mcp-server 

Purpose: Uses IBM's cloud-based AI transpiler to optimize and synthesize quantum circuits. 

Authentication required: Yes, requires an IBM Quantum API token set as QISKIT_IBM_TOKEN. 

Key tools: 

- ai_routing_tool: AI-powered routing and SWAP insertion 

- ai_clifford_synthesis_tool: Clifford circuit synthesis 

- ai_linear_function_synthesis_tool: Linear Boolean function synthesis 

- ai_permutation_synthesis_tool: Permutation routing 

- ai_pauli_network_synthesis_tool: Pauli network synthesis 

- hybrid_ai_transpile_tool: End-to-end classical plus AI transpilation 

Key resources: 

- qiskit-ibm-transpiler://info 

- qiskit-ibm-transpiler://synthesis-types 

How to start: 

```
cd /Users/omkandpal/mcp-servers
source /Users/omkandpal/.local/bin/env
```

```
QISKIT_IBM_TOKEN=<your-token> uv run --package qiskit-ibm-
transpiler-mcp-server qiskit-ibm-transpiler-mcp-server
```

### **5.5 Qiskit Gym MCP Server** 

Package name: qiskit-gym-mcp-server 

Purpose: Uses reinforcement learning to train agents that optimize quantum circuits for tasks such as SWAP insertion, CNOT reduction, and Clifford synthesis. 

Authentication required: No. Runs locally on CPU or GPU, but requires heavy dependencies such as PyTorch and Ray. 

Key tools: 

- create_permutation_env_tool / create_linear_function_env_tool / create_clifford_env_tool: Create RL environments 

- start_training_tool / batch_train_environments_tool / get_training_status_tool / wait_for_training_tool / stop_training_tool: Training management 

- synthesize_permutation_tool / synthesize_linear_function_tool / synthesize_clifford_tool: Use trained models 

- save_model_tool / load_model_tool / list_saved_models_tool / delete_model_tool: Model management 

- extract_subtopologies_tool / get_fake_backend_coupling_map_tool / create_coupling_map_tool: Topology utilities 

How to start: 

```
cd /Users/omkandpal/mcp-servers
source /Users/omkandpal/.local/bin/env
uv run --package qiskit-gym-mcp-server qiskit-gym-mcp-server
```

## **6. Installation and Setup** 

Prerequisites: 

- Python 3.10 or later (3.11+ recommended) 

- uv package manager installed 

- Cloned repository at /Users/omkandpal/mcp-servers 

Install uv if not already installed: 

```
curl -LsSf https://astral.sh/uv/install.sh | sh
source /Users/omkandpal/.local/bin/env
```

Run any server on demand using uv: 

```
uv run --package <package-name> <server-command>
```

Meta-package installation from PyPI: 

```
pip install qiskit-mcp-servers
```

## **7. Quick Start Commands** 

|**Server**|**Start Command**|**IBM Token Required**|
|---|---|---|
|**Qiskit**|uv run --package qiskit-mcp-<br>serverqiskit-mcp-server|No|
|**Qiskit Docs**<br>|uv run --package qiskit-docs-<br>mcp-server qiskit-docs-mcp-<br>server|No|
|**Qiskit IBM Runtme**|QISKIT_IBM_TOKEN=<token><br>uv run --package qiskit-ibm-<br>runtime-mcp-server qiskit-<br>ibm-runtime-mcp-server|Yes|
|**Qiskit IBM Transpiler**|QISKIT_IBM_TOKEN=<token><br>uv run --package qiskit-ibm-<br>transpiler-mcp-server qiskit-<br>ibm-transpiler-mcp-server|Yes|
|**Qiskit Gym**|uv run --package qiskit-gym-<br>mcp-server qiskit-gym-mcp-<br>server|No|



## **8. How It Works** 

The AI assistant acts as the client. When a user asks a quantum-related question, the assistant decides which MCP server to call, sends a request, and receives the result. The MCP server runs the actual Python code and returns structured output. 

#### **User** 

↓ 

#### **AI Assistant** 

#### **MCP Server** 

#### **Qiskit / IBM Quantum** 

#### **Result back to AI Assistant** 

#### **Answer to User** 

## **9. Important Notes** 

- These servers are backend-only and communicate over stdio using the MCP protocol. There is no graphical user interface. 

- The IBM Runtime and Transpiler servers require an IBM Quantum account and API token. 

- Running on real IBM Quantum hardware consumes credits or time from the IBM Quantum account. 

- The Qiskit Gym server is resource-intensive because it depends on PyTorch and Ray for reinforcement learning. 

- The repository is currently at alpha maturity; APIs and behavior may change. 

- A previously included server, qiskit-code-assistant-mcp-server, was removed because IBM discontinued the underlying service. 

