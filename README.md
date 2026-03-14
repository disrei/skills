# knowledge base skills
1. project-kb-init
Purpose: Initializes a project knowledge base for the first time when knowledge/ doesn't exist.
Key Features:
- Two modes: Lightweight (creates only directory framework + data_structure.md files) and Full (complete documentation with code analysis)
- Supports multiple project types: Web, Godot, large modular games, and generic services
- Creates hierarchical index trees with data_structure.md at every level
- Can operate in full-project or single-module mode
- Requires user confirmation before creating functional module directories
2. project-kb-update
Purpose: Maintains and updates an existing project knowledge base when knowledge/ already exists.
Key Features:
- Structure-first principle: reads existing structure before updating
- Evidence-driven: only documents verified knowledge from code or user confirmation
- Boundary protection: prompts for confirmation when content is out-of-scope
- Concurrency safety: uses pending markers and cache mechanism to prevent conflicts
- Conflict handling: manages version drift and concurrent update conflicts
- Incremental updates with minimal changes to existing structure
3. kb-retriever (rag-skill)
Purpose: Retrieves information and answers questions from local knowledge base directories.
Key Features:
- Hierarchical navigation via data_structure.md index files
- Multi-file type support: Markdown/txt, PDF, Excel
- Learn-first approach: must read reference documentation before processing PDF/Excel files
- Progressive retrieval: uses grep, partial reads, and specialized tools (pdfplumber, pandas)
- Iterative search mechanism (up to 5 rounds) with keyword refinement
- Provides source attribution in answers (file paths, line numbers)
