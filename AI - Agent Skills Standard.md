[SOURCE](https://dev.to/loc_carrre_0d798813c662/agent-skills-explained-what-they-are-what-they-arent-and-how-to-use-them-bf9)

## <u>Introduction</u>

Agent Skills is an open standard for packaging reusable AI agent capabilities as plain Markdown files. A skill is a folder with a `SKILL.md` file containing metadata and instructions. When an agent activates a skill, it loads the ==instructions== on demand, follows them as an explicit workflow, and produces structured, repeatable output. Skills solve the problem of ==bloated== prompts and ==inconsistent== behaviour.

## <u>Structure</u>

The `SKILL.md` file has two parts:
1. **YAML frontmatter** with metadata (name, description, version)
2. **Markdown body** with the actual instructions

```
For more complex skills, the directory can include additional folders:

code-review/
  SKILL.md                # Required: instructions + metadata
  scripts/                # Optional: executable code
    lint-check.sh
  references/             # Optional: documentation
    coding-standards.md
  assets/                 # Optional: templates, data files
    review-template.json
  examples/               # Optional: sample inputs/outputs
    sample-review.md
```

The agent only ==reads== these when it actually ==needs== them, not everything at startup. We can also only write a single `SKILL.md` file for a simple implementation.

## <u>YAML Frontmatter Reference</u>

```
name: explain
description: Explains any topic in plain language. Type a word or phrase and get a clear, jargon-free explanation.
metadata:
  version: "1.0"
```

The [Agent Skills specification](https://agentskills.io/specification) defines these frontmatter fields:

![[Attachments/YAML-Frontmatter-Reference.png]]

Note that `version` is not a top-level spec field. Use the `metadata` map for it:
```
metadata:
  author: your-org
  version: "1.0"
```


## <u>What Agent Skills are NOT</u>

- **Skills are not prompts**
>Skills ==load dynamically== based on context. Prompts are always present.

- **Skills are not deterministic**
>The same skill can produce slightly different outputs. If you need ==guaranteed structure== combine a skill with structured ==output constraints== (grammar, JSON schema) or use tool calls for the critical parts.

- **Skills are not a replacement for MCP**
>MCP provides secure connectivity to external systems: databases, APIs, file systems. Skills provide procedural knowledge for _using_ those systems. A "Database Query" skill might instruct the agent on how to write safe SQL and handle edge cases, while an MCP server provides the actual database connection. Different layers, same stack.

## <u>How Progressive Disclosure Works</u>
The key architectural innovation of Agent Skills is **progressive disclosure**: a three-tier loading strategy that keeps context efficient.

![[AI_Progressive_Disclosure_tokens_efficiency.png]]

**At startup**, the agent loads only names and descriptions of all available skills. If you have 20 skills, that is roughly 1,000 tokens of metadata. The agent knows _what_ it can do, but carries none of the detailed instructions.

**When a skill is activated**, the full SKILL.md body is ==loaded into context==. The agent now has the detailed instructions, output format, and rules for that specific task.

This means you can give an agent access to a large library of capabilities without paying the token cost for all of them at once. The context window stays clean, the agent stays focused, and you only pay for what you use.

## <u>What Problems Do Agent Skills Solve in Practice?</u>

Without skills, a multi-capability agent needs all instructions loaded simultaneously. With 10 workflows averaging 500 tokens each, that is 5,000 tokens of permanent context overhead. With skills, the overhead is ~500 tokens for the catalog plus ~500 tokens for whichever skill is currently active. A ==10x reduction==.
### **Consistent, repeatable output**
A skill specifies an exact output format, a set of rules, and optionally examples. Every time the agent runs that skill, it follows the same structure. This is the difference between "review this code" (unpredictable) and "review this code using the code-review skill" (==consistent format, every time==).

### **Modularity**
Add a new capability? Drop a folder. Remove one? Delete the folder. Update a workflow? Edit the Markdown. ==No code changes, no redeployment, no risk of breaking unrelated features==.

### **Version control and governance**
Skills are plain text files. They live in your Git repository. You can:
- Review changes in pull requests
- Track the history of every instruction change
- Roll back to a previous version if a new one underperforms
- Require approval before behaviour changes go live

### **Team collaboration**
Because skills are Markdown, anyone can write or improve them. A domain expert (lawyer, compliance officer, product manager) can author a skill without touching code. Developers review and deploy it. This separates "==what the AI should do==" (domain knowledge) from "==how the AI works==" (infrastructure).
>Both teams contribute to what they know best.

## <u>Why Agent Skills Make Even More Sense for Local AI Models</u>

### **Smaller models need more guidance**
Cloud models are pre-trained. A 4B or 8B parameter model running on your GPU does not have the same capacity to guess your intent. It needs explicit instructions: ==what format to follow, what constraints to respect, what to include and what to skip==. `SKILL.md` compensate for a smaller model's weaker instruction-following by giving it the specific, structured guidance it needs.

### **Context windows are tighter**
Cloud models now offer 128K to 1M token context windows. Local models typically run with 4K to 32K contexts, sometimes up to 128K but at a real performance cost (memory, speed). Every token matters more.

This is where progressive disclosure goes from "nice optimisation" to "architectural necessity." With a local model, you simply cannot afford to load all possible instructions at once. You need the skill system: load only metadata at startup (~50 tokens per skill), activate the relevant skill on demand (~500 to 2,000 tokens), keep the rest of the context free for the actual user data.

### **Every token costs compute, not dollars**
With a cloud API, the cost of extra context tokens is measured in money. With local inference, it is measured in latency and memory. Stuffing a 5,000-token system prompt into every request to a local model directly impacts response time, especially on consumer hardware. Skills let you keep prompts lean for the common case and only expand them when the specific expertise is actually needed.

### **No network dependency**
Skills are local files. The model is local. The entire skill activation pipeline (read file, inject into prompt, generate response) happens in-process with zero network calls. This makes the architecture simpler, faster, and fully offline-capable. For edge deployments, air-gapped environments, or embedded applications, this matters. You ship the model and the skills folder together, and the system works with no external dependencies.

### **Privacy and control**
When skills run locally, the instructions themselves never leave your machine. If you have a skill encoding proprietary compliance rules, internal procedures, or sensitive domain knowledge, that content stays in your process memory. It is never sent to a third-party API. This is a significant advantage for healthcare, finance, legal, defence, and any industry with data residency requirements.

In short: skills make large cloud models more organised. They make small local models actually capable. The smaller the model, the more it benefits from well-structured skill instructions.

## <u>Before and After: What a Skill Actually Changes</u>

![[Attachments/Screenshot 2026-10-27 at 10.00.56 AM.png]]

## <u>Best Practices for Writing Great Skills</u>

### **Naming**
- Use lowercase with hyphens: `code-review`, `email-writer`, `weekly-report`
- Keep it short and descriptive (max 64 characters)
- Avoid generic names like `helper` or `assistant`

### **Description**
- Write it as a one-sentence action statement: "Reviews code for bugs, security issues, and style violations"
- Include keywords that help with automatic matching
- Keep it under 1024 characters

### **Instructions**
- **Put the most important constraints first.** LLMs pay more attention to the beginning.
- **Specify the exact output format.** Show a template with placeholders. The more specific, the more consistent.
- **Write explicit rules.** Number them. State what the model should and should not do.
- **Include at least one example.** Show a sample input and the expected output. This is the single most effective technique for consistent behaviour.
- **Keep it concise.** Move lengthy reference material to `references/` and load it on demand. The instructions themselves should fit in ~500 to 2,000 tokens.
## <u>Structure Template</u>

```
---
name: your-skill-name
description: One sentence explaining when to use this skill.
metadata:
  version: "1.0"
---

# Role Title

One paragraph defining the agent's role and what it does.

## Output Format

<exact template with placeholders>

## Rules

1. **Rule one.** Short explanation.
2. **Rule two.** Short explanation.
3. ...

## Example

**Input:** <sample input>

**Output:** <sample output following the format above>
```

### **The Skill Authoring Checklist**

Before shipping a skill, verify:

- [ ] Name is lowercase, hyphenated, and descriptive
- [ ] Description clearly states when to use the skill
- [ ] Version is set (start with `1.0.0`)
- [ ] Output format is explicitly defined with a template
- [ ] At least 3 clear rules constrain behavior
- [ ] At least 1 example input/output is included
- [ ] Instructions fit within ~2,000 tokens (move long references to `references/`)
- [ ] Skill validates without errors (`AgentSkill.Validate()`)
- [ ] Tested with at least 2 different models

## <u>Security Considerations</u>

Agent Skills are powerful, and that power requires care. Cisco's AI Defense team [found](https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare) that a significant number of publicly shared skills contained vulnerabilities, including prompt injection and credential exfiltration.

**Best practices for skill security:**

- **Review all skills before deployment.** Treat SKILL.md files like code: review them in pull requests.
- **Be cautious with `scripts/`.** Scripts execute on the host machine. Only include scripts from trusted sources. Consider sandboxing execution.
- **Use the `allowed-tools` field.** Explicitly declare which system tools a skill can invoke. This helps runtime environments enforce boundaries.
- **Never put secrets in skills.** No API keys, passwords, or tokens in SKILL.md files or their resources.
- **Validate inputs.** If a skill processes user-provided data, ensure your agent validates that data before acting on it.
- **Pin versions.** When loading remote skills, pin to specific versions or commit hashes rather than tracking `main`.

## <u>Industry Adoption</u>

![[Attachments/Screenshot 2026-10-27 at 10.08.40 AM.png]]

## <u>Conclusion</u>

Agent Skills solve a real, specific problem: how to give AI agents specialized, reusable expertise without overwhelming their context windows. The solution is elegant in its simplicity. A skill is a Markdown file in a folder. The agent loads it on demand. The output becomes consistent.

Here is what to remember:

1. **A skill is a folder with a SKILL.md file.** YAML frontmatter for metadata, Markdown body for instructions.
2. **Progressive disclosure keeps context efficient.** Metadata at startup, instructions on activation, resources on execution.
3. **Skills are not tools, not prompts, not agents.** They are portable knowledge modules that complement all three.
4. **The standard is open and widely adopted.** OpenAI, Microsoft, Anthropic, and a growing number of platforms support it.
5. **LM-Kit.NET provides a complete implementation.** Registry, activator, hot reload, remote loading, validation, tool integration.
