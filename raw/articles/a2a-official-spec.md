---
title: A2A Protocol Official Specification Research
created: 2026-05-02
updated: 2026-05-02
type: raw-source
tags: [protocol, agent]
source_url: https://github.com/a2aproject/A2A
---

# A2A Official Spec — Raw Research Notes

## Source
- Repository: https://github.com/a2aproject/A2A
- Version: v1.0
- Canonical spec: Protocol Buffers (a2a.proto)

## Key Findings
- See compiled wiki pages: a2a-protocol.md, a2a-task-state-machine.md
- Three protocol bindings: JSON-RPC 2.0, gRPC, HTTP/REST
- Content-Type: application/a2a+json
- Task is the central unit, contextId for session continuity
- AgentCard for discovery via .well-known/agent-card.json
- Full task state machine with 8 states (2 new in v1.0: auth_required, unknown)
- SSE streaming with polymorphic StreamResponse
