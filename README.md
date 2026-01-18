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

*TODO*

## Architectures

### Native Android (Compose + Dagger/Hilt)

My standard architecture for native Android applications using Jetpack Compose and Dagger Hilt.

[Read the documentation](./android-compose-dagger/DEVELOPMENT.md)

### Compose Multiplatform

Planned support for cross-platform development using Compose Multiplatform.

*TODO: Not implemented yet.*

## Misc

Please note that docs are incomplete. For example, there are still no docs related to testing.

Feel free to open a pull request!

## License

This project is licensed under the WTFPL License - see the [LICENSE](LICENSE) file for details.
