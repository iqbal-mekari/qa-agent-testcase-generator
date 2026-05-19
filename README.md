# QA Agent UI Automation

Custom GitHub Copilot agent for comprehensive test case generation and maintenance.

## 🎯 Agent: qa-test-case-generator

Located in `.agent/` — automatically detected by GitHub Copilot for this repository.

### Capabilities

1. **Create Test Cases** — Generate exhaustive integration test cases from:
   - Jira tickets
   - PRD/Confluence documents
   - Figma designs
   - Free-text feature descriptions

2. **Regenerate Test Cases** — Update test cases based on:
   - Git diffs (branch comparisons)
   - Pull request changes
   - Existing test case CSVs

### How to Use

#### Create Test Cases

Ask Copilot to create test cases:

```
@copilot Create test cases for https://jurnal.atlassian.net/browse/QON-12345

@copilot Generate test cases from this PRD: https://jurnal.atlassian.net/wiki/...

@copilot Create integration test cases for a contact form with name, phone, and email fields
```

#### Regenerate Test Cases

Update test cases when code changes:

```
@copilot Regenerate test cases for branch feature/new-contact-form

@copilot Update test cases based on this PR: https://github.com/.../pull/123

@copilot Refresh test cases against /test-cases/QON_contact_test_cases.csv
```

### Output

Test cases are delivered to:

1. **Jira Comments** — Posted on the source ticket
2. **CSV Files** — Consolidated in `/test-cases/` directory

### Trigger Phrases

The agent activates on these keywords:

- `create test cases`, `generate test cases`
- `integration test`, `self-test`
- `test case from ticket`, `test case from PRD`
- `regenerate test cases`, `update test cases`
- `test cases from Figma`, `visual test cases`

### Skills

| Skill | Purpose |
|-------|---------|
| `create-test-cases` | Generate new test cases from requirements |
| `regenerate-test-cases` | Update test cases based on code changes |

See `.agent/skills/` for skill documentation.

## Structure

```
.agent/
├── AGENTS.md                              ← Agent definition
└── skills/
    ├── create-test-cases/
    │   └── SKILL.md                       ← Creation skill
    └── regenerate-test-cases/
        └── SKILL.md                       ← Regeneration skill
```

## Repository-Specific Configuration

This agent configuration is committed to this repository and will be:
- ✅ Detected automatically by GitHub Copilot
- ✅ Available only for contributors to this repo
- ✅ Inherited by all clones of this repository
- ✅ Updated via git commits

## Customization

To modify the agent or skills:

1. Edit files in `.agent/`
2. Commit changes to git
3. Push to repository
4. GitHub Copilot will automatically use the updated configuration

---

*Managed by: GitHub Copilot with custom agent skills*
