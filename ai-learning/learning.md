## **OpenCode + Google Gemini Setup**

Follow these steps to install OpenCode, connect Google Gemini, and start analysing a project.

**1. Install Node.js**

Download and install the LTS version of Node.js from: [https://nodejs.org/](https://nodejs.org/?utm_source=chatgpt.com)

**2. Verify Node.js and npm installation**

```
node --version
npm --version
```

**3. Install OpenCode**

```
npm.cmd install -g opencode-ai
```

**4. Verify OpenCode installation**

```
opencode --version
```

**5. Launch OpenCode**

```
opencode
```

**6. Connect Google Gemini**

- Visit [https://aistudio.google.com/apikey](https://aistudio.google.com/apikey?utm_source=chatgpt.com)
- Sign in and create a Google Gemini API key.
- Copy the API key.
- In OpenCode, execute `/connect`.
- Select Google and provide your API key.

**7. Select an AI model**

```
/models
```

Choose a text-based Gemini Flash model for regular code analysis and development tasks.

**8. Exit OpenCode**

```
/exit
```

**9. Navigate to your GitHub project**

Replace the path with your actual project folder.

```
cd path\to\career-ops
```

**10. Launch OpenCode inside the project**

```
opencode
```

**11. Start the Career-Ops workflow**

```
/career-ops
```

**12. Change OpenCode mode**

Press `Tab` to switch between modes:

- **Plan mode:** Analyse the project and suggest changes without implementing them.
- **Build mode:** Implement changes and modify project files.

**13. Start analysing your project**

Example prompt:

```
Analyse the project structure, understand the existing workflows,
identify the important modules, and explain how the application works.
Do not modify any files.
```
