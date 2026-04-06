# Module 04c -- Structured Output
> Prompt Engineering Series · Getting JSON and XML reliably, output schemas, parse failure handling, structured output APIs

---

## Table of Contents
1. [Why Structured Output Matters](#1-why-structured-output-matters)
2. [Prompting for JSON](#2-prompting-for-json)
3. [Handling Parse Failures](#3-handling-parse-failures)
4. [Using Structured Output APIs](#4-using-structured-output-apis)
5. [XML as an Alternative](#5-xml-as-an-alternative)
6. [Validation Beyond Parsing](#6-validation-beyond-parsing)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Why Structured Output Matters

LLMs produce text. Applications need structured data. This gap is where a lot of production failures happen.

When you ask a model for JSON and it returns JSON wrapped in markdown code fences with an explanation before and after, your JSON parser breaks. When you ask for a specific set of fields and the model decides to add extra ones or rename the ones you asked for, downstream code breaks.

Reliable structured output is a combination of three things: clear prompting that specifies exactly what format you need, defensive parsing code that handles variations in how the model returns output, and ideally a structured output API that enforces the schema at the model level.

---

## 2. Prompting for JSON

### Tell the Model Exactly What You Need

Be explicit about three things: return JSON, here is the exact schema, and nothing else.

```
Extract the following fields from the customer message and return them as a JSON object.
Return only the JSON object. Do not include any explanation, preamble, or markdown formatting.

Fields:
  customer_name  -- string, the name the customer used to identify themselves
  issue_type     -- string, one of: billing, technical, account, other
  urgency        -- string, one of: low, medium, high
  summary        -- string, a one-sentence description of the issue, maximum 50 words

Customer message:
"Hi, my name is Sarah and I have not been able to log into my account since yesterday.
 I have an important presentation tomorrow and really need access urgently."
```

### Show the Schema Structure

Include a concrete example of the JSON structure you expect. This removes ambiguity about field names, nesting, and data types.

```
Return a JSON object with exactly this structure:
{
  "customer_name": "string",
  "issue_type": "billing | technical | account | other",
  "urgency": "low | medium | high",
  "summary": "string, one sentence, max 50 words"
}
```

### Show a Complete Example

For complex schemas, include a worked example with both the input and the expected JSON output.

```
Example input:
"My invoice from last month shows a charge I did not authorize. My name is John Chen."

Example output:
{
  "customer_name": "John Chen",
  "issue_type": "billing",
  "urgency": "medium",
  "summary": "Customer reports an unauthorized charge on last month's invoice."
}
```

### Specify Null Handling

Tell the model what to do when a field cannot be extracted. Otherwise it will invent something or skip the field.

```
If the customer did not provide their name, set customer_name to null.
If urgency cannot be determined from the message, set urgency to "medium" as a default.
```

---

## 3. Handling Parse Failures

Even with excellent prompting, the model will occasionally return output that does not parse cleanly. This happens more often than you might expect in production, especially under varied inputs or when the input itself contains characters that interfere with JSON (like quotes or curly braces).

### Common Failure Patterns

The model wraps JSON in markdown code fences:
```
```json
{"customer_name": "Sarah", ...}
```
```

The model adds explanation before the JSON:
```
Here is the extracted information:
{"customer_name": "Sarah", ...}
```

The model adds a trailing comment:
```
{"customer_name": "Sarah", ...}  // extracted from the message above
```

The model uses single quotes instead of double quotes:
```
{'customer_name': 'Sarah', ...}
```

### Defensive Parsing

Write a parser that handles these common variations before giving up.

```python
import json
import re

def parse_json_response(llm_response: str) -> dict | None:
    response = llm_response.strip()

    # Attempt 1: direct parse
    try:
        return json.loads(response)
    except json.JSONDecodeError:
        pass

    # Attempt 2: extract from markdown code fences
    match = re.search(r'```(?:json)?\s*([\s\S]*?)\s*```', response)
    if match:
        try:
            return json.loads(match.group(1))
        except json.JSONDecodeError:
            pass

    # Attempt 3: find the first { ... } block in the response
    match = re.search(r'\{[\s\S]*\}', response)
    if match:
        try:
            return json.loads(match.group(0))
        except json.JSONDecodeError:
            pass

    # All attempts failed
    return None
```

### Retry on Parse Failure

When parsing fails, tell the model what went wrong and ask it to try again.

```python
def call_with_structured_retry(prompt: str, llm, max_retries: int = 3) -> dict | None:
    messages = [{"role": "user", "content": prompt}]

    for attempt in range(max_retries):
        response = llm.invoke(messages)
        result = parse_json_response(response)

        if result is not None:
            return result

        # Tell the model what went wrong
        messages.append({"role": "assistant", "content": response})
        messages.append({
            "role": "user",
            "content": "Your response was not valid JSON. Please return only a valid JSON object with no explanation, no markdown formatting, and no extra text."
        })

    return None
```

The retry message is specific about what was wrong and what to do differently. A generic "try again" message is less effective.

---

## 4. Using Structured Output APIs

OpenAI and Anthropic both provide APIs that enforce a schema at the model level. The model is constrained to produce output that exactly matches your schema. Parse failures are essentially eliminated.

This is the most reliable approach for production systems where structured output is critical.

### OpenAI Structured Output

Define your schema as a Pydantic model and pass it as the response format. The API guarantees the response will match the schema.

```python
from pydantic import BaseModel, Field
from typing import Literal
import openai

class CustomerIssue(BaseModel):
    customer_name: str | None = Field(description="The customer's name, or null if not provided")
    issue_type: Literal["billing", "technical", "account", "other"]
    urgency: Literal["low", "medium", "high"]
    summary: str = Field(description="One sentence summary of the issue, max 50 words")

client = openai.OpenAI()

response = client.beta.chat.completions.parse(
    model="gpt-4o",
    messages=[
        {
            "role": "system",
            "content": "Extract the customer issue details from the message."
        },
        {
            "role": "user",
            "content": "Hi, my name is Sarah and I cannot log into my account since yesterday..."
        }
    ],
    response_format=CustomerIssue
)

issue = response.choices[0].message.parsed
print(issue.customer_name)   # "Sarah"
print(issue.issue_type)      # "account"
print(issue.urgency)         # "high"
```

The `.parsed` attribute is already a `CustomerIssue` object. No JSON parsing needed.

### Anthropic Tool Use for Structured Output

Anthropic does not have the same parse API as OpenAI but you can achieve reliable structured output by defining the schema as a tool and forcing the model to use it.

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "extract_customer_issue",
        "description": "Extract structured customer issue details from a support message.",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_name": {
                    "type": ["string", "null"],
                    "description": "The customer name, or null if not mentioned"
                },
                "issue_type": {
                    "type": "string",
                    "enum": ["billing", "technical", "account", "other"]
                },
                "urgency": {
                    "type": "string",
                    "enum": ["low", "medium", "high"]
                },
                "summary": {
                    "type": "string",
                    "description": "One sentence summary, max 50 words"
                }
            },
            "required": ["customer_name", "issue_type", "urgency", "summary"]
        }
    }
]

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_customer_issue"},
    messages=[
        {
            "role": "user",
            "content": "Hi, my name is Sarah and I cannot log into my account since yesterday..."
        }
    ]
)

# Extract the tool call result
tool_use = next(block for block in response.content if block.type == "tool_use")
issue = tool_use.input   # already a dict matching your schema
```

Setting `tool_choice` to force a specific tool ensures the model always calls it rather than responding with plain text.

---

## 5. XML as an Alternative

JSON is the most common structured format for LLM output, but XML can work better in specific situations.

The main advantage of XML over JSON for prompting: the model can write reasoning or explanation text before the opening tag and structured content inside the tags. This avoids the conflict between "think step by step" and "return only JSON".

```
Analyze this support ticket. Think through what type of issue it is before deciding.
Then return your structured output inside the XML tags below.

Ticket: "Hi, I think I was double charged last month. My name is Sarah."

<analysis>
  <customer_name>Sarah</customer_name>
  <issue_type>billing</issue_type>
  <urgency>medium</urgency>
  <summary>Customer believes they were charged twice in the previous month.</summary>
</analysis>
```

The model can write its reasoning before the `<analysis>` tag, and you extract just the XML for parsing.

### Parsing XML Output

```python
import xml.etree.ElementTree as ET
import re

def parse_xml_response(response: str, root_tag: str) -> dict | None:
    # Extract the XML block
    pattern = f'<{root_tag}>([\s\S]*?)</{root_tag}>'
    match = re.search(pattern, response)
    if not match:
        return None

    xml_string = f"<{root_tag}>{match.group(1)}</{root_tag}>"

    try:
        root = ET.fromstring(xml_string)
        return {child.tag: child.text for child in root}
    except ET.ParseError:
        return None

result = parse_xml_response(llm_response, "analysis")
# {"customer_name": "Sarah", "issue_type": "billing", ...}
```

### When to Use XML vs JSON

Use JSON when:
- You are using a structured output API (OpenAI parse, Anthropic tool use)
- The output is consumed by code that already handles JSON
- The schema is straightforward and does not require reasoning before filling

Use XML when:
- You want chain-of-thought reasoning before the structured output
- The model needs to think before deciding on values
- You are working with Claude specifically, which follows XML-structured prompts particularly well

---

## 6. Validation Beyond Parsing

Successfully parsing the output into a Python object or dict is not the same as getting correct output. A parsed response can still contain:
- Wrong values for enum fields
- Strings where numbers were expected
- Missing required fields that the model set to an empty string instead of null
- Summaries that exceed your length limit
- Numbers outside an acceptable range

Always validate the parsed output against your business rules before using it.

```python
from pydantic import BaseModel, validator, Field
from typing import Literal

class CustomerIssue(BaseModel):
    customer_name: str | None
    issue_type: Literal["billing", "technical", "account", "other"]
    urgency: Literal["low", "medium", "high"]
    summary: str = Field(max_length=300)

    @validator("summary")
    def summary_not_empty(cls, v):
        if not v or not v.strip():
            raise ValueError("Summary cannot be empty")
        return v.strip()

try:
    validated = CustomerIssue(**parsed_dict)
except ValidationError as e:
    # Log the validation error and retry or fall back
    print(f"Validation failed: {e}")
```

Pydantic validators run on parse. If the model returns a value that violates your constraints, you get a clear error that you can retry with or log for debugging.

---

## 7. Key Takeaways

- Reliable structured output requires clear prompting plus defensive parsing plus ideally a schema-enforcing API.
- When prompting for JSON: say return JSON only, show the exact schema, show an example, and specify null handling.
- Parse failures happen in production. Write defensive parsers that handle code fences, preamble text, and trailing comments before falling back to a retry.
- Use the OpenAI parse API or Anthropic tool use with forced tool choice for the most reliable structured output in production.
- XML is a good alternative to JSON when you want the model to reason before filling in structured fields.
- Validate parsed output against your business rules using Pydantic or equivalent. Successful parsing does not mean correct output.

---

