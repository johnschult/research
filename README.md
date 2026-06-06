# Research projects

Each directory in this repo is a self-contained research project — a question
I wanted answered by building and experimenting rather than reading.  Nice

Most of the code and write-ups are produced by an async coding agent
(Claude Code) following the workflow in [AGENTS.md](AGENTS.md): spin up a
folder, investigate, keep running notes, and write a `README.md` report at the
end. Prompts and transcripts are linked from the commits that added each project.

Pattern credit: [Simon Willison's async code research approach](https://simonwillison.net/2025/Nov/6/async-code-research/).

## Projects

<!-- Project summaries are appended below as each investigation completes. -->

### [llm-cors-static-html](llm-cors-static-html/)

Can a single static HTML file call an LLM API directly via CORS with no backend? Empirically tested CORS preflight responses from Anthropic, Groq, OpenAI, Gemini, and Together AI — all five pass as of June 2026. Built a working single-file demo (`demo.html`) using Anthropic's API with a bring-your-own-key pattern. Conclusion: **yes, CORS is solved**; the real constraint is API key exposure, which the BYOK pattern manages for internal tools and local experiments. A restricted or throwaway key is needed for anything public-facing.
