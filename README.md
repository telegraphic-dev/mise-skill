# mise skill

A practical agent skill for [mise](https://mise.jdx.dev/): reproducible development tools, project environments, and task execution.

It emphasizes the safe agent workflow: inspect configuration first, use `mise exec` / `mise run` instead of shell activation, and do not trust unknown configs or commit secrets.

## Install

```sh
npx --yes skills add telegraphic-dev/mise-skill
```

This installs the skill guidance; it does **not** install the mise binary. Install or verify mise separately:

```sh
mise --version
```

See [SKILL.md](SKILL.md) for workflows, commands, troubleshooting, and safety boundaries.

## Sources

- [mise documentation](https://mise.jdx.dev/)
- [mise CLI reference](https://mise.jdx.dev/cli/)
- [jdx/mise](https://github.com/jdx/mise)

## License

[MIT](LICENSE)
