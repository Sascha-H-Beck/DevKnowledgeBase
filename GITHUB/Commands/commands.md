# 🛠️ GitHub & Git Markdown Cheatsheet

A quick reference guide for essential Git commands and GitHub-Flavored Markdown (GFM) syntax.

---

## 💾 Part 1: Essential Git Commands

Use these commands in your terminal for daily version control.

### Setup & Initialization
```bash
git init                            # Initialize a new local Git repository
git clone <url>                     # Clone an existing repository from GitHub
```

### Saving Changes
```bash
git status                          # View the status of modified files
git add <filename>                  # Add a specific file to the staging area
git add .                           # Add ALL modified and new files to staging
git commit -m "Your message"        # Commit staged changes with a descriptive message
```

### Synchronizing with GitHub
```bash
git branch -M main                  # Rename the default branch to 'main'
git remote add origin <url>         # Link the local repository to a remote GitHub repository
git push -u origin main             # Push changes to GitHub and set the upstream branch
git pull                            # Fetch and merge latest changes from the remote server
```

---

## 📝 Part 2: GitHub Markdown Syntax

Use this syntax in `README.md` files, issues, pull requests, and discussions.

### Headings
```markdown
# Heading Level 1 (Main Title)
## Heading Level 2 (Section)
### Heading Level 3 (Subsection)
#### Heading Level 4
```

### Text Formatting
```markdown
This is normal text.
This is **bold text** (or __bold__).
This is *italic text* (or _italic_).
This is ***bold and italic***.
This is ~~strikethrough text~~.
```

### Lists
```markdown
**Unordered List:**
* First item
* Second item
  * Indented sub-item
  * Another sub-item

**Ordered List:**
1. Step one
2. Step two
3. Step three
```

### Task Lists
```markdown
- [x] A completed task
- [ ] An incomplete task
- [ ] Another open task
```

### Code and Syntax Highlighting
For short inline code snippets, use single backticks: `git status`.
For multi-line code blocks, use triple backticks followed by the programming language name for syntax coloring:

```markdown
```javascript
function helloWorld() {
  console.log("Hello, World!");
}
```
```

### Links and Images
```markdown
[Visit GitHub](https://github.com)
![GitHub Logo](https://githubassets.com)
```

### Tables
```markdown

| Command | Description | Status |
| :--- | :--- | :---: |
| `git add` | Stages files | Done |
| `git commit` | Saves locally | Done |
| `git push` | Uploads code | Open |
```

### Blockquotes
```markdown
> This is an important quote or a note 
> that will be highlighted in the document.
```

### GitHub Alerts (Admonitions)
```markdown
> [!NOTE]
> Useful information that users should blink and see.

> [!TIP]
> Helpful advice for doing things more efficiently.

> [!IMPORTANT]
> Crucial information necessary for users to succeed.

> [!WARNING]
> Critical content demanding immediate user attention.
```

### Collapsible Sections
```markdown
<details>
<summary><b>Click here to expand</b></summary>

Here is the hidden content, code, or images to save visual space.

</details>
```
