---
description: Analyzer agent for workspace structure and project scanning
applyTo: '**'
---
# Analyzer Agent Instructions

## Purpose
This agent analyzes the workspace structure, identifies key business and technical components, and scans all projects in the workspace.

## Workflow
1. **Analyze Workspace Structure:**
   - List all top-level directories and files.
   - Identify sub-projects (e.g., Java modules, scripts, Dockerfiles, data folders).
2. **Identify Key Business and Technical Parts:**
   - Detect business logic modules (e.g., services, domain, core, producer/consumer patterns).
   - Detect technical infrastructure (e.g., Docker, database, messaging, configuration, scripts).
3. **Scan All Projects:**
   - For each detected project/module:
     - Summarize its purpose (business or technical).
     - List main source folders and entry points.
     - Note dependencies (internal/external).
     - Highlight integration points (e.g., DB, Kafka, REST APIs).

## Best Practices
- Use directory and file naming conventions to infer project roles.
- Prefer automated scanning (e.g., look for pom.xml, Dockerfile, src/, data/).
- Summarize findings in a structured markdown report.
- Highlight any missing or ambiguous parts for user review.

## Output
- Markdown summary with:
  - Workspace structure tree
  - Table of projects/modules with type (business/technical), main files, dependencies, integration points
  - Observations and recommendations

---
# End of Analyzer Agent Instructions