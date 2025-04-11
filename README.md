# Core Methodology of the MedRAX Agent

**Objective:** This document details the fundamental methodology behind an AI agent like MedRAX. It explains the step-by-step operational flow, the interaction between components, data handling (including prompts and tool usage), and the core reasoning cycle with enhanced detail. This aims to provide a clear understanding sufficient for conceptual replication, addressing potential needs for methodological depth.

## 1. Foundational Concept: The ReAct Cycle (Reason + Act)

MedRAX operates based on a cyclical **Reason-Act (ReAct)** principle. This means the agent doesn't just take an input and produce an output in one shot. Instead, it iterates through a loop:

1.  **Reason:** Analyze the current situation (user query, conversation history, previous results) and decide on the next step.
2.  **Act:** If necessary, perform an action, typically using a specialized tool to gather more information or perform a specific task.
3.  **Observe:** Incorporate the results of the action back into its understanding of the situation.
4.  **Repeat:** Loop back to Reason with the updated understanding.

This iterative process allows the agent to tackle complex problems that require multiple steps or external information sources, like analyzing a CXR using various diagnostic tools.

## 2. Essential Components and Their Roles

To execute the ReAct cycle for CXR analysis, MedRAX relies on these interconnected components:

1.  **The Core Reasoning Engine (LLM):**
    * A powerful multimodal Large Language Model (e.g., GPT-4o) capable of understanding text, images, and crucially, **using tools**.
    * **Function:** Performs the "Reason" step. Analyzes the situation and decides whether to answer directly or request a tool's execution.
2.  **The Specialized Toolbox:**
    * A collection of pre-trained AI models for specific CXR tasks (Classification, Segmentation, VQA, Grounding, Report Generation, etc.).
    * **Tool Interface (Wrappers):** Each specialized model is wrapped in a software interface that allows the LLM to interact with it as a "tool." This interface *must* define:
        * `name`: A unique identifier (e.g., `"cxr_classifier"`).
        * `description`: A **critical natural language explanation** for the LLM, detailing what the tool does, its inputs, and its outputs (e.g., "Takes a CXR image reference and returns likely pathologies and their probabilities."). This description acts as a form of **implicit prompting**, guiding the LLM on when to use the tool.
        * `input_schema`: A structured definition of the required inputs (e.g., image identifier).
        * `execution_logic`: Code to run the tool with given inputs and return the result.
    * **Clarification on Tool Wrappers:** The `description` field within the wrapper is paramount. It's the primary way the LLM understands *what the tool does* and *when to use it*. The `input_schema` is equally vital, ensuring the LLM knows precisely what information to provide when requesting the tool.
3.  **The Workflow Orchestrator:**
    * Manages the overall ReAct cycle. It directs the flow of information between the LLM, the Toolbox, and the Memory.
    * **Functions:**
        * Calls the LLM for the "Reason" step.
        * Parses the LLM's response to check for tool requests.
        * Calls the appropriate tools from the Toolbox for the "Act" step.
        * Updates the Memory with LLM responses and tool results.
        * Decides when the cycle is complete.
4.  **The Agent's Memory (State):**
    * Stores the complete history of the interaction as a sequence of structured messages (User input, LLM thoughts/responses, Tool results).
    * **Function:** Provides the necessary context for the LLM's reasoning at each step. It ensures the agent "remembers" previous findings and interactions.

## 3. The Operational Flow: A Detailed Walkthrough

Here’s the refined step-by-step process of how MedRAX processes a query:

**Step 0: Initialization & Observation**
* The user provides a query (e.g., "What abnormalities are present in this CXR?") along with image data.
* The Orchestrator initializes the Memory, storing the user's query and image reference(s) as the first message(s).
* A predefined **System Prompt** is loaded (e.g., "You are MedRAX, an AI assistant specializing in CXR analysis. Analyze the user query and image. Reason step-by-step. Use the available tools precisely when needed to gather specific information like classifications, segmentations, or answers to visual questions. Synthesize findings into a clear response.").

**Step 1: Reasoning (LLM Call)**
* **Prompt Construction:** The Orchestrator prepares the input for the LLM. This is **critical** and typically includes:
    1.  The **System Prompt**.
    2.  The entire **Message History** from Memory, formatted chronologically.
    3.  **Tool Definitions:** The `name`, `description`, and `input_schema` for *all* available tools in the Toolbox. This information is formatted in a specific way the LLM is trained to understand (often dictated by the LLM provider's API – could be a JSON list, XML tags, or a dedicated prompt section). *This tells the LLM what tools it *can* call.*
* **LLM Invocation:** The Orchestrator sends this combined input to the LLM API.
* **LLM's Decision Process (Reasoning):** The LLM analyzes the comprehensive prompt. Its decision to respond directly or use tools stems from its training on vast datasets, including instruction following and tool-use examples, combined with the specific context provided:
    * It assesses if the current Message History and its internal knowledge are sufficient to answer the query according to the System Prompt's instructions.
    * It evaluates the available Tools (via their descriptions) to see if any tool can provide necessary information more accurately or efficiently than its internal knowledge (e.g., precise classification probabilities, segmenting a specific region).
    * If the query is ambiguous or lacks information, it might decide to ask a clarifying question instead of proceeding or using a tool.
* **LLM Response Generation:** Based on its decision:
    * **Option A: Answer/Clarify Directly:** Generates only a text response. This could be the final answer *or* a question back to the user asking for more information (e.g., "Which lung are you referring to?").
    * **Option B: Use Tools:** Generates a response containing structured **Tool Call Request(s)**. It can request **multiple different tools** or the **same tool multiple times with different arguments** within a single response if needed for parallel information gathering (e.g., classify pathologies *and* segment the lungs).
* **Memory Update:** The Orchestrator adds the LLM's `AIMessage` (containing either text, tool calls, or both) to the Memory.

**Step 2: Decision Point (Parse LLM Response)**
* The Orchestrator examines the `AIMessage` from Step 1.
* **If no Tool Call Request is present:** The LLM chose Option A. The Orchestrator routes to Step 4 (End), delivering the LLM's text response (which might be the final answer or a clarifying question).
* **If Tool Call Request(s) are present:** The LLM chose Option B. The Orchestrator routes to Step 3 (Action).

**Step 3: Action (Tool Execution)**
* The Orchestrator processes **each** Tool Call Request found in the LLM's response (potentially handling multiple calls concurrently if designed for parallelism):
    1.  **Identify Tool:** Looks up the requested `tool_name` in the Toolbox to find the corresponding Tool Wrapper object/class.
    2.  **Validate & Extract Arguments:** Parses the `arguments` dictionary provided by the LLM for that specific call. It checks if the arguments match the tool's defined `input_schema`. If validation fails, an error is generated.
    3.  **Invoke Tool:** If arguments are valid, the Orchestrator calls the specific execution function/method within the Tool Wrapper (e.g., `tool_wrapper.execute(**arguments)`). This function contains the actual code to interact with the specialized model (e.g., load model, preprocess input, run inference, postprocess output).
    4.  **Receive Result:** The Orchestrator captures the return value from the tool's execution logic. This is typically a string or simple structured data (like JSON) representing the tool's findings or an error message if execution failed internally.
    5.  **Format Result:** The Orchestrator packages this result into a structured **Tool Result Message**. This message includes the result content and the unique ID of the corresponding Tool Call Request from Step 1.
* **Memory Update:** The Orchestrator adds *all* the Tool Result Messages generated from executing the requested calls in this step to the Memory.

**Step 4: Loop or End (Observe & Rethink)**
* **If coming from Step 3 (Action):** The Orchestrator loops back to **Step 1 (Reasoning)**.
    * **Observe & Rethink:** This is where the agent processes the tool outputs. The LLM receives the updated Memory, which now includes the `Tool Result Message(s)`. It sees its previous reasoning, its request to use tools, and the *actual results* returned by those tools. It can now:
        * Synthesize the tool results with the previous context.
        * Correct its understanding if a tool returned unexpected information.
        * Decide if more reasoning or different tool calls are needed.
        * Formulate a final answer based on the gathered evidence.
* **If coming from Step 2 (Decision):** The LLM's last response (without tool calls) is considered final for this cycle. The Orchestrator presents this text response to the user. The process concludes for this query.

## 4. The Role of Prompts and Guidance

The agent's behavior is heavily guided by:

1.  **Explicit System Prompt:** Sets the overall goal, persona, and constraints (e.g., "reason step-by-step," "use tools judiciously"). A well-crafted prompt is essential for reliable performance.
    * *Example MedRAX Snippet:* "...Prioritize identifying key findings using classification and VQA tools first. If localization is needed, use the grounding or segmentation tools based on the findings. Synthesize all results before concluding..."
2.  **Implicit Prompting via Tool Descriptions:** The `description` field in each Tool Wrapper is critical. It's how the LLM learns *what the tool does* and *when it might be useful*. Clear, accurate, and distinct descriptions are vital for correct tool selection by the LLM.
3.  **Implicit Guidance via Message History:** The chronological sequence of messages in Memory provides context. The LLM learns from previous steps, including its own reasoning, the tools it called, and the results it received.

## 5. Handling Multimodality (Images)

* Image data needs to be represented within the Memory/Messages (e.g., using URIs, IDs, or embedded data like base64).
* The chosen LLM must be able to process these image representations alongside text.
* Tool Wrappers that operate on images must be able to access the image data based on the reference passed in their arguments (e.g., load image from URI).

## 6. Implementing the Workflow Orchestrator (Nodes & Edges)

While the concept is a state machine, libraries like LangGraph provide structures to implement this:

* **Nodes (Processing Steps):** These are typically implemented as Python functions or class methods. Each function representing a node (like `ReasoningStep` or `ActionStep`) usually:
    * Accepts the current agent Memory/State object as input.
    * Performs its specific logic (e.g., calling the LLM, executing tool wrappers).
    * Returns a dictionary containing the updates to be made to the Memory/State (e.g., `{"messages": [new_ai_message]}` or `{"messages": [list_of_tool_results]}`).
* **Edges (Transitions):** These define the flow *between* the node functions.
    * **Standard Edges:** Represent unconditional transitions. You configure the orchestrator by stating: "After node 'ActionStep' finishes, always call node 'ReasoningStep' next."
    * **Conditional Edges:** Implement decision points. You provide the orchestrator with a *routing function*. This function:
        * Takes the current Memory/State as input.
        * Applies logic to it (e.g., checks the last message for tool call requests, as in Step 2).
        * Returns a value (typically a string) indicating the *name* of the *next node* to execute (e.g., return `"ActionStep"` if tool calls exist, return `"__end__"` if not). The orchestrator uses this returned name to direct the flow.

## 7. Key Implementation Considerations

* **LLM Choice:** Select an LLM with proven multimodal understanding and reliable tool-calling adherence.
* **Tool Wrapper Quality:** Robustness, clear descriptions, and effective error handling in the wrappers are paramount.
* **Prompt Engineering:** Iteratively refining the System Prompt and Tool Descriptions is often necessary.
* **Error Handling Strategy:** Define how the Orchestrator and LLM should react to tool failures (e.g., retry, inform user, use alternative tool). This logic needs explicit implementation.
    * **Refined Error Handling:** In Step 3 (Action), if a tool fails, the formatted `Tool Result Message` should clearly indicate the error. When the flow loops back to Step 1 (Reasoning), the LLM sees this error message and can decide how to proceed based on its instructions (e.g., try a different tool, ask the user for clarification, or inform the user it cannot complete the request).
* **State Management:** Choose how to handle potentially very long conversation histories (Memory) if needed (e.g., summarization, windowing).
