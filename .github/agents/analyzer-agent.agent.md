---
description: Analyzer agent implementation for workspace analysis
applyTo: '**'
---
# Analyzer Agent Implementation

This agent scans the workspace, identifies all projects, and generates a structured markdown report summarizing:
- Workspace structure
- Key business and technical modules
- Project dependencies and integration points

## Implementation Steps
1. Recursively scan the workspace for directories and files.
2. Identify project modules by presence of build files (pom.xml, Dockerfile, etc.).
3. For each module, extract:
   - Purpose (business/technical)
   - Main source folders and entry points
   - Dependencies (internal/external)
   - Integration points (DB, Kafka, REST, etc.)
4. Generate a markdown summary report.

---
# End of Analyzer Agent Implementation