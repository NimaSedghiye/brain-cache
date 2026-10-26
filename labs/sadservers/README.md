# SadServers

Educational write-ups for [SadServers](https://sadservers.com/) scenarios.
SadServers provides hands-on Linux and DevOps exercises on real virtual
machines, covering topics such as services, storage, networking, databases,
and containers.

The goal of this collection is to explain how to approach each problem,
interpret diagnostic evidence, identify the root cause, and verify the fix.
Each write-up should help readers apply the same reasoning to similar issues.

## Before You Start

Basic familiarity with the Linux command line, files and permissions, and
`sudo` will help you follow the examples. Some scenarios also involve services,
networking, databases, or containers; use the topics in the index to find an
exercise that matches what you want to learn.

## How to Use This Collection

1. Choose a scenario from the index below and read its problem, environment,
   and success criteria.
2. Open the linked scenario on SadServers and try it yourself before reading
   the solution.
3. Compare your approach with the investigation and resolution in the write-up.
   Focus on what each command reveals and why the next step follows.
4. Run the verification steps in the scenario environment and review the key
   takeaways for techniques you can reuse.

Solution sections contain spoilers. Commands refer to the scenario's lab
environment, so follow along in its VM.

## Scenario Index

| Scenario | Difficulty | Type | Topics | Documentation |
| --- | --- | --- | --- | --- |
| Saint John: What Is Writing to This Log File? | Easy | Fix | Processes, Open Files, Signals | [Read write-up](easy/saint-john/README.md) |
| Manhattan: Can't Write Data into Database | Medium | Fix | Disk Volumes, PostgreSQL, systemd | [Read write-up](medium/manhattan/README.md) |
| Tokyo: Can't Serve Web File | Medium | Fix | HTTP, Apache, iptables, File Permissions | [Read write-up](medium/tokyo/README.md) |

Scenario types follow the terminology used by SadServers:

- **Fix:** Diagnose and repair a broken or misconfigured system.
- **Do:** Complete an installation, configuration, or other task.
- **Hack:** Solve a security challenge within the provided lab environment.

## Organization

Scenarios are grouped by their official difficulty: `easy`, `medium`, or `hard`.
Each scenario has its own directory and a `README.md` containing the problem
and its solution.

The intended layout is:

```text
sadservers/
├── README.md
├── easy/
│   └── <scenario-slug>/
│       └── README.md
├── medium/
│   └── manhattan/
│       └── README.md
└── hard/
    └── <scenario-slug>/
        └── README.md
```

Difficulty directories appear as write-ups are added. For a particular
technology, such as PostgreSQL or Docker, browse the topics in the index.

## Reading a Write-up

The write-ups follow the path from understanding the problem to checking the
result:

- **Problem:** The objective, environment, constraints, and success criteria,
  with a link to the official scenario.
- **Investigation:** Diagnostic commands and relevant output, with the reasoning
  that connects observations to the next step.
- **Root Cause:** How the underlying issue explains the symptoms. Task-based
  scenarios instead explain the requirements behind the chosen approach.
- **Resolution:** The commands and changes used to reach the objective, with
  explanations of how they work.
- **Verification:** Checks that demonstrate the result, including the scenario's
  acceptance test where available.
- **Key Takeaways:** Troubleshooting principles and techniques for similar
  problems.

## Resources

- [Official scenarios](https://sadservers.com/scenarios)
- [SadServers FAQ](https://sadservers.com/faq)
- [Official documentation](https://docs.sadservers.com/)
