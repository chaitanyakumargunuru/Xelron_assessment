# Part 3: Prompt Preparation 

## Selected PR: MetaGPT #1061 - Repository to Markdown Tool

---

## 3.1 Repository Context 

MetaGPT is a multi-agent framework that simulates a software company by assigning different roles to large language models. The framework includes agents that act as Product Managers, Architects, Engineers, and QA specialists, all collaborating to complete software development tasks. MetaGPT implements a "Code = SOP(Team)" philosophy, where Standard Operating Procedures are encoded as agent behaviors that coordinate through structured message passing.

The intended users are AI researchers, software developers, and teams exploring autonomous agent systems for software development. They use MetaGPT to automate code generation, project scaffolding, documentation creation, and software architecture design. The framework enables natural language programming where users describe what they want to build through prompts, and the agent team collaboratively generates working code.

The problem domain MetaGPT addresses is the challenge of making large language models effective at complex, multi-step software development tasks. Single-prompt approaches often produce incomplete or inconsistent code. By breaking tasks into specialized roles with defined workflows, MetaGPT achieves better results through agent collaboration and iterative refinement.


---

## 3.2 Pull Request Description 

This pull request introduces a utility tool that converts entire code repositories into structured markdown documents, enabling MetaGPT agents to understand and work with existing codebases effectively. The tool thoroughly goes through repository directories, excludes irrelevant files like .gitignore, and generates a markdown document containing both the directory structure and file contents.

The specific changes include creating a tree visualization utility that generates directory trees showing the repository's hierarchical structure, and the main repository-to-markdown converter that walks directories, reads source files, and formats them into markdown code blocks with appropriate syntax highlighting hints. The implementation includes a --gitfile option that uses git's index to determine which files to include, ensuring only version-controlled files are processed.

These changes are needed because MetaGPT agents frequently need to analyze, modify, or document entire projects. earlier, there was no standardized way to provide repository context to agents. Agents would either miss important files, or developers would manually copy-paste code which is time-consuming and error-prone. The markdown format is ideal and sufficient enough because it's text-based, structured and widely supported for llms understanding.

The previous behaviour can only instruct multiple-agents to do their respective tasks by randomly selecting the uploaded files without proper context of file locations and repo's structure. The new architecture i.e. taking whole document and understanding unnecessary files like git files and file positions to code them sequentially by correlating with created files by other agents. The markdown format helps llm's to understand the code and repo structure.

The implementation preserves the visual directory tree at the beginning of the document, followed by the full content of each file with clear headers, making it easy for both humans and AI agents to navigate the repository structure and locate specific code sections.

---

## 3.3 Acceptance Criteria 

° The tool generates a valid markdown file from a repository directory, including both directory structure and file contents.
° An ASCII-art tree view of the directory structure is included at the start, showing a hierarchical representation of folders and files.
° Each source file is represented with its full relative path as a header, and its content is wrapped in markdown code blocks with            appropriate language identifiers (e.g., ```python).
° The tool respects .gitignore patterns, ensuring excluded files and directories do not appear in the markdown.
° When using the --gitfile option, only files tracked by git are included, determined via git's index or through the command git ls-files.
° Binary files, images, and other non-text files are either excluded from output or noted as binary.
° The tool identifies various programming languages for correct syntax highlighting in the code blocks.
° The output markdown file is formatted according to markdown specifications to be parseable by standard markdown processors.
° It handles edge cases such as empty directories, files with special characters, and deeply nested structures without errors.

---

## 3.4 Edge Cases 

**Edge Case 1: Large Repositories Exceeding Token Limits**

When input is very large repositories (thousands of files, millions of lines of code), the generated markdown could exceed language model token limits. The tool should either provide options to filter files by pattern (e.g., only include *.py files), split output into multiple markdown files organized by directory, or implement intelligent sampling strategies. Users should receive warnings when output size is likely to exceed common LLM context windows like claude ai provides.

**Edge Case 2: Files with Encoding Issues**

Source files may use various character encodings (UTF-8 etc.), contain non-printable characters, or have encoding errors. A robust tool should effectively manage these challenges by employing multiple encoding detection methods, such as libraries like chardet, replacing problematic characters with placeholders, or omitting files that cannot be decoded while providing appropriate logging for warnings. the tool's design must ensure that it does not crash in the face of encoding issues.


**Edge Case 3: Symbolic Links and Circular References**

Repositories may contain symbolic links that create circular directory references or link to locations outside the repository. The tool must detect and handle these to avoid infinite loops during directory traversal. Implementation should track visited directories and skip links that would create cycles or point outside the repository boundary.

**Edge Case 4: Git Submodules**

Repositories using git submodules present special challenges. Submodules are separate git repositories nested within the main repository. The tool should handle these by either treating submodule directories as hidden or providing an option to recursively process submodules. The implementation must respect each submodule's own gitignore rules.

**Edge Case 5: Special Files and Metadata**

Some repositories contain special files like .git directories, __pycache__ folders, node_modules, or large binary assets. The tool must intelligently skip these automatically even if not explicitly in gitignore. The implementation should have a default exclusion list for common build artifacts and dependency directories across different ecosystems (Python, Node.js, Java, etc.) to prevent the output with irrelevant content or to avoid exceeding the output token limit.

---

## 3.5 Initial Prompt 

**Task: Implement Repository to Markdown Conversion Tool for MetaGPT**

You are working on MetaGPT, a multi-agent framework for AI-powered software development. Your task is to create a utility tool that converts code repositories into structured markdown documents that can be easily understood by MetaGPT's AI agents.

**Context:**

MetaGPT agents need to understand existing codebases to perform tasks like code review, documentation generation, refactoring, and analysis. Language models work best with text-based, structured input. Your tool will enable agents to receive complete repository context in a format optimized for LLM consumption.

**Requirements:**

1. Create {metagpt/utils/tree.py}:
   - Implement a function that generates ASCII-art directory trees of provided folder or repo
   - Use box-drawing characters (├──, └──, │) for visual structure
   - Support recursive directory traversal
   - Return string representation of the tree

2. Create {metagpt/utils/repo_to_markdown.py}:
   - Implement main conversion function that takes repository path as input
   - Walk directory tree recursively using pathlib
   - Read .gitignore and exclude matching files/directories
   - Add --gitfile option to only include git-tracked files
   - Generate markdown with two sections:
     a) Directory tree view at the top
     b) File contents with headers and code blocks
   - Detect file types and add appropriate language identifiers to code blocks
   - Handle binary files intelligently ( like skip or note as binary)

3. Create {tests/metagpt/utils/test_repo_to_markdown.py}:
   - Test basic conversion with a sample repository structure
   - Test gitignore pattern handling
   - Test various file types and encodings
   - Test edge cases like empty directories and special characters

4. Create {tests/metagpt/utils/test_tree.py}:
   - Test tree generation for various directory structures
   - Verify correct ASCII art formatting
   - Test depth limits if implemented

5. Update {metagpt/utils/__init__.py}:
   - Export the new utility functions for easy import

**Technical Specifications:**

- Use `pathlib.Path` for cross-platform path handling
- Use `gitignore_parser` or similar library for .gitignore support
- For git integration, use subprocess to call `git ls-files` when --gitfile is used
- Detect binary files using magic bytes or file extensions
- Support language detection for syntax highlighting (Python, JavaScript, Java, etc.)
- Write output to a file with timestamp or hash in filename to avoid overwrites


Your implementation must satisfy all acceptance criteria 
- Valid markdown output with tree view and file contents
- Proper gitignore handling
- Binary file exclusion
- Correct syntax highlighting hints

Pay special attention to the edge cases :
- Large repositories that might exceed token limits
- Files with encoding issues or special characters
- Symbolic links and circular references
- Git submodules
- Common build artifacts and dependencies (node_modules, __pycache__, etc.)

- Follow MetaGPT's coding style and conventions
- Add comprehensive docstrings for all functions
- Use type hints for function parameters and returns
- Include error handling with informative messages
- Log warnings for skipped files or issues encountered

- Create a test repository structure with various file types
- Test with actual git repositories to verify git integration
- Verify markdown output is valid and properly formatted
- Test performance with moderately large repositories (100+ files)
- Ensure tool works on Windows, Linux, and macOS
* incase in need of any extra file or functionality according to the need add it corresponding folder.
  
**Output Format Example:**

```markdown
# Repository Structure

\```
project/
├── src/
│   ├── main.py
│   └── utils.py
└── tests/
    └── test_main.py
\```

# File Contents

## src/main.py

\```python
def main():
    print("Hello, world!")
\```

## src/utils.py

\```python
def helper():
    return 
\```
```

**Deliverables:**

1. Working tree.py utility module
2. Complete repo_to_markdown.py implementation
3. Comprehensive test suites for both utilities
4. Example output markdown from running on a sample repository
5. documentation of the repo

---

