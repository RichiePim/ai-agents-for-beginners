# Interactive Chat Implementation Guide

## Overview
This document explains how to create an interactive chat interface for AI agents using Jupyter notebooks with ipywidgets. The implementation allows users to have real-time conversations with AI agents while maintaining conversation context.

## Complete Interactive Chat Code

```python
import ipywidgets as widgets
from IPython.display import display, HTML
import asyncio

# --- 1. Widgets for user input and output display ---
input_box = widgets.Text(
    value='',
    placeholder='Type your message and press Enter',
    description='You:',
    disabled=False,
    layout=widgets.Layout(width='80%')
)
output_area = widgets.Output()

# Display the widgets
chat_box = widgets.VBox([output_area, input_box])
display(chat_box)

# --- 2. State for chat ---
thread: ChatHistoryAgentThread | None = None
agent_name = agent.name if hasattr(agent, "name") else "Assistant"

# --- 3. Async function to process user input and display agent response ---
async def process_user_input(user_input):
    global thread
    html_output = (
        f"<div style='margin-bottom:10px'>"
        f"<div style='font-weight:bold'>User:</div>"
        f"<div style='margin-left:20px'>{user_input}</div></div>"
    )
    full_response = []
    function_calls = []
    current_function_name = None
    argument_buffer = ""
    async for response in agent.invoke_stream(
        messages=user_input,
        thread=thread,
    ):
        thread = response.thread
        content_items = list(response.items)
        for item in content_items:
            if isinstance(item, FunctionCallContent):
                if item.function_name:
                    current_function_name = item.function_name
                if isinstance(item.arguments, str):
                    argument_buffer += item.arguments
            elif isinstance(item, FunctionResultContent):
                if current_function_name:
                    formatted_args = argument_buffer.strip()
                    try:
                        parsed_args = json.loads(formatted_args)
                        formatted_args = json.dumps(parsed_args)
                    except Exception:
                        pass
                    function_calls.append(f"Calling function: {current_function_name}({formatted_args})")
                    current_function_name = None
                    argument_buffer = ""
                function_calls.append(f"\nFunction Result:\n\n{item.result}")
            elif isinstance(item, StreamingTextContent) and item.text:
                full_response.append(item.text)
    if function_calls:
        html_output += (
            "<div style='margin-bottom:10px'>"
            "<details>"
            "<summary style='cursor:pointer; font-weight:bold; color:#0066cc;'>Function Calls (click to expand)</summary>"
            "<div style='margin:10px; padding:10px; background-color:#f8f8f8; "
            "border:1px solid #ddd; border-radius:4px; white-space:pre-wrap; font-size:14px; color:#333;'>"
            f"{chr(10).join(function_calls)}"
            "</div></details></div>"
        )
    html_output += (
        "<div style='margin-bottom:20px'>"
        f"<div style='font-weight:bold'>{agent_name}:</div>"
        f"<div style='margin-left:20px; white-space:pre-wrap'>{''.join(full_response)}</div></div><hr>"
    )
    output_area.append_display_data(HTML(html_output))

# --- 4. Handler for input box ---
def on_submit(change):
    user_input = change.value
    if user_input.strip():
        asyncio.create_task(process_user_input(user_input))
        input_box.value = ''  # Clear input

input_box.on_submit(on_submit)

# --- 5. Optionally, display the agent's intro message at the start ---
output_area.append_display_data(HTML(
    f"<div style='margin-bottom:20px'><div style='font-weight:bold'>{agent_name}:</div>"
    "<div style='margin-left:20px; white-space:pre-wrap'>Hello! I'm your TravelAgent assistant. I can help plan vacations and suggest interesting destinations for you. Here are some things you can ask me:\n"
    "1. Plan a day trip to a specific location\n"
    "2. Suggest a random vacation destination\n"
    "3. Find destinations with specific features (beaches, mountains, historical sites, etc.)\n"
    "4. Plan an alternative trip if you don't like my first suggestion\n\n"
    "What kind of trip would you like me to help you plan today?</div></div><hr>"
))
```

## Major Components Breakdown

### 1. **Widgets for Input/Output**
- `ipywidgets.Text` for user input
- `ipywidgets.Output` for displaying chat history
- Displayed together in a `VBox`

### 2. **State Management**
- `thread` variable to maintain chat history context
- `agent_name` for displaying the assistant's name

### 3. **Async Processing of User Input**
- `process_user_input` handles sending user input to the agent, streaming responses, and updating the output area
- Handles function calls and streaming text, similar to batch code, but now per user message

### 4. **Input Submission Handler**
- `on_submit` triggers when the user presses Enter, calls the async function, and clears the input box

### 5. **Initial Agent Message**
- Optionally displays a welcome message from the agent at the start

## Detailed Walkthrough: Steps 3, 4, and 5

### **Step 3: Thread State Management**

```python
# --- 2. State for chat ---
thread: ChatHistoryAgentThread | None = None
agent_name = agent.name if hasattr(agent, "name") else "Assistant"
```

**What is `thread`?**
- The `thread` is a **conversation context object** that maintains the chat history between you and the agent
- It starts as `None` (no conversation yet)
- Each time you send a message, the agent returns an **updated thread** containing the full conversation history
- This allows the agent to remember previous messages and maintain context

**Why use `global thread`?**
- The `thread` needs to persist across multiple user inputs
- Each function call needs to access and update the same thread object

### **Step 4: Async Function (`process_user_input`)**

```python
async def process_user_input(user_input):
    global thread  # Access the shared thread state
    
    # 1. Create HTML for user message display
    html_output = (
        f"<div style='margin-bottom:10px'>"
        f"<div style='font-weight:bold'>User:</div>"
        f"<div style='margin-left:20px'>{user_input}</div></div>"
    )
    
    # 2. Initialize response collectors
    full_response = []
    function_calls = []
    current_function_name = None
    argument_buffer = ""
    
    # 3. Send message to agent and stream response
    async for response in agent.invoke_stream(
        messages=user_input,
        thread=thread,  # Pass current thread (could be None for first message)
    ):
        thread = response.thread  # Update thread with new conversation state
        # ... process streaming content ...
    
    # 4. Display the complete response
    output_area.append_display_data(HTML(html_output))
```

**Key aspects:**
- **`async`**: Allows the function to handle streaming responses without blocking the UI
- **`global thread`**: Updates the shared conversation state
- **Streaming**: Processes chunks of response as they arrive from the agent
- **HTML Generation**: Builds formatted output for display

### **Step 5: Event Handler (`on_submit`)**

```python
def on_submit(change):
    user_input = change.value  # Get text from input box
    if user_input.strip():     # Only process non-empty input
        asyncio.create_task(process_user_input(user_input))  # Start async processing
        input_box.value = ''   # Clear input box immediately
```

**What happens here:**
- **Event Trigger**: Called automatically when user presses Enter in the text box
- **Input Extraction**: Gets the text the user typed
- **Async Task Creation**: Starts the async processing without waiting for it to complete
- **UI Update**: Immediately clears the input box so user can type the next message

## How They All Interact: The Complete Flow

### **First Message Scenario:**
```
1. User types: "What destinations are available?"
2. User presses Enter
   ↓
3. on_submit() is triggered
   - Extracts: "What destinations are available?"
   - Creates async task for process_user_input()
   - Clears input box
   ↓
4. process_user_input() starts executing
   - thread is currently None
   - Calls agent.invoke_stream(messages="What destinations...", thread=None)
   ↓
5. Agent processes the message and streams back response
   - Returns response object with updated thread
   - thread = response.thread (now contains first message + response)
   ↓
6. Response is displayed in output_area
```

### **Second Message Scenario:**
```
1. User types: "Is Barcelona available?"
2. User presses Enter
   ↓
3. on_submit() triggered again
   ↓
4. process_user_input() starts
   - thread now contains previous conversation
   - Calls agent.invoke_stream(messages="Is Barcelona...", thread=existing_thread)
   ↓
5. Agent sees full conversation context
   - Knows about previous "destinations" question
   - Can provide contextual response
   - Returns updated thread with all 4 messages (2 user + 2 agent)
   ↓
6. New response displayed, thread updated
```

## Key Interaction Points

1. **Thread Persistence**: The `global thread` variable ensures conversation memory across multiple interactions

2. **Async Non-blocking**: `asyncio.create_task()` allows the UI to remain responsive while the agent processes

3. **Immediate UI Feedback**: Input box clears instantly, giving user immediate feedback that their message was received

4. **Streaming Display**: As the agent streams its response, chunks are collected and displayed all at once when complete

## Visual Flow Diagram

```
User Input → on_submit() → asyncio.create_task() → process_user_input()
     ↑                           ↓                          ↓
Input cleared               Non-blocking              Update thread
                               start                   Display response
                                                           ↓
                                                    Ready for next input
```

## Key Changes from Batch to Interactive

- **Batch → Interactive**: Instead of looping over a list of user inputs, the code now waits for real user input via a text box and processes each message as it comes in.

- **Display**: Uses `ipywidgets` to create a persistent chat interface, rather than displaying HTML for each batch input.

- **State**: Maintains the `thread` variable across messages to preserve conversation context.

- **Async Handling**: Uses `asyncio.create_task` to allow non-blocking processing of user input.

This design creates a smooth, responsive chat experience where users can type naturally and see responses appear in real-time! 