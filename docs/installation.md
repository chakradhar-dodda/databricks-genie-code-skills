# Installation Guide

This guide will help you set up the Databricks Skills repository for use with Genie Code in your Databricks workspace.

## Quick Installation

### Step 1: Access Your Databricks Workspace
Navigate to your Databricks workspace and open a terminal or notebook.

### Step 2: Clone or Copy the Repository

**Option A: Git Clone (Recommended)**
```bash
cd /Workspace/Users/your.email@company.com/
git clone <your-repo-url> databricks-genie-code-skills
```

**Option B: Manual Copy**
If you don't have Git access, manually copy the repository files to your workspace.

### Step 3: Set Up User Instructions
Copy the user instructions template to your home directory:

```bash
cp databricks-genie-code-skills/examples/user-instructions.md ~/.assistant_instructions.md
```

Edit the file to customize for your preferences:
```bash
# Start your message with # in Genie Code chat to quickly add instructions
# Or edit the file directly
```

### Step 4: Install Skills
Link or copy skills to your Genie Code skills folder:

**Option A: Copy All Skills**
```bash
mkdir -p ~/.assistant/skills
cp -r databricks-genie-code-skills/skills/* ~/.assistant/skills/
```

**Option B: Copy Specific Skills**
```bash
mkdir -p ~/.assistant/skills
cp -r databricks-genie-code-skills/skills/data-quality-checks ~/.assistant/skills/
cp -r databricks-genie-code-skills/skills/delta-lake-optimization ~/.assistant/skills/
cp -r databricks-genie-code-skills/skills/incremental-processing ~/.assistant/skills/
# Add more as needed
```

### Step 5: Verify Installation

Check that skills are accessible:
```bash
ls -la ~/.assistant/skills/
```

You should see directories like:
* data-quality-checks
* delta-lake-optimization
* incremental-processing
* unity-catalog-governance
* And more...

---

## Using Skills with Genie Code

### Activating Skills
Once skills are in `~/.assistant/skills/`, Genie Code automatically knows about them.

### Testing a Skill
In Genie Code, ask questions related to the skills:
* "Help me optimize this Delta table"
* "How do I implement data quality checks?"
* "Show me incremental processing patterns"

Genie Code will reference the relevant skills to provide targeted guidance.

---

## Workspace-Level Installation (Admins Only)

If you're a workspace admin, you can install skills workspace-wide:

### Create Workspace Skills Directory
```bash
# Create central skills location
mkdir -p /Workspace/Shared/.databricks/skills

# Copy skills
cp -r databricks-genie-code-skills/skills/* /Workspace/Shared/.databricks/skills/
```

### Set Up Workspace Instructions
Create workspace instructions file (admin-configured):
```bash
cp databricks-genie-code-skills/examples/workspace-instructions.md \
   /Workspace/Shared/.databricks/workspace_instructions.md
```

Edit this file to reflect your organization's standards.

### Notify Users
Inform users that workspace skills are available and how to access them.

---

## Customization

### Customize User Instructions
Edit `~/.assistant_instructions.md`:
```markdown
# Add your project context
## Current Projects
* Project name, catalogs, tables, business context

# Add your preferences
* Coding style
* Naming conventions
* Team standards
```

### Create Custom Skills
1. Create a new skill directory:
   ```bash
   mkdir -p ~/.assistant/skills/my-custom-skill
   ```

2. Create SKILL.md:
   ```bash
   cat > ~/.assistant/skills/my-custom-skill/SKILL.md << 'EOF'
   # My Custom Skill
   
   ## Purpose
   What this skill teaches...
   
   ## When to Use
   Situations where this applies...
   
   ## Examples
   Code examples...
   EOF
   ```

3. Test with Genie Code

---

## Update and Maintenance

### Updating Skills
When new skills are added or existing ones updated:

```bash
cd databricks-genie-code-skills
git pull  # If using Git

# Re-copy updated skills
cp -r skills/* ~/.assistant/skills/
```

### Backing Up Your Configuration
Back up your personal instructions:
```bash
cp ~/.assistant_instructions.md databricks-genie-code-skills/backups/my_instructions_$(date +%Y%m%d).md
```

---

## Troubleshooting

### Skills Not Working
1. **Verify installation path**:
   ```bash
   ls -la ~/.assistant/skills/
   ```

2. **Check file permissions**:
   ```bash
   chmod -R 755 ~/.assistant/skills/
   ```

3. **Validate SKILL.md format**:
   * Ensure proper markdown formatting
   * Check for syntax errors

### Genie Code Not Using Skills
* Skills must be in `~/.assistant/skills/` (not `.assistant/skill`)
* File must be named `SKILL.md` (case-sensitive)
* Restart Genie Code session if needed

### Permission Issues
If you can't write to `~/.assistant/`:
```bash
mkdir -p ~/.assistant/skills
chmod 755 ~/.assistant
chmod 755 ~/.assistant/skills
```

---

## Next Steps

1. **Read the Skills**: Browse through `databricks-genie-code-skills/skills/` to understand what's available
2. **Customize Instructions**: Edit your `~/.assistant_instructions.md`
3. **Test with Genie Code**: Ask questions related to the skills
4. **Share Learnings**: Contribute back improvements and new skills

---

## Support

* **Issues**: Check the repository issues page
* **Questions**: Ask in your team's Slack channel
* **Contributions**: Submit pull requests with improvements

---

## Uninstallation

To remove skills:
```bash
rm -rf ~/.assistant/skills/*
rm ~/.assistant_instructions.md
```

---

*For more information, see the main [README.md](../README.md)*
