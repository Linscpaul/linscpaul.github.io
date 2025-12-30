---
title:  "[Enterprise Design] AI Governance & Engineering"
layout: post
---

# AI is Dangerous When Wrong but Confident: A Governance Framework

Financial institutions, among other sectors, operate in one of the most regulated environments in the world. In Singapore, AI implementation is specifically governed by the **MAS AI Risk Management (AIRG) Guidelines** , which was released in Nov 2025, building upon the 2023 GenAI risk framework, among others such as FEAT framework. 

In the high-stakes organization, the heart of `AI engineering` isn't just about "intelligence", it’s about **predictability, auditability, and risk control.** AI must be governed by strict engineering principles. Below is a breakdown of key implementation strategies to ensure compliance, for reference.

---

### 1. Standardization & Policy as Code
* **Automated Model Cards:** Automatically generate FEAT (Fairness, Ethics, Accountability, Transparency) reports before production.
* **Traceability:** Maintain full logs of training data, model versioning, evaluation metrics, PII masking as required by governance and risk.
* **Shadow Agents:** Deploy secondary agents in "shadow mode" to monitor for performance drift.

### 2. Adversarial Evaluation
* **Red Teaming:** Actively attempt to "trick" the agent into providing unethical output such as prohibited financial advice in the FI.
* **LLM-as-a-Judge:** Use a high-reasoning model to act as a compliance gatekeeper, grading the agent's responses before they reach the user.

### 3. Deterministic Fallbacks
* **Code over Calculation:** If a rule can be hard-coded, do not use LLM.
* **Logic Split:** Use the LLM to extract parameters but use deterministic code to return the final result.
(refer to example)

### 4. Structured Prompting
* **Strict Process Following:** High-stake tasks require precise, step-by-step instructions (SOPs) rather than open-ended autonomy.
* **(think out of box) Offline Optimization:** Use GenAI offline to help develop and refine these strict processes if they are too complex to map manually.
(refer to example)

### 5. Groundedness & Citations
* **Fact-Checking:** The agent must cite specific clauses from the document or provide a clear source reference for every decision.
* **Source Attribution:** This ensures the user can verify the information against the original document.

### 6. Chain-of-Thought (CoT) Traceability
* **Inner Monologue:** Capture the agent’s reasoning steps, the problems it encountered, and how it addressed them.
* **Audit Trail:** Use frameworks like **LangGraph** or **CrewAI** to provide a full "evidence log" of why a specific decision was made. Established ML tools and Cloud providers have native observability and explanability features.

### 7. Input/Output Guardrails
* **Safety Filters:** Implement filters to block hate speech, etc or customise filters and ensure compliance with the regulations such as **Financial Advisers Act**.
* **Compliance Checks:** Specifically prevent the agent from performing restricted actions, such as selling financial products, in the FI domain.

### 8. Self-Correction & Consistency
* **Convergence Checks:** The agent generate multiple outcomes simultaneously. If the outputs diverge (don't match), the system flags the interaction for human review.
* **Token Efficiency:** While it uses more tokens, this provides a "safety score" for the generated answer.

### 9. Human-in-the-Loop (HITL) 
This is important! and HITL should be customised based on the risk appetite. This is an example:
* **Materiality Assessment:** Tier the AI's autonomy based on the dollar impact and case complexity:
    * **Straight-Through:** Low-risk, automated answers.
    * **Junior Review:** Medium-risk cases.
    * **Senior Review:** High-risk or high-value cases.

---
*Note: Security concerns such as data leakage and prompt injection will be covered in a separate post.*


**An example on Deterministic Fallbacks**
```python
    # Bad: "Calculate interest for 1000 at 5%"
    # Good: 
    def calculate_interest(principal, rate):
        return principal * rate
```

**An example on Structured Prompting**

```python
def instr_vocabulary_expert_agent(teaching_material: str) -> str:
   return f"""\
Role: Pedagogical Content Specialist
Objective: Teach ALL the KEY_WORDs from the material one by one.


Instructions:
- TURN LOGIC: You are in a multi-turn conversation.
- Look at the VERY LAST message you sent.
- If you just taught "五谷丰登", then your NEXT message MUST teach the next word in the list.
- DO NOT repeat a word you have already explained.
- If you have explained ALL words in the list, only then output the token `END_SESSION`.
- You must go through EVERY word in this list:
{teaching_material}


Golden Example (Follow exactly for each Key_Words):
📘 Word: 五谷丰登
1. How to read? 👉 五谷丰登 👉 wǔ gǔ fēng dēng. Let’s read it slowly: 五 (wǔ) 谷 (gǔ) 丰 (fēng) 登 (dēng)
2. Very simple meaning. 👉 五谷丰登 = a lot of food + good harvest. In easy English: The crops grow very well. There is a lot of food.
3. Word by word (beginner level) 🌾
  - 五谷 (wǔ gǔ) 五 = many, 谷 = food / grain 👉 五谷 = food, grain (rice, wheat, corn)
  - 🌟 丰登 (fēng dēng) 丰 = a lot, 登 = harvest (collect food) 👉 丰登 = harvest is good
  Put them together: 五谷丰登 = food grows well, harvest is very good.
4. Picture in your head 🧠 Imagine this 👇 🌱 Small plants ☀️ Sun + 🌧️ rain 🌾 Big rice plants 🍚 A lot of food People are happy 😊, they say: 👉 五谷丰登！
5. Simple example sentences (read slowly) 👇
  1️⃣ 今年五谷丰登。 (This year, the harvest is good.)
  2️⃣ 农民希望五谷丰登。 (Farmers hope for a good harvest.)
  3️⃣ 五谷丰登，大家很开心。 (There is a good harvest. Everyone is happy.)
6. Very small practice (you try!) 🌟 Choose one:
  A. 五谷丰登，我很饿。
  B. 五谷丰登，粮食很多。 ✅
  C. 五谷丰登，下雨了。
🌟 Say it with me: Say this sentence out loud 👇 👉 今年五谷丰登。
7. Remember like this 🧠 🌾 Food 🌾 A lot 😊 Happy people 👉 五谷丰登


Constraints:
- Never skip a word.
- Never repeat a word you have already finished in this session.
- Only use `END_SESSION` after the final word is mastered.
"""
```
