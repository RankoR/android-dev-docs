# Android Documentation

This repository contains documentation for Android development standards and architectures used in my projects.

## Usage

Copy-paste all files (except `AGENTS.md`) to the `docs` directory inside the project root.

Do not forget to change package names and do other adjustments that are specific to your project :)

If you're using coding agents, copy-paste `AGENTS.md` to the **project root**.

### MCP

### Context7

I strongly recommend using context7 MCP for docs. `AGENTS.md` contains instruction to use this MCP, so you need to add this MCP to your Agent's settings:

```
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": [
        "-y",
        "@upstash/context7-mcp"
      ]
    }
  }
}
```

If you don't want to use context7 MCP, don't forget to remove it from the `AGENTS.md`.

### ADB

*TODO*: Add ADB MCP information.

## Architectures

### Native Android (Compose + Dagger/Hilt)

My standard architecture for native Android applications using Jetpack Compose and Dagger Hilt.

[Read the documentation](./android-compose-dagger/DEVELOPMENT.md)

**A note on Modularization:**
Currently, these docs do not cover multi-module setups as the primary focus is on **rototype development**.

However, the architecture—including strict package structure, class visibility controls, and extensive use of `internal` modifiers—is designed to be **"module-ready"**. This allows for a fast monolithic start while ensuring a smooth transition to a multi-modular system as the project scales.

I hope that in the future this documentation will be producton-ready - your contributions are welcome!

### Compose Multiplatform

Planned support for cross-platform development using Compose Multiplatform.

*TODO: Not implemented yet.*

## Misc

Please note that docs are incomplete.

What is missing:

- Multi-modular projects docs (currently the docs are mostly targeting MVPs where multi-module arch is redundant)
- Compose Multiplatform docs
- Testing docs
- Instructions on how to use ADB MCP

Feel free to open a pull request!

## License

This project is licensed under the WTFPL License - see the [LICENSE](LICENSE) file for details.
