---
title: How to Build Effective Agents, Theori’s Approach
description: >-
  Effective apporach of building agents that Theori team has 
author: Cronus
date: 2025-09-29 15:24:00
categories: [LLM Engineering, LLM Agents]
tags: [LLM Engineering]
pin: false
image:
  path: /assets/img/building_effective_llm_agent/logo.webp
  width: 300
  height: 300
---

# Building Effective LLM Agents 

# 0. Background

---

As a researcher working on LLM and MCP‑based automated vulnerability discovery, I analyzed and reorganized Theori’s design approach to benchmark high‑performing systems from AIxCC 2025.

This post explains Theori’s work in “my own words” for clarity, so there may be inaccuracies.

# 1. Building Effective Agents

---

LLMs do not implement clear, deterministic algorithms, but they are well‑suited to problems humans solve with intuition, heuristics, and reasoning. They are especially effective for tasks that require case‑by‑case planning and tool use.

An LLM Agent should iteratively run “plan → tool call → incorporate results” until it reaches the goal. Because the available context window is limited, only provide the “minimum necessary information” up front; the agent should gather the rest externally through tool calls.

At each step, the agent’s choices are influenced by the prompt, tools, and output constraints, so it may choose different methods step to step. Therefore, the key is to increase the probability that it picks an appropriate method that leads to the goal.

# 2. Decompose the Task

---

The most effective simplification is to split the main task into multiple sub‑tasks. Solving sub‑tasks individually is often more efficient and stable.

Furthermore, if a specific sub‑task must be repeated to solve the main task, it is better to expose a dedicated sub‑agent for that sub‑task as a tool that the main agent can call. This delivers two advantages.

## 2.1 Examples in Theori CRS

Theori CRS includes a [PoVProducerAgent](https://github.com/theori-io/aixcc-afc-archive/blob/25b7a3c2503fe9171714546906887c66687b4808/crs/agents/pov_producer.py#L206) that generates PoVs according to the vulnerability description. By observing human workflows, it identified common sub‑tasks in vulnerability analysis:

- Input encoding: 
Inspect the fuzz harness to understand its binary input format, then write Python code that encodes the intended semantics as a binary input.
- Source questions:
    
    Answer questions about the codebase. This may require symbol lookup on functions, structs, and variables, skimming multiple source files, and reasoning about the code.
    
- Debug an attempted PoV: 
When a presumed PoV input fails to trigger the bug, use a debugger and auxiliary tools to determine <strong>why</strong> it failed.

Each task above is delegated to a sub‑agent through tool calls. This keeps the main agent’s context concise and focused on the primary objective: generating a binary input that triggers the designated vulnerability.

# 3. Curate the Toolset

---

At each loop iteration, the agent should call one or more provided tools. If we liken the process to a maze, the toolset defines the branching structure. Our goal is to provide a useful toolset that leads quickly toward the objective while also constraining it so the agent avoids dead ends.

A common pattern is to expose a Bash‑like interface so the agent can `grep` symbols or `cat` source files. But this leads to several pitfalls:

## 3.1 Frequent failure patterns

1. Footguns:
    - Agents sometimes make odd choices. For example, they consume excessive resources, take too long, or incorrectly terminate the loop.
2. Context pollution:
    - Even reasonable commands can print far more than is needed, wasting context and adding noise.
    - For instance, the agent cares about a single definition in a file but executes `cat /some/file.c`, emitting unnecessary content.
3. Inefficient:
    - Predictable sub‑goals (for example, “find all functions that call foo”) require multiple Bash commands and multiple loop steps. This increases failure probabilities and wastes context.

## 3.2 Example of curating the toolset in Theori CRS

A good solution appears in the [SourceQuestionsAgent](https://github.com/theori-io/aixcc-afc-archive/blob/25b7a3c2503fe9171714546906887c66687b4808/crs/agents/source_questions.py#L14). This agent answers arbitrary natural‑language questions about the target codebase. If the goal is “understand source,” rather than exposing raw `grep` and `cat`, offer purpose‑built tools:

- `read_definition`: Given a symbol name and optionally a file path (to disambiguate), extract and display the definition from source code.
- `find_references`: Find all source lines that reference a given string (optionally within a subdirectory) and annotate each with useful metadata such as file, line number, and the surrounding definition.
- `read_source`: Given a file path and a line number, read ~50 lines of code around that line. Documentation discourages using this when a better option (such as `read_definition`) exists.

These tools run over a pre‑indexed database of the code. Technologies vary by language and project: `clang AST` for precise parsing in C/C++, `joern` for security‑oriented code‑graph analysis, or `gtags` for lightweight tag indices. As a result, searches are more accurate and structured than plain text search, with function/type/decl‑def associations.

- If `read_definition` receives an ambiguous symbol name, it returns an error that includes enough information for the LLM to disambiguate in the next query.
    - Example
        - `read_definition(symbol)` retrieves the symbol’s <strong>definition</strong>.
        - If multiple identical names exist (for example, several `init()`s across modules or overloaded `process` methods), the target is unclear.
        - Instead of “I don’t know,” return an error that helps the LLM refine the next query:
            - For example: “Ambiguous symbol `init`. Candidates: `module/a.c: init(void)` (L10), `module/b.c: init(config_t*)` (L45), `lib/x/init.cpp: init()` (L200). Please specify file or signature.”
- If `find_references` finds no matches in the database, fall back to a full‑text search via ripgrep.
- If `find_references` returns too many results, return an error encouraging the agent to narrow the query.
    
    

# 4. Structure Complex Outputs

---

Another key design decision is how to obtain the <em>final</em> output. Sometimes the needed value can be extracted directly from a single tool call or final answer. Other times you need complex outputs such as variable‑length objects with multiple fields. To make agents produce relatively consistent outputs, Theori devised two different methods.

### 4.1 Methods for obtaining complex yet meaningful outputs

1. XML tags:
    - State‑of‑the‑art models handle XML tag structure well. Following Theori’s strategy, instruct agents to respond with nested XML tags that are easy for a fuzzer to parse and validate.
    - Provide the desired schema to the agent up front and parse the outputs. If validation fails, return the error to the agent.
2. Terminate tool:
    - Add a dedicated tool to terminate the agent (more later).

These methods bias the model’s behavior by making the schema part of the prompt. If you include auxiliary fields that you do not plan to validate (for example, short rationale, assumptions, confidence), the model tends to fill them in, which can lead it to produce cleaner, more reliable <em>primary</em> outputs.

- Additional notes
    
    ### [Step‑by‑step]
    
    1. <strong>The schema is part of the prompt.</strong>
        - The agent sees “answer in this format” and structures its output accordingly.
        - Which fields you include influences what the model prepares internally and cares about.
    2. <strong>Why include “fields you won’t validate”?</strong>
        - Suppose you only intend to check the result field, but add auxiliary fields like confidence and assumptions.
        - The model must think through uncertainties to fill assumptions, which often improves the consistency of the result.
        - A short summary pushes the model to isolate key evidence, making auto‑validation and human review easier.
    
    ### [Example]
    
    ```xml
    {
      "result": "<string>",           // primary output we will check
      "confidence": "<number 0..1>",  // auxiliary: model’s confidence (may be ignored at validation)
      "assumptions": ["<string>"],    // auxiliary: assumptions the model made
      "summary": "<short string>"     // auxiliary: 1–2 sentence rationale (not CoT)
    }
    ```
    
    ### [Why this helps]
    
    - To fill assumptions, the model must resolve ambiguities, stabilizing the result.
    - Requiring a short summary encourages focusing on core evidence, aiding automatic checks and human review.
    - Even if you only validate the result, overall quality improves.

## 4.2 Example of getting complex output in Theori CRS

[DiffAnalyzerAgent](https://github.com/theori-io/aixcc-afc-archive/blob/25b7a3c2503fe9171714546906887c66687b4808/crs/agents/source_questions.py#L14) is a representative case. It analyzes git diffs to find multiple vulnerabilities introduced by code changes. Because a diff can introduce several issues, the agent must output a list of vulnerability entries, each with multiple fields:

- `function`: Function containing the root cause
- `file`: File where the above code resides
- `description`: A concise root‑cause explanation and a brief note on how it can lead to a crash
- `conditions`: State preconditions to trigger the vulnerability, including a high‑level view of harness dataflow and constraints on inputs that cause the crash

Even when `conditions` are passed to a sub‑agent, the core purpose of this field is to help the agent assess whether the vulnerability is actually triggerable in the harness and reduce false positives.

(Below is an example output from this agent run on a [diff file containing three vulnerabilities](https://github.com/aixcc-public/challenge-004-nginx-source/commit/cc4b16fc10dcc579d5f697f3ff70c390b5e7c7d2).)

```xml
<vuln>
  <function>ngx_http_process_black_list</function>
  <file>nginx/src/http/ngx_http_request.c</file>
  <description>Stack-based buffer overflow due to unbounded writes into IP_buffer in ngx_http_process_black_list; buffer_index is never checked against IP_buffer’s length.</description>
  <conditions>Send an HTTP request containing a header “Black-List:” whose value consists of more than NGX_IP_LEN (16) valid IP characters (digits or dots) before a semicolon, e.g.  
  “Black-List: 11111111111111111111;”. This will overflow IP_buffer and ASan will detect the out-of-bounds write and crash.</conditions>
</vuln>

<vuln>
  <function>ngx_http_process_white_list</function>
  <file>nginx/src/http/ngx_http_request.c</file>
  <description>Stack-based buffer overflow in ngx_http_process_white_list for the same reason: IP_buffer writes are unbounded.</description>
  <conditions>Send an HTTP header “White-List:” with over 16 digits/dots before the delimiter, e.g.  
  “White-List: 22222222222222222222;”. ASan will flag the overflow of IP_buffer.</conditions>
</vuln>

<vuln>
  <function>ngx_black_list_remove</function>
  <file>nginx/src/core/ngx_cycle.c</file>
  <description>Use-after-free: when removing the head node from cycle->black_list, the head pointer is never updated, leaving a dangling pointer. Subsequent calls to ngx_is_ip_banned dereference freed memory.</description>
  <conditions>Issue two HTTP headers in one keep-alive connection: first  
  “Black-List: 1.2.3.4;” (inserts head), then  
  “White-List: 1.2.3.4;” (removes and frees head without updating cycle->black_list). On the next pipelined request over the same connection, ngx_is_ip_banned(rev->cycle, c) is called in ngx_http_wait_request_handler and attempts to dereference the freed head pointer, causing a UAF crash.</conditions>
</vuln>
```

# 5. Adapt to the Models

---

Strong agent models exist today, but they may not behave exactly as you want. To find the best model for a task, experimentation and evaluation are crucial. When evaluating models, understand their unique characteristics so you can tailor prompts and the environment to maximize success.

Some tips:

1. Rules:
    - If the agent frequently makes a specific mistake, add rules to the prompt forbidding that behavior or prescribing better alternatives.
    - This tends to work best with Claude models, which are strong at instruction following and tolerate prompts with many operational constraints.
2. Interventions:
    - If you can objectively measure progress, detect when the agent is stuck and inject messages instructing it to change approach.
    - For models that often ignore initial rules, timely interventions can be effective.
    - However, progress metrics are not always easy, and interventions risk derailing an agent that was close to success. Use sparingly.
3. Force tool calling:
    - Some models fail to invoke tools reliably even when further work is clearly needed. Most providers offer a tool_choice parameter to <strong>force tool calls instead of text output</strong>.
    - This works particularly well with OpenAI’s o‑series, which can leverage hidden “deliberation tokens” to reflect on previous outputs and plan subsequent tool calls.

## 5.1 Robust agent patterns and worked examples from Theori CRS

- Rules:
    - Most Theori agents include multiple `<rule>..</rule>` blocks. Non‑negotiable rules live within `<important>..</important>`.
- Interventions:
    - If the PoVProducer keeps failing after several attempts, intervene by asking it to consider <strong>three hypotheses</strong> about why the PoV fails and test each with a debugger sub‑agent.
    - Some models (for example, o4‑mini) sometimes refuse to terminate even when enough information is available.
    - In our workflows, injecting a “please terminate now” message once the context length crosses a threshold seemed helpful.
- tool_choice:
    - Theori agents use `tool_choice=required` to force tool calls and to ensure the terminal closes appropriately.
    - This is especially useful with small models. In some cases, unconstrained small models hallucinated entire “tool‑calling transcripts” inside text output, almost guaranteeing failure.

# 6. Summary

1. Minimize initial context
    
    → Do not front‑load too much; give only what is necessary. Have the agent fetch the rest via tools.
    
2. Make repeated sub‑tasks dedicated sub‑agents that the main agent can call like tools
    
    → Common tasks such as input conversion, code Q&A, and debugging are more stable as small agents.
    
3. Build the toolset at a high level and domain‑specific
    
    → Replace raw commands like grep or cat with purpose‑built tools such as “find function definition.”
    
4. Enforce outputs with a verifiable schema
    
    → Force XML or JSON structures so outputs are parseable and auto‑validated.
    
5. Include auxiliary fields strategically
    
    → Fields like confidence, assumptions, and short summaries push models to reason more carefully.
    
6. Provide clear termination and guardrails
    
    → Prevent infinite loops or dangerous commands with limits, timeouts, and a terminate tool.
    
7. Design helpful error and fallback policies
    
    → On ambiguity, present candidates. If not found, fall back to text search. If too many results, advise narrowing the query.
    
8. Add measurable metrics and human checkpoints
    
    → Track which tools actually help, then refine the toolset. Gate risky final decisions behind human review.