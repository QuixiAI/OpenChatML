# OpenChatML Specification v0.1

## 1. Introduction
OpenChatML is a markup language designed for representing conversational data in a structured format. It provides a standardized way to encode chat messages, including the role of the speaker, the content of the message, and optional metadata such as the name of the speaker.

### 1.1 Overview

This document defines OpenChatML, a markup language for representing and exchanging conversational data in a standardized format. OpenChatML provides a structured approach to encode chat messages, including the role of the speaker, the content of the message, and optional metadata.

### 1.2 Purpose

The purpose of OpenChatML is to address the following challenges in the conversational AI domain:

1. Lack of interoperability between different platforms, tools, and systems.
2. Inconsistency in the representation of conversational data across various sources and target systems.
3. Limited expressiveness in capturing the nuances and context of conversations.
4. Difficulty in extending and evolving existing formats to meet the changing needs of the conversational AI community.

### 1.3 Scope

OpenChatML is designed to be a lightweight and flexible markup language for representing conversational data. It focuses on the core elements and structures necessary to capture the essence of conversations, while allowing for extensibility and customization.

The specification defines the syntax and semantics of OpenChatML, including the special tokens, message structure, conversation structure, fill-in-the-middle tasks, and multi-file sequences, and function calling. It also provides guidelines for parsing and generating OpenChatML data.

### 1.4 Comparison to Other Specifications

OpenChatML differs from other specifications in the following aspects:

1. Simplicity: OpenChatML prioritizes simplicity and readability, making it easy for humans to understand and write conversational data in this format.
2. Flexibility: The specification allows for optional metadata attributes to be associated with each message, providing flexibility without imposing strict requirements.
3. Fill-in-the-Middle Tasks: OpenChatML provides built-in support for fill-in-the-middle tasks, which are commonly used in conversational AI for completion and generation tasks.
4. Multi-File Sequences: OpenChatML introduces the concept of multi-file sequences, enabling the representation of conversations that span multiple files or documents.

### 1.5 Use Cases

OpenChatML is intended to be used in various conversational AI scenarios, including:

1. Dialogue Systems: Representing and storing conversation data for training and evaluating dialogue systems.
2. Chatbots: Building and deploying chatbots across different platforms, ensuring consistent handling of conversational data.
3. Conversational Datasets: Creating, sharing, and analyzing conversational datasets for research and development purposes.
4. Human-in-the-Loop Interactions: Representing and capturing interactions between human agents and AI systems for training and evaluation.

## 2. Tokens
OpenChatML uses the following special tokens:

- `[BOS]`: Rather than redefining it, we will use the same BOS token as the base model uses. This token indicates the start of a conversation.
- `[EOS]`: Rather than redifining it, we will use the same EOS token as the base model uses. This token indicates the end of a conversation.
- `<|im_start|>`: Start of Turn token, indicating the beginning of a new message within the conversation. 
- `<|im_end|>`: End of Turn token, indicating the end of the current message.
- `<|fim_prefix|>`: Before Cursor token, indicating the content before the cursor in a fill-in-the-middle task.
- `<|fim_suffix|>`: After Cursor token, indicating the content after the cursor in a fill-in-the-middle task.
- `<|fim_middle|>`: At Cursor token, indicating where the model should fill in content in a fill-in-the-middle task.
- `<|file_separator|>`: File Separator token, used to separate content from different files within the same sequence.
- `<|reflect|>`,`<|introspect|>`,`<|reason|>`: flag tokens to indicate whether these blocks should be generated.
- `<|start_reflect|>`, `<|end_reflect|>`: a block containing reflection on the user's position and intent.
- `<|start_introspect|>`, `<|end_introspect|>`: a block containing introspection on the model's position and intent.
- `<|start_reason|>`, `<|end_reason|>`: a block containing chain of thought required to solve the question.
- `<|function_list|>`: Function List token, for providing available tools.
- `<|function_output|>`: Function Output token, indicating output from a tool use.
- `<|function_call|>`: Function Call token, indicating tool to call.

## 3. Message Structure
Each message in OpenChatML is represented as follows:

```
<|im_start|>role [name=<name>]
message_content
<|im_end|>
```

- `role`: A string indicating the role of the speaker. It must be one of the following: "system", "tool", "user", or "assistant".
- `name` (optional): A string representing the name of the speaker. If present, it should be added after the role, in the format `name=<name>`. The name cannot contain whitespace.
- `message_content`: The actual content of the message, which can span multiple lines.

## 4. Thought Structure
OpenChatML introduces a new structure to represent reflection, introspection, and reason.  If the flags are present in the system prompt, that indicates to the model to generate the associated thought block.

Reflection is analysis of the user's emotions and intentions.  Introspection is the model's analysis of its own emotions and intentions.  Reason is the chain of thought to solve problems.

This behavior will need to be trained into the model with SFT.

Example:

```
[BOS]<|im_start|>system
You are a helpful AI assistant.<|reflect|><|introspect|><|reason|><|im_end|>
<|im_start|>user
I have here a closed box with the label Band-Aid printed on it. What do you suppose is inside the box?<|im_end|>
<|im_start|>assistant
<|start_reflect|>The user is asking a straightforward question about the likely contents of a labeled box. They seem curious but not emotionally invested in the answer. There is no apparent hidden agenda or ulterior motive behind the question.<|end_reflect|>
<|start_introspect|>As an AI assistant, my goal is to provide helpful and informative responses to the user's questions. In this case, I do not have any strong feelings or opinions about the contents of the box. My intention is to make a reasonable inference based on the label and common knowledge.<|end_introspect|>
<|start_reason|>The box has a "Band-Aid" label printed on it. Band-Aid is a well-known brand name for adhesive bandages and related first-aid products. When a box is labeled with a brand name, it very often contains that brand's products inside. Therefore, the most logical inference is that this box contains Band-Aid brand adhesive bandages.<|end_reason|>
Based on the "Band-Aid" label printed on the box, it is very likely that the box contains Band-Aid brand adhesive bandages inside. Band-Aid is a famous brand name for this type of first-aid product. When a box has a brand name printed on the outside, it usually contains that brand's products on the inside. So while I cannot say with 100% certainty, I think it is a very reasonable assumption that you would find Band-Aid adhesive bandages if you opened up this labeled box.<|im_end|>[EOS]
```

## 5. Conversation Structure
A conversation in OpenChatML is represented as a sequence of messages, enclosed within `[BOS]` and `[EOS]` tokens:

```
[BOS]<|im_start|>role1 [name=<name1>]
message1
<|im_end|>
<|im_start|>role2 [name=<name2>]
message2
<|im_end|>
...
[EOS]
```

## 6. Fill-in-the-Middle Tasks
OpenChatML supports fill-in-the-middle (FIM) tasks where the model is asked to complete content given surrounding context. The FIM structure is represented as:

```
<|fim_prefix|>prefix_content<|fim_middle|><|fim_suffix|>suffix_content
```

The model should generate content to replace the `<|fim_middle|>` token, using `prefix_content` as the preceding context and `suffix_content` as the following context. The generated content should smoothly connect the prefix to the suffix.

## 7. Multi-File Sequences
OpenChatML allows combining content from multiple files into a single sequence using the `<|file_separator|>` token:

```
file1_content
<|file_separator|>
file2_content
<|file_separator|>
file3_content
```

The `<|file_separator|>` token is used to demarcate the boundaries between content from different files while keeping them as part of the same overall sequence. This can be useful for tasks involving multiple input sources.

## 8. Function Calling
OpenChatML supports function calling, allowing the model to interact with external tools and APIs. Function calling enables the model to perform specific tasks, retrieve information, and generate more accurate and relevant responses based on the available tools.  The design for function calling in OpenChatML is adapted from the [Hermes-Function-Calling](https://github.com/NousResearch/Hermes-Function-Calling) project, by Nous Research.

### 8.1 Function Signature

To enable function calling, the available functions or tools should be provided to the model after the `<|function_list|>` tag in the system message. The function signature is represented as a JSON object with the following properties:

- `type`: Indicates the type of the tool, which should be "function".
- `function`: An object representing the function details, containing:
  - `name`: The name of the function.
  - `description`: A brief description of what the function does.
  - `parameters`: An object specifying the parameters of the function, following the JSON Schema format.

Example function signature:

```json
<|function_list|>
{
  "type": "function",
  "function": {
    "name": "get_stock_fundamentals",
    "description": "Get fundamental data for a given stock symbol using yfinance API.",
    "parameters": {
      "type": "object",
      "properties": {
        "symbol": {
          "type": "string"
        }
      },
      "required": ["symbol"]
    }
  }
}
```

### 8.2 Function Call

To make a function call, the model should generate a JSON object after the `<|function_call|>` tag. The JSON object should follow the Pydantic model schema:

```json
{
  "title": "FunctionCall",
  "type": "object",
  "properties": {
    "arguments": {
      "title": "Arguments",
      "type": "object"
    },
    "name": {
      "title": "Name",
      "type": "string"
    }
  },
  "required": ["arguments", "name"]
}
```

Example function call:

```
<|function_call|>
{"arguments": {"symbol": "TSLA"}, "name": "get_stock_fundamentals"}
```

### 8.3 Function Response

After executing the function call, the response should be passed back to the model aftert the `<|function_output|>` tag. The response should be a JSON object containing the function name and the content of the response.

Example function response:

```
<|function_output|>
{
  "name": "get_stock_fundamentals",
  "content": {
    "symbol": "TSLA",
    "company_name": "Tesla, Inc.",
    "sector": "Consumer Cyclical",
    "industry": "Auto Manufacturers",
    "market_cap": 611384164352,
    "pe_ratio": 49.604652,
    "pb_ratio": 9.762013,
    "dividend_yield": null,
    "eps": 4.3,
    "beta": 2.427,
    "52_week_high": 299.29,
    "52_week_low": 152.37
  }
}
```

### 8.4 Recursive Function Calls

OpenChatML allows for recursive function calls, where the model can make multiple function calls in a single conversation turn. The model can generate multiple `<|function_call|>` tags, and the corresponding `<|function_output|>` tags should be provided in the same order.

### 8.5 Example Conversation with Function Calling

Here's an example conversation demonstrating function calling in OpenChatML:

```
[BOS]<|im_start|>system
You are a function calling AI model. You are provided with function signatures after the `<|function_list|>` tag. You may call one or more functions to assist with the user query. Don't make assumptions about what values to plug into functions. Here are the available tools:
<|function_list|>
{
  "type": "function",
  "function": {
    "name": "get_stock_fundamentals",
    "description": "Get fundamental data for a given stock symbol using yfinance API.",
    "parameters": {
      "type": "object",
      "properties": {
        "symbol": {
          "type": "string"
        }
      },
      "required": ["symbol"]
    }
  }
}

Use the following pydantic model json schema for each tool call you will make:
{
  "title": "FunctionCall",
  "type": "object",
  "properties": {
    "arguments": {
      "title": "Arguments", 
      "type": "object"
    },
    "name": {
      "title": "Name",
      "type": "string"
    }
  },
  "required": ["arguments", "name"]
}
For each function call return a json object with function name and arguments after the <|function_call|> tag as follows:
<|function_call|>
{"arguments": <args-dict>, "name": <function-name>}
<|im_end|>
<|im_start|>user
Fetch the stock fundamentals data for Tesla (TSLA)
<|im_end|>
<|im_start|>assistant
<|function_call|>
{"arguments": {"symbol": "TSLA"}, "name": "get_stock_fundamentals"}
<|im_end|>
<|im_start|>tool
<|function_output|>
{
  "name": "get_stock_fundamentals",
  "content": {
    "symbol": "TSLA",
    "company_name": "Tesla, Inc.",
    "sector": "Consumer Cyclical", 
    "industry": "Auto Manufacturers",
    "market_cap": 611384164352,
    "pe_ratio": 49.604652,
    "pb_ratio": 9.762013,
    "dividend_yield": null,
    "eps": 4.3,
    "beta": 2.427,
    "52_week_high": 299.29,
    "52_week_low": 152.37
  }
}
<|im_end|>
<|im_start|>assistant
The stock fundamentals data for Tesla (TSLA) are as follows:
- **Symbol**: TSLA  
- **Company Name**: Tesla, Inc.
- **Sector**: Consumer Cyclical
- **Industry**: Auto Manufacturers
- **Market Capitalization**: $611,384,164,352
- **Price-to-Earnings Ratio (P/E Ratio)**: 49.60
- **Price-to-Book Ratio (P/B Ratio)**: 9.76
- **Dividend Yield**: N/A
- **Earnings Per Share (EPS)**: $4.30
- **Beta**: 2.427
- **52-Week High**: $299.29
- **52-Week Low**: $152.37

This data provides key financial metrics and ratios that can help assess Tesla's stock performance and valuation. The high P/E and P/B ratios suggest that investors have high expectations for Tesla's future growth. The lack of a dividend yield indicates that Tesla is reinvesting its profits into the business rather than distributing them to shareholders. The high beta value implies that Tesla's stock is more volatile than the overall market.

<|im_end|>[EOS]
```

## 9. Examples
Here are a few examples of OpenChatML structures:

Example conversation:
```
[BOS]<|im_start|>user
Hello there, AI.
<|im_end|>
<|im_start|>assistant
Hi. Nice to meet you.
<|im_end|>[EOS]
```

Example conversation with speaker name:
```
[BOS]<|im_start|>user name=Eric
Hello there, AI.
<|im_end|>
<|im_start|>assistant
Hi Eric. Nice to meet you.
<|im_end|>[EOS]
```

Example fill-in-the-middle task:
```
<|fim_prefix|>The capital of France is <|fim_middle|><|fim_suffix|>, which is known for its famous Eiffel Tower.
```

Example multi-file sequence:
```
This is the content from the first file.
<|file_separator|>
This is the content from the second file.
And this is more content from the second file.
<|file_separator|>
Finally, this is the content from the third file.
```

Here are some more comprehensive examples showcasing various features and use cases of OpenChatML:

Example: Named roles and longer conversation
```
[BOS]<|im_start|>system name=GoalTracker
You are an AI assistant that helps users track and achieve their goals.
<|im_end|>
<|im_start|>user name=Alice
Hi, I need help staying motivated to reach my fitness goals. Any advice?
<|im_end|>
<|im_start|>assistant name=FitnessCoach
Hi Alice! To stay motivated, try setting specific, measurable goals and tracking your progress. Celebrate small victories along the way. Remember why you started this journey and keep that in mind when things get tough. I'm here to support you every step of the way!
<|im_end|>
<|im_start|>user name=Alice  
Thanks, that's helpful! Can you suggest a good workout routine for beginners?
<|im_end|>
<|im_start|>assistant name=FitnessCoach
Absolutely! Here's a simple beginner-friendly workout routine:

Monday: 30-minute brisk walk or jog
Tuesday: 15 minutes of bodyweight exercises (squats, push-ups, lunges)
Wednesday: Rest day or light stretching 
Thursday: 30-minute bike ride or swim
Friday: 15 minutes of bodyweight exercises
Saturday: 30-minute yoga or Pilates session
Sunday: Rest day

Remember to warm up before and cool down after each workout. Listen to your body and don't push too hard too soon. Consistency is key!
<|im_end|>
<|im_start|>user name=Bob
Hey, I couldn't help but overhear. I'm also trying to get into a fitness routine. Mind if I join the conversation?
<|im_end|>  
<|im_start|>assistant name=FitnessCoach
Of course, Bob! The more, the merrier. Feel free to chime in with your own experiences and questions. We can all learn from and support each other on our fitness journeys.
<|im_end|>[EOS]
```

Example: Fill-in-the-middle task
```
<|fim_prefix|>def fibonacci(n):
    if n <= 0:
        return []
    elif n == 1:
        return [0]
    elif n == 2:
        return [0, 1]
    else:
        fib = [0, 1]
        <|fim_middle|><|fim_suffix|>
        return fib

# Test the function
print(fibonacci(10))
```

Completion:
```
<|fim_prefix|>def fibonacci(n):
    if n <= 0:  
        return []
    elif n == 1:
        return [0]
    elif n == 2:
        return [0, 1]
    else:
        fib = [0, 1]
        <|fim_middle|>for i in range(2, n):
            fib.append(fib[i-1] + fib[i-2])
        <|fim_suffix|>  
        return fib

# Test the function
print(fibonacci(10))
```

Example 3: Multi-file sequence for document summarization
```
<|file_separator|>
A black hole is a region of spacetime where gravity is so strong that nothing, not even light, can escape from it. The boundary of a black hole is called the event horizon, beyond which events cannot affect an outside observer. Black holes form when massive stars collapse at the end of their life cycle. 
<|file_separator|>
The first modern solution of general relativity that would characterize a black hole was found by Karl Schwarzschild in 1916. However, its interpretation as a region of space from which nothing can escape was first published by David Finkelstein in 1958. Long considered a mathematical curiosity, it was during the 1960s that theoretical work showed black holes were a generic prediction of general relativity.
<|file_separator|>
The discovery of neutron stars by Jocelyn Bell Burnell in 1967 sparked interest in gravitationally collapsed compact objects as a possible astrophysical reality. The first black hole known as such was Cygnus X-1, identified by several researchers independently in 1971. Black holes of stellar mass form when very massive stars collapse at the end of their life cycle.
<|file_separator|>
<|fim_prefix|><|fim_middle|><|fim_suffix|> Despite their invisible interior, the presence of black holes can be inferred through their interaction with other matter and with electromagnetic radiation such as visible light. If there are other stars orbiting a black hole, their orbit can be used to determine the black hole's mass and location. Matter falling into a black hole can form an accretion disk, one of the brightest objects in the universe.
```

Completion:
```
<|fim_prefix|>Black holes are regions of spacetime where gravity is extremely strong, preventing anything, including light, from escaping. They form when massive stars collapse at the end of their life cycle. The first modern solution describing black holes was found by Karl Schwarzschild in 1916, but their interpretation as inescapable regions was published by David Finkelstein in 1958.
<|fim_middle|>The existence of black holes was confirmed with the discovery of Cygnus X-1 in 1971. Black holes can be detected through their interaction with nearby matter and radiation. Stars orbiting a black hole can reveal its mass and location, while matter falling into a black hole forms a bright accretion disk.
<|fim_suffix|> Despite their invisible interior, the presence of black holes can be inferred through their interaction with other matter and with electromagnetic radiation such as visible light. If there are other stars orbiting a black hole, their orbit can be used to determine the black hole's mass and location. Matter falling into a black hole can form an accretion disk, one of the brightest objects in the universe.
```

## 10. Parsing and Generation
When parsing OpenChatML, the following rules should be applied:
- The `role` must be one of the predefined values: "system", "tool", "user", or "assistant".
- The `name` attribute is optional and should be parsed if present.

When generating OpenChatML, the same structure and rules should be followed to ensure compatibility and consistency.

## 11. References
- Fill In the Middle (FIM) https://arxiv.org/abs/2207.14255
- Quiet Star (chain of thought) https://arxiv.org/abs/2403.09629
