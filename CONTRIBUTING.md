# Contributing Guidelines

We welcome your contributions to further improve this project and to increase the success rate of AIs writing SketchUp extensions!

## How to Contribute

1. **Fork** this repository.
2. Create a new branch: `git checkout -b feature/AddNewRule`
3. Commit your changes: `git commit -m 'feat: added new rule for Observers'`
4. Push your branch: `git push origin feature/AddNewRule`
5. Open a **Pull Request (PR)**.

## AI-Friendly Format Requirements

When adding a new rule to the `sketchup_api_skill.md` file, please maintain the following format exactly so that AI models can properly understand your instructions:
- **Clear Headings:** Use a clear heading indicating what the rule is about.
- **Code Examples:** Always provide `❌ Bad` (incorrect usage) and `✅ Good` (correct usage) examples.
- **Provide Reasoning:** Briefly explain why the rule exists (e.g. Performance overhead, crash risk, SketchUp API limitations). Always write your explanations in English, as AI algorithms process English technical terms and context much better.
