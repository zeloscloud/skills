# Zelos skills

Skills for the Zelos ecosystem, in the open [Agent Skills](https://agentskills.io) format.

| Skill | What it does |
|-------|--------------|
| [develop-extension](skills/develop-extension/SKILL.md) | Builds a Zelos extension that reads your device or data feed and streams its decoded signals live into the Zelos app. |

## Install

```bash
npx skills add zeloscloud/skills
```

This works with any coding agent that reads `SKILL.md` files. An agent without
skill support can fetch the file and follow it:

```text
https://raw.githubusercontent.com/zeloscloud/skills/main/skills/develop-extension/SKILL.md
```

## Use

Open the Zelos app, then ask your coding agent:

```text
Build a Zelos extension so I can see my device's data live in the Zelos app.
This is a first proof: get these signals live, nothing more.

Use the develop-extension skill.

What I have:
- Device: <what it is>
- Connection: <port and settings, or host and port>, on <OS>
- Protocol: <what you know, or attach the document>
- Samples: <raw bytes or lines from the device>
- I want to see: <the values, with units>

Do not send anything to the device.
```

You need the [Zelos CLI](https://docs.zeloscloud.io/latest/getting-started/install-cli/)
and [uv](https://docs.astral.sh/uv/). More on extensions:
<https://docs.zeloscloud.io/latest/sdk/how-to/develop-agent-extensions/>.
