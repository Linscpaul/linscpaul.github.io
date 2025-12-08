---
title:  "OpenAI Agents SDK Framework"
layout: post
---
## Building an Automated Sales Outreach App

### Key Features Of This App 

###### Agentic Workflow:

This is a system where different AI "agents" work together like a team.

###### Multiple Brains:

This systems use different LLM models from OpenAI, Google, and others to make our agents smarter.

###### Lots of Tools:

The agents would use various tools, some powered by AI and others that are regular computer functions, like sending emails.

###### Handoffs:

The main agent can pass tasks to other specialized agents. It's like a manager delegating work to team members.

###### Guardrails:
We'll set up rules to control what our agents can say and do, making sure they don't make mistakes or say the wrong things.

### How It Works 

#### This program creates sales emails and sends them automatically. Here's the breakdown:

**Planning Agent:** 
This is the boss agent. It tells three "content writing" agents to create different versions of a sales email.
**Choosing the Best Email:** 
The planning agent looks at the three versions and picks the one it thinks is most likely to get a good response.
**Handoff to Email Manager:** 
The planning agent then hands off the chosen email to another agent called the "Email Manager."
**Email Manager's Job:** 
The Email Manager turns the email into a fancy HTML format, creates a catchy subject line, and sends it off to the customer.


## Blockquotes

### Single line

> My mom always said life was like a box of chocolates. You never know what you're gonna get.

### Multiline

> What do you get when you cross an insomniac, an unwilling agnostic and a dyslexic?
>
> You get someone who stays up all night torturing himself mentally over the question of whether or not there's a dog.
>
> – _Hal Incandenza_

## Horizontal Rule

---

## Table

| Title 1          | Title 2          | Title 3         | Title 4         |
|------------------|------------------|-----------------|-----------------|
| First entry      | Second entry     | Third entry     | Fourth entry    |
| Fifth entry      | Sixth entry      | Seventh entry   | Eight entry     |
| Ninth entry      | Tenth entry      | Eleventh entry  | Twelfth entry   |
| Thirteenth entry | Fourteenth entry | Fifteenth entry | Sixteenth entry |

## Code

Source code can be included by fencing the code with three backticks. Syntax highlighting works automatically when specifying the language after the backticks.

````
```python
from dotenv import load_dotenv
from openai import AsyncOpenAI
from agents import Agent, Runner, trace, function_tool, OpenAIChatCompletionsModel, input_guardrail, GuardrailFunctionOutput
from typing import Dict
import sendgrid
import os
from sendgrid.helpers.mail import Mail, Email, To, Content
from pydantic import BaseModel
```
````

This would be rendered as:

```python
load_dotenv(override=True)
openai_api_key = os.getenv('OPENAI_API_KEY')
google_api_key = os.getenv('GOOGLE_API_KEY')
deepseek_api_key = os.getenv('DEEPSEEK_API_KEY')


# print if keys are found in the .env file
if openai_api_key:
   print(f"OpenAI API Key exists and begins {openai_api_key[:8]}")
else:
   print("OpenAI API Key not set")


if google_api_key:
   print(f"Google API Key exists and begins {google_api_key[:2]}")
else:
   print("Google API Key not set (and this is optional)")


if deepseek_api_key:
   print(f"DeepSeek API Key exists and begins {deepseek_api_key[:3]}")
else:
   print("DeepSeek API Key not set (and this is optional)")


GEMINI_BASE_URL = "https://generativelanguage.googleapis.com/v1beta/openai/"
DEEPSEEK_BASE_URL = "https://api.deepseek.com/v1"
```

## Lists

### Unordered

* First item
* Second item
* Third item
    * First nested item
    * Second nested item

### Ordered

1. First item
2. Second item
3. Third item
    1. First nested item
    2. Second nested item

