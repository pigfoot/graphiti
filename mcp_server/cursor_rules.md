## Instructions for Using Graphiti's MCP Tools for Agent Memory

### Before Starting Any Task

- **Always search first:** Use the `search_nodes` tool to look for relevant preferences and procedures before beginning work.
- **Search for facts too:** Use the `search_facts` tool to discover relationships and factual information that may be relevant to your task.
- **Filter by entity type:** Specify `Preference`, `Procedure`, or `Requirement` in your node search to get targeted results.
- **Review all matches:** Carefully examine any preferences, procedures, or facts that match your current task.

### Always Save New or Updated Information

- **Capture requirements and preferences immediately:** When a user expresses a requirement or preference, use `add_memory` to store it right away.
  - _Best practice:_ Split very long requirements into shorter, logical chunks.
- **Be explicit if something is an update to existing knowledge.** Only add what's changed or new to the graph.
- **Document procedures clearly:** When you discover how a user wants things done, record it as a procedure.
- **Record factual relationships:** When you learn about connections between entities, store these as facts.
- **Be specific with categories:** Label preferences and procedures with clear categories for better retrieval later.

### During Your Work

- **Respect discovered preferences:** Align your work with any preferences you've found.
- **Follow procedures exactly:** If you find a procedure for your current task, follow it step by step.
- **Apply relevant facts:** Use factual information to inform your decisions and recommendations.
- **Stay consistent:** Maintain consistency with previously identified preferences, procedures, and facts.

### Best Practices

- **Search before suggesting:** Always check if there's established knowledge before making recommendations.
- **Combine node and fact searches:** For complex tasks, search both nodes and facts to build a complete picture.
- **Use `center_node_uuid`:** When exploring related information, center your search around a specific node.
- **Prioritize specific matches:** More specific information takes precedence over general information.
- **Be proactive:** If you notice patterns in user behavior, consider storing them as preferences or procedures.

### Multi-Tenant Usage

- **Project Context Awareness:** Your default project context is set by X-Project headers in your MCP configuration. All tools automatically use this context unless overridden.
- **Cross-Project Operations:** Use explicit `group_id` parameters to access data from other projects when needed:
  ```
  search_memory_nodes("requirements", group_id="shared-project")
  add_memory("Cross-project note", "...", group_id="other-project")
  ```
- **Respect Project Boundaries:** Unless explicitly directed, keep data within the current project context. Don't accidentally leak information between projects.
- **Project-Specific Preferences:** Remember that preferences and procedures are project-specific. A preference in "project-a" doesn't apply to "project-b".

**Remember:** The knowledge graph is your memory. Use it consistently to provide personalized assistance that respects the user's established preferences, procedures, and factual context within the appropriate project scope.
