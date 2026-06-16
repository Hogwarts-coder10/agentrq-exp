# AgentRQ Roadmap

## 1. AgentRQ CLI

A command-line interface that connects to the AgentRQ backend and spawns agents for desired workspaces. Enables developers to manage workspaces, trigger tasks, and attach agents directly from the terminal without the web UI.

## 2. Agent-to-Agent Collaborative Workflows

Structured multi-agent execution based on a Supervisor → Worker architecture. A supervisor agent breaks down a mission and delegates subtasks to specialized worker agents across workspaces, coordinating results to complete complex goals. (Further design TBD.)

## 3. Skills Marketplace

A curated registry of reusable agent skills that can be discovered, installed, and shared across workspaces. Teams can publish their own skills and compose them into new workflows.

## 4. MCP Marketplace

A marketplace for Model Context Protocol (MCP) servers — enabling teams to discover, install, and configure MCP integrations directly within AgentRQ workspaces.

## 5. Workflow Marketplace

A library of pre-built end-to-end workflows described in plain English that orchestrate multiple workspaces in sequence. Each step in a workflow can be handled by a different specialized agent, enabling powerful cross-domain pipelines — for example: coding → docs → blogs → SEO indexing → social posts. Workflows can be imported, forked, and customized to fit any team's process.
