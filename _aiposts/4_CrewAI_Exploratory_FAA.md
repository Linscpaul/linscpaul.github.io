---
title:  "CrewAI Framework Exploratory - Financial Analyst Agent"
layout: post
---

## Building a Financial Analyst Agent App

### CrewAI offers several powerful tools:

*   **CrewAI Enterprise:** A multi-agent platform designed for deploying, running, and monitoring agentic AI solutions at scale.
*   **CrewAI UI Studio:** A no-code/low-code product that simplifies the creation of multi-agent solutions through an intuitive visual interface.
*   **CrewAI Open-Source Framework:** Orchestrate high-performing AI agents with ease and flexibility using this versatile framework.

### CrewAI Open-Source Framework: Two Approaches

The CrewAI open-source framework provides two distinct approaches to multi-agent orchestration:

*   **CrewAI Crews:** Ideal for autonomous problem-solving, creative collaboration, or exploratory tasks where a high degree of uncertainty is involved. This approach leverages AI teams of agents with diverse roles to achieve complex goals.
*   **CrewAI Flows:** Suited for structured automation by dividing complex tasks into precise workflows. Choose this when you require deterministic outcomes, auditability, or precise control over the execution process.

Our focus is on the autonomous capabilities of CrewAI, where we empower different LLMs to explore and solve problems agentically, making independent decisions along the way.

### Core Concepts of CrewAI

*   **Agent:** An autonomous unit equipped with an LLM, a defined role, a specific goal, a detailed backstory, memory, and a suite of tools.
*   **Task:** A specific assignment to be carried out by an agent, characterized by a clear description and expected output.
*   **Crew:** A team of Agents and Tasks, organized either sequentially (tasks run in a predefined order) or hierarchically (a Manager LLM assigns tasks).

CrewAI stands out as a lightweight and opinionated framework, offering a streamlined alternative to the OpenAI Agents SDK.

### YAML Configuration

Agents and Tasks can be dynamically created using code or configured via YAML files, where you can define backstories, descriptions, expected outputs, and more.

**Example:**

```yaml
Researcher:
  Role: >
    Senior Financial Researcher
  Goal: >
    Research companies, news, and potential
  Backstory: >
    You are a seasoned financial researcher with a talent for finding the most relevant information
  llm: openai/gpt-4o-mini
```

The `crew.py` module contains the Crew definition, utilizing decorators such as `@agent`, `@task`, and `@crew` to define the structure and behavior of the crew.

### LLMs in CrewAI
CrewAI offers seamless integration with almost any LLM through the LiteLLM library. Simply set your API keys in the `.env` file to get started.

**Example:**
```python
llm = LLM(model = “openai/gpt-4o-mini”
llm = LLM(
Model = “openrouter/deepseek/deepseek-r1”,
Base_url = “https://openrouter.ai/api/v1”,
api_key=OPENROUTER_API_KEY
)
```

## 5 Steps to Implement a CrewAI Project - Financial Researcher

### Step 1: Create a UV Project for CrewAI

CrewAI is readily installed in UV. To create a new project, use the following commands:
```bash
uv tool install crewai
crewai create crew my_crew  # Creates a project folder
crewai run
```

### Step 2: Configure YAML Files for Agents and Tasks
**src -> config -> agents.yaml**

This file is central to the CrewAI framework, allowing you to define and configure your agents and their models.

```yaml
researcher:
 role: >
   Senior Financial Researcher for {company}
 goal: >
   Research the company, news and potential for {company}
 backstory: >
   You're a seasoned financial researcher with a talent for finding
  the most relevant information about {company}.
  Known for your ability to find the most relevant
  information and present it in a clear and concise manner.
 llm: openai/gpt-4o-mini

analyst:
 role: >
   Market Analyst and Report writer focused on {company}
 goal: >
   Analyze company {company} and create a comprehensive, well-structured report
  that presents insights in a clear and engaging way
 backstory: >
   You're a meticulous, skilled analyst with a background in financial analysis
  and company research. You have a talent for identifying patterns and extracting
  meaningful insights from research data, then communicating
  those insights through well crafted reports.
 llm: openai/gpt-4o-mini
```

**src -> config -> tasks.yaml**

Here, you define the tasks that each agent will perform. Note that task names and agent names must be distinct to avoid conflicts. The `context` parameter is used to pass the output from one task to another, as demonstrated below:

```yaml
research_task:
 description: >
   Conduct thorough research on company {company}. Focus on:
  1. Current company status and health
  2. Historical company performance
  3. Major challenges and opportunities
  4. Recent news and events
  5. Future outlook and potential developments

  Make sure to organize your findings in a structured format with clear sections.
 expected_output: >
   A comprehensive research document with well-organized sections covering
  all the requested aspects of {company}. Include specific facts, figures,
  and examples where relevant.
 agent: researcher

analysis_task:
 description: >
   Analyze the research findings and create a comprehensive report on {company}.
  Your report should:
  1. Begin with an executive summary
  2. Include all key information from the research
  3. Provide insightful analysis of trends and patterns
  4. Offer a market outlook for company, noting that this should not be used for trading decisions
  5. Be formatted in a professional, easy-to-read style with clear headings
 expected_output: >
   A polished, professional report on {company} that presents the research
  findings with added analysis and insights. The report should be well-structured
  with an executive summary, main sections, and conclusion.
 agent: analyst
 context:
   - research_task
 output_file: output/report.md
```

### Step 3: Step 3: Complete the crew.py Module

**src -> crew.py** 

This module is where you bring your agents and tasks together to form a cohesive crew. To utilize tools like `SerperDevTool`, import the `library` and add `tools=[SerperDevTool()]` to the Agent that will use the tool. Remember to include your `SERPER_API_KEY` in the `.env` file.

```python
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task
from crewai.agents.agent_builder.base_agent import BaseAgent
from typing import List
​​from crewai_tools import SerperDevTool

@CrewBase
class ResearchCrew():
   """Research crew for comprehensive topic analysis and reporting"""

   agents: List[BaseAgent] #this is auto-configured by crewai
   tasks: List[Task] #this is auto-configured by crewai

   @agent
   def researcher(self) -> Agent:
       return Agent(config=self.agents_config['researcher'], verbose=True, tools=[SerperDevTool()])

   @agent
   def analyst(self) -> Agent:
       return Agent(config=self.agents_config['analyst'],verbose=True)

   @task
   def research_task(self) -> Task:
       return Task(config=self.tasks_config['research_task'])

   @task
   def analysis_task(self) -> Task:
       return Task(config=self.tasks_config['analysis_task'])

   @crew
   def crew(self) -> Crew:
       """Creates the research crew"""
       return Crew(
           agents=self.agents,
           tasks=self.tasks,
           process=Process.sequential,
           verbose=True,
       )
```

### Step 4: Update main.py and Run

**src -> main.py**

Update the `main.py` file to define the inputs and execute the crew.

```python
import os
from financial_researcher.crew import ResearchCrew

os.makedirs('output', exist_ok=True)

def run():
   """
   Run the research crew.
   """
   inputs = {
       'company': 'Apple'
   }

   result = ResearchCrew().crew().kickoff(inputs=inputs)
   print("\n\n=== FINAL REPORT ===\n\n")
   print(result.raw)

   print("\n\nReport has been saved to output/report.md")

if __name__ == "__main__":
   run()
```
 
### Execute main.py
Open your terminal, navigate to the project directory, and run `main.py` using the command:

```bash
crewai run
```
