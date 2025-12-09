---
title:  "[Agentic Framework] OpenAI Agents SDK - Automated Sales App"
layout: post
---
## Building an Automated Sales Outreach App

### Key Features of This App 

**Agentic Workflow:**
This is a system where different AI "agents" work together like a team.

**Multiple Brains:**
This systems use different LLM models from OpenAI, Google, and others to make our agents smarter.

**Lots of Tools:**
The agents would use various tools, some powered by AI and others that are regular computer functions, like sending emails.

**Handoffs:**
The main agent can pass tasks to other specialized agents. It's like a manager delegating work to team members.

**Guardrails:**
We'll set up rules to control what our agents can say and do, making sure they don't make mistakes or say the wrong things.

### How it Works 

#### This program creates sales emails and sends them automatically. Here's the breakdown:

**Planning Agent:** 
This is the boss agent. It tells three "content writing" agents to create different versions of a sales email.

**Choosing the Best Email:** 
The planning agent looks at the three versions and picks the one it thinks is most likely to get a good response.

**Handoff to Email Manager:** 
The planning agent then hands off the chosen email to another agent called the "Email Manager."

**Email Manager's Job:** 
The Email Manager turns the email into a fancy HTML format, creates a catchy subject line, and sends it off to the customer.

**Important Note:** 
The code examples below are like building blocks. You need to create the tools before you create the agents that use them. So, we'll start with the tools and work our way up to the main agent.

---

## Part 1: Setting Up the Environment

Before we can start building our app, we need to set up our computer with the right tools and keys

### Module 1: Importing Libraries

First, we need to import some libraries that will help us build our app. Think of it like getting all the necessary LEGO bricks before starting to build a LEGO set.

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

### Module 2: Configuring the .env File

A .env file is like a secret vault where you store your special keys. These keys allow our app to access services like OpenAI, Google, and SendGrid (for sending emails).

Why .env? We don't want to share our secret keys with everyone, so we store them in a separate file that's not uploaded to the internet.

How to Set Up: Create a file named .env in the same folder as your project. Inside, put your keys like this: (if you have any doubts, please ask ChatGPT for step-by-step guide)

```python
# this is configured in the .env file saved in the project folder. Please do not execute the codes in the .ipynb notebook.
OPENAI_API_KEY=your api key
GOOGLE_API_KEY=your api key
DEEPSEEK_API_KEY=your deepseek api key
SENDGRID_API_KEY=your sendgrid api key
```

Important: Don't share your .env file with anyone!

Now, let's load the keys from the .env file into our program:

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

---

## Part 2: The Sales Manager Agent

The Sales Manager is the brain of our operation. It decides which sales email is the best and tells the Email Manager to send it.

### Module 1: Creating the Sales Manager Agent with Input Control

First, we need to give the Sales Manager some instructions:

```python
sales_manager_instructions = """
You are a Sales Manager at ComplAI. Your goal is to find the single best cold sales email using the sales_agent tools.
Follow these steps carefully:
1. Generate Drafts: Use all three sales_agent tools to generate three different email drafts. Do not proceed until all three drafts are ready.
2. Evaluate and Select: Review the drafts and choose the single best email using your judgment of which one is most effective.
You can use the tools multiple times if you're not satisfied with the results from the first try.
3. Handoff for Sending: Pass ONLY the winning email draft to the 'Email Manager' agent. The Email Manager will take care of formatting and sending.
Crucial Rules:
- You must use the sales agent tools to generate the drafts — do not write them yourself.
- You must hand off exactly ONE email to the Email Manager — never more than one.
"""
```

### Guardrails: Keeping Things Safe

We want to make sure people don't put personal names in the emails, this is to protect privacy. So, we'll use a "guardrail" to check for names.

The OpenAI Agents SDK framework allows you to create agents using the following syntax: Agent(name, instructions, model, ...). We will use this syntax repeatedly throughout this application, as we will be creating several agents.

```python
class NameCheckOutput(BaseModel):
   is_name_in_message: bool
   name: str

guardrail_agent = Agent(
   name="Name check",
   instructions="Check if the user is including someone's personal name in what they want you to do.",
   output_type=NameCheckOutput,
   model="gpt-4o-mini"
)
```

```python
@input_guardrail
async def guardrail_against_name(ctx, agent, message):
   result = await Runner.run(guardrail_agent, message, context=ctx.context)
   is_name_in_message = result.final_output.is_name_in_message
   return GuardrailFunctionOutput(output_info={"found_name": result.final_output},tripwire_triggered=is_name_in_message)
```

We will create the sales_manager agent using the OpenAI Agents SDK framework. This agent can call tools, which we will define in the next step. A guardrail is also implemented to prevent users from including personal names in the instructions or prompts given to the sales_manager agent.

```python
sales_manager = Agent(
   name="Sales Manager",
   instructions=sales_manager_instructions,
   tools=tools, #find the tools creation in Part 3 below
   handoffs=[emailer_agent], #find the handoff creation in Part 4 below
   model="gpt-4o-mini",
   input_guardrails=[guardrail_against_name]
   )
```

---

## Part 3: Creating the Sales Agent Tools

These tools are like different writers with unique styles. They'll each create a sales email draft.

### Module 1: Content-Writing Agents

We are essentially creating three content-writing agents, each with a distinct writing style. The following instructions or prompts will guide each agent's behavior, and you are encouraged to experiment with different prompts to explore various outcomes. 

```python
instructions1 = "You are a sales agent working for ComplAI, \
a company that provides a SaaS tool for ensuring SOC2 compliance and preparing for audits, powered by AI. \
You write professional, serious cold emails."

instructions2 = "You are a humorous, engaging sales agent working for ComplAI, \
a company that provides a SaaS tool for ensuring SOC2 compliance and preparing for audits, powered by AI. \
You write witty, engaging cold emails that are likely to get a response."

instructions3 = "You are a busy sales agent working for ComplAI, \
a company that provides a SaaS tool for ensuring SOC2 compliance and preparing for audits, powered by AI. \
You write concise, to the point cold emails."
```

To create an agent, use the Agent() syntax from the OpenAI Agents SDK. This requires you to provide the agent's name, instructions (or prompt), and the LLM model you want it to use.

```python
sales_agent1 = Agent(name="DeepSeek Sales Agent", instructions=instructions1, model=deepseek_model)
sales_agent2 =  Agent(name="Gemini Sales Agent", instructions=instructions2, model=gemini_model)
sales_agent3  = Agent(name="GPT Sales Agent", instructions=instructions3, model=gpt-4o-mini)
```

To convert the three content-writing agents into tools, use the .as_tool() syntax provided by the OpenAI Agents SDK. You also need to provide a description explaining the purpose of each tool.

```python
description = "Write a cold sales email"
tool1 = sales_agent1.as_tool(tool_name="sales_agent1", tool_description=description)
tool2 = sales_agent2.as_tool(tool_name="sales_agent2", tool_description=description)
tool3 = sales_agent3.as_tool(tool_name="sales_agent3", tool_description=description)
```

Now, let's create the tools package, which will be passed to the sales_manager agent.

```python
tools = [tool1, tool2, tool3] #this definition is passed to the sales-manager agent
```

---

## Part 4: Creating the Handoff - The Email Manager Agent

The Email Manager takes the best email and sends it out. It uses tools to create a subject, convert the email to HTML, and then send it.

### Module 1: Creating the Email Manager Agent

When the sales_manager agent triggers the handoff to the emailer_agent, the emailer_agent follows its instructions and uses the email_tools, which include the subject_tool, html_tool, and send_html_email.

```python
instructions ="You are an email formatter and sender. You receive the body of an email to be sent. \
You first use the subject_writer tool to write a subject for the email, then use the html_converter tool to convert the body to HTML. \
Finally, you use the send_html_email tool to send the email with the subject and HTML body."

emailer_agent = Agent(
   name="Email Manager",
   instructions=instructions,
   tools=email_tools,
   model="gpt-4o-mini",
   handoff_description="Convert an email to HTML and send it")

handoffs = [emailer_agent] #this definition is passed to the sales-manager agent
```

### Module 2: Creating the Subject Writer and HTML Converter Tools

Both tools are essentially agents powered by LLMs. These agents are created using the Agent(name, instructions, model) syntax from the OpenAI Agents SDK, and then converted into tools using the .as_tool(tool_name, tool_description) syntax, also from the OpenAI Agents SDK.

```python
subject_instructions = "You can write a subject for a cold sales email. \
You are given a message and you need to write a subject for an email that is likely to get a response."

html_instructions = "You can convert a text email body to an HTML email body. \
You are given a text email body which might have some markdown \
and you need to convert it to an HTML email body with simple, clear, compelling layout and design."

subject_writer = Agent(name="Email subject writer", instructions=subject_instructions, model="gpt-4o-mini")
subject_tool = subject_writer.as_tool(tool_name="subject_writer", tool_description="Write a subject for a cold sales email")

html_converter = Agent(name="HTML email body converter", instructions=html_instructions, model="gpt-4o-mini")
html_tool = html_converter.as_tool(tool_name="html_converter",tool_description="Convert a text email body to an HTML email body")
```

### Module 3: Creating the Send Email Tool

Let's write the send_html_email function and convert it into a tool using the @function_tool syntax from the OpenAI Agents SDK. 

```python
@function_tool
def send_html_email(subject: str, html_body: str) -> Dict[str, str]:
   """ Send out an email with the given subject and HTML body to all sales prospects """
   sg = sendgrid.SendGridAPIClient(api_key=os.environ.get('SENDGRID_API_KEY'))
   from_email = Email("lsanchuan.m21@gmail.com")  # Change to your verified sender
   to_email = To("linsanchuan@msn.com")  # Change to your recipient
   content = Content("text/html", html_body)
   mail = Mail(from_email, to_email, subject, content).get()
   sg.client.mail.send.post(request_body=mail)
   return {"status": "success"}
```

These three tools are now packaged together as email_tools, which will then be passed to the handoff emailer_agent.

```python
email_tools = [subject_tool, html_tool, send_html_email]
```

---

Now, you should have a good understanding of how to use the OpenAI Agents SDK to create an automated app with agents!
