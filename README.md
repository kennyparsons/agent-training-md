# Agent Training & Context Library

## Purpose

This repository serves as a modular "brain" or "skill set" for AI agents. It contains a collection of Markdown documents that, when read by an agent, instantly train it on specific operational protocols, tools, and infrastructure tasks.

Instead of prompting an agent from scratch every time ("Here is how you use my remote command tool...", "Here is my security policy..."), you point the agent to this folder. It ingests the context and immediately understands the rules of engagement and how to perform complex tasks on your specific infrastructure.

## How to Use

1.  **Initialize Agent**: Start your agent session.
2.  **Load Context**: Instruct the agent to "read all instructions in this directory".
3.  **Execute**: The agent is now capable of performing the documented tasks (like creating LXC containers or configuring Tailscale) while adhering to your safety and operational standards.

## Philosophy

*   **Context as Code**: Operational knowledge is version-controlled and human-readable.
*   **Zero Assumption**: Instructions are written to assume zero prior state knowledge (e.g., "Find the next ID" vs "Use ID 100").
*   **Idempotency & Safety**: Protocols emphasize verification and explicit user approval for destructive actions.
