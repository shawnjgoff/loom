# Loom Learning Curriculum

Welcome to the comprehensive Loom learning curriculum! This guide will walk you through every component of the Loom system in the optimal learning order.

## Who This Is For

This curriculum is designed for anyone wanting to deeply understand how Loom works internally:
- Developers contributing to Loom
- Operators deploying and managing Loom
- Advanced users wanting to understand the system architecture
- Anyone curious about building AI-powered developer tools

## How To Use This Curriculum

The documents are designed to be **listened to**—they contain no code snippets, only descriptions of how things work. Each document:
- Explains the component's role in the ecosystem
- Describes what other components it interacts with
- Includes a glossary of referenced components at the top
- Explains how the component works in words

**Read (or listen to) them in order.** Each document builds on concepts from previous ones.

## Curriculum Structure

### Start Here: Overview

📄 **[00-overview.md](./00-overview.md)** - Start here to understand the big picture

This document assembles all components into a working system and walks through a complete user journey from starting a session to deploying to weavers. Read this first to get oriented.

### Core Learning Path

These documents cover the essential components that form the foundation of Loom. Read them in this order:

1. 📄 **[01-agent-state-machine.md](./01-agent-state-machine.md)** - The orchestrator at the heart of Loom

   Understand how conversations flow through explicit states and why this design enables predictable, testable behavior.

2. 📄 **[02-thread-system.md](./02-thread-system.md)** - Persistence and synchronization of conversations

   Learn how conversations are saved, synced across devices, and recovered after crashes.

3. 📄 **[03-tool-system.md](./03-tool-system.md)** - How agents interact with the real world

   Discover how tools let AI read files, execute commands, and make changes while maintaining security boundaries.

4. 📄 **[04-llm-integration.md](./04-llm-integration.md)** - Connecting to AI providers

   Explore the abstraction layer that enables Loom to work with multiple LLM providers and stream responses in real-time.

5. 📄 **[05-server-authentication.md](./05-server-authentication.md)** - Multi-user security and organizations

   Understand authentication flows, authorization policies, and how multi-tenant organizations work.

6. 📄 **[06-weaver-system.md](./06-weaver-system.md)** - Remote execution environments

   Learn how Loom provisions isolated containers on Kubernetes for secure, ephemeral compute.

### Advanced Topics

After completing the core path, explore these additional systems:

7. 📄 **[07-advanced-topics.md](./07-advanced-topics.md)** - All the additional systems

   A comprehensive overview of analytics, feature flags, SCM, TUI, editor integration, web frontend, and more.

## Learning Tips

**Take Your Time** - These are dense technical documents. Don't rush. It's fine to re-read sections or take breaks.

**Follow Cross-References** - When a document mentions another component, you can jump to that component's document to understand how they interact.

**Check the Specs** - Each document references specifications in `specs/`. The specs provide even more detail and include implementation notes.

**Explore the Code** - After reading a document, browse the mentioned crates in `crates/`. The code structure mirrors the conceptual structure.

**Ask Questions** - If something isn't clear, open an issue on GitHub or ask in the community channels.

## Document Structure

Each document (except the overview) follows this structure:

**Component Role** - A one-paragraph description of what this component does and why it exists.

**Referenced Components** - A glossary of other components mentioned in the document with brief descriptions.

**How It Works** - The detailed explanation of the component's design and behavior.

**Why This Design?** - The rationale for key design decisions.

**Implementation Location** - Where to find the code in the repository.

## Estimated Time

- **Overview**: 30-45 minutes
- **Each Core Document**: 45-60 minutes
- **Advanced Topics**: 60-90 minutes

**Total Time**: ~8-10 hours for complete curriculum

You don't need to do it all in one sitting! Many people find it helpful to:
- Complete the overview first (establishes mental model)
- Do one core document per day
- Refer back to earlier documents when needed

## After The Curriculum

Once you've completed the curriculum, you'll have a comprehensive understanding of:
- How conversations flow through the agent state machine
- How threads are persisted and synced across devices
- How tools enable AI to interact with the world securely
- How LLM integration abstracts provider differences
- How authentication and authorization work
- How weavers provide isolated compute environments
- What additional systems extend Loom's capabilities

You'll be ready to:
- Contribute features to any part of the system
- Debug issues across component boundaries
- Design new integrations or extensions
- Deploy and operate Loom in production

## Contributing

Found an error or unclear explanation? Please open a pull request! These documents are version-controlled and community-maintained.

Good documentation is as important as good code. Help us make this curriculum better for future learners.

## Related Resources

- **specs/** - Detailed technical specifications for all components
- **CLAUDE.md** - Guidelines for working with the codebase
- **Architecture Decision Records** - Context for major design decisions (coming soon)
- **API Documentation** - OpenAPI specs for all HTTP endpoints

## Getting Help

- **GitHub Issues**: Bug reports and feature requests
- **GitHub Discussions**: Questions and general discussion
- **Discord/Slack** (if available): Real-time chat with the community

---

**Ready to begin?** Start with [00-overview.md](./00-overview.md) to see the big picture!
