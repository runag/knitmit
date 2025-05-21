# Knitmit

**`knitmit`** is a portable, self-contained, and configurable Bash script that streamlines your Git workflow by generating commit messages using Large Language Models (LLMs). It analyzes your staged changes, constructs a detailed prompt, queries your chosen LLM, and returns a commit message — ready for review or immediate use.

## Features

* **LLM-powered commit messages:** Uses large language models to suggest clear, well-formatted commit messages.
* **Contextual prompting:** Builds prompts from staged Git diffs and recent commit history to provide relevant context.
* **Prompt customization:** Offers a "short" prompt option with minimal diff context — ideal for large changes or when hitting token limits.
* **Custom commit instructions:** Supports overriding default commit message guidelines via a local configuration file.
* **Flexible LLM configuration:**
  * Supports multiple LLM backends defined in the `model_preferences` section of the config file.
  * Attempts each configured model in order until one succeeds.
  * **Built-in Gemini support:** Includes a `query::gemini` function that uses `curl` and the `GEMINI_API_KEY`.
  * **Support for local CLI-based LLMs:** Easily integrates with command-line tools that read from `stdin` and write to `stdout`.
* **Configuration file:** Centralizes settings in `knitmit.json` located in your platform's standard config directory.
* **Clipboard integration:**
  * Copies the generated prompt to your clipboard.
  * Copies the LLM’s response to your clipboard.
* **Platform-aware behavior:** Detects the appropriate configuration directory and clipboard tool (`wl-copy`, `xclip`, `pbcopy`, `clip`) for the current OS.
* **Git-aware integration:**
  * Automatically uses the LLM’s output as a `git commit` message.
  * Verifies that you're inside a Git repository with staged changes.
* **Smart output handling:** When output is redirected or piped, it skips clipboard and commit actions — printing the LLM response and exiting.
* **Robust error handling:** Displays clear error messages and full stack traces to aid in debugging.
* **Informative logging:** Clearly communicates what the tool is doing — including model selection, clipboard actions, and more.

## Usage

Run `knitmit` (or `git knitmit`) inside a Git repository with staged changes — both forms behave the same:

```bash
# Query the LLM for a commit message and open the commit editor with the suggestion
knitmit

# Use a shorter prompt (minimal diff) to query the LLM and open the commit editor with the suggestion
knitmit short

# Generate the full prompt, copy it to the clipboard, and exit (no LLM query)
knitmit copy

# Generate a short prompt, copy it to the clipboard, and exit (no LLM query)
knitmit short copy

# Query the LLM for a commit message, copy the response to the clipboard, and exit (no commit)
knitmit result

# Use a short prompt to query the LLM, copy the response to the clipboard, and exit (no commit)
knitmit short result

# Query the LLM for a commit message, write the response to stdout, and exit (no commit)
knitmit > message.txt

# Use a short prompt to query the LLM, write the response to stdout, and exit (no commit)
knitmit short > message.txt

# Show commit message formatting instructions (useful for creating a custom config file)
knitmit commit-instructions

# Display the help message
knitmit help
knitmit -h
knitmit --help
```

## Requirements

* **Bash:** This script is written in Bash and requires version 4.0 or later (released in 2009). macOS includes Bash 3.2 by default and no longer updates it, so you’ll need to install a newer version if you're on macOS — use Homebrew: `brew install bash`.
* **Git:** Must be installed, and the script must be run within a Git repository.
* **jq:** Required for parsing JSON responses and configuration files. Install it with your package manager, e.g., `sudo apt install jq` or `brew install jq`.
* **curl:** Used to make requests to the Gemini API when using the built-in `query::gemini`. Typically pre-installed on most systems.
* **LLM access and tools:**
  * For **Gemini models** (`query::gemini`):
    * A Google Gemini API key.
    * The `GEMINI_API_KEY` environment variable must be set.
  * For **other local CLI LLM tools** listed in `model_preferences` (e.g., `aichat`, `llm`, `sgpt`, `ollama`):
    * These tools must be installed and accessible via your system’s `PATH`.
    * They should accept input via `stdin` and return output via `stdout`.
* **Clipboard tools** (optional but recommended):
  * **Linux (Wayland):** `wl-copy`
  * **Linux (X11):** `xclip`
  * **macOS:** `pbcopy`
  * **Windows (Git Bash/MSYS/Cygwin):** `clip`
  * If no clipboard tool is detected, the script will simply print the content to standard output.

## Installation

1.  **Download the script:**
    Save the script content as `knitmit` (or any name you prefer) in a directory.

2.  **Make it executable:**
    ```bash
    chmod +x knitmit
    ```

3.  **Place it in your PATH:**
    Move the `knitmit` script to a directory that is part of your system's `PATH` environment variable (e.g., `~/.local/bin`, `/usr/local/bin`).
    ```bash
    mv knitmit ~/.local/bin/ # Example for Linux/macOS
    ```

4.  **(Optional) Set up `git-knitmit` alias/subcommand:**
    To use the script as `git knitmit`, you have two main options:

    * **Symlink/Rename:** Create a symlink named `git-knitmit` pointing to your `knitmit` script in a directory that's in your `PATH`. Git automatically picks up executables named `git-*` as subcommands.
        ```bash
        ln -s ~/.local/bin/knitmit ~/.local/bin/git-knitmit
        ```
        Or, simply rename `knitmit` to `git-knitmit`.

    * **Git Alias (Alternative):** You can also create a Git alias in your `~/.gitconfig`:
        ```ini
        [alias]
            knitmit = !/path/to/your/knitmit
        ```
        Replace `/path/to/your/knitmit` with the actual path to the script.

## Configuration

`knitmit` looks for a configuration file named `knitmit.json`.

* **Linux:** `~/.config/knitmit.json` (or `$XDG_CONFIG_HOME/knitmit.json`)
* **macOS:** `~/Library/Application Support/knitmit.json`
* **Windows:** `%APPDATA%/knitmit.json`

If the file is not found, `knitmit` falls back to the following default configuration:

```json
{
  "commit_with_template": true,
  "copy_prompt": false,
  "copy_response": false,
  "interactive_prompt_limit": 139000,
  "query_language_model": true,
  "report_unavailable_commands": false,
  "model_preferences": [
    ["query::gemini", "gemini-2.5-flash-preview-05-20"],
    ["query::gemini", "gemini-2.0-flash"],
    ["query::gemini", "gemini-2.0-flash-lite"],
    ["aichat"],
    ["llm"],
    ["sgpt"]
  ]
}
```

### Configuration options

* `commit_with_template` (boolean, default: `true`): If `true`, after receiving a response from the LLM, `knitmit` will run `git commit --template <(response)` to open your editor with the suggested message.
* `copy_prompt` (boolean, default: `false`): If `true`, the generated prompt is copied to the clipboard *before* sending it to the LLM.
* `copy_response` (boolean, default: `false`): If `true`, the LLM's response is copied to the clipboard.
* `interactive_prompt_limit` (integer, default: `139000`): Maximum prompt length considered suitable for interactive use (e.g., pasting into a web UI). If exceeded while `copy_prompt` is active or the `copy` command is used, a warning is shown.
* `query_language_model` (boolean, default: `true`): If `false`, no LLM will be queried. This is useful when you only want to generate and copy the prompt.
* `report_unavailable_commands` (boolean, default: `false`): If `true`, `knitmit` will immediately report any missing or unconfigured model commands in `model_preferences`. If `false`, errors are deferred until all models have been tried and are only shown if none return a successful response.
* `model_preferences` (array of arrays): A prioritized list of commands to query an LLM.
  * Models are tried in order until one returns a successful response.
  * Each item represents a command and its arguments. The generated prompt is sent to the command’s `stdin`, and the response is read from `stdout`.
  * **Format:** `["command", "arg1", "arg2", ...]`
  * **Example:**
    ```json
    "model_preferences": [
      ["query::gemini", "gemini-2.0-flash"],
      ["ollama", "run", "mistral"],
      ["aichat", "--model", "gpt-4o"],
      ["llm", "-m", "claude-3-haiku"],
      ["sgpt", "--model", "gpt-3.5-turbo"]
    ]
    ```

### Supported model types and commands

* **`query::gemini`**: A built-in function that sends API calls directly to Google's Gemini models using `curl`. Requires `GEMINI_API_KEY` to be set.
  * Example: `["query::gemini", "gemini-2.0-flash"]`
  * Supported models: [https://ai.google.dev/gemini-api/docs/models](https://ai.google.dev/gemini-api/docs/models)
* **Local CLI tools**: Various command-line LLM tools can be used, provided they accept prompts via `stdin` and return responses via `stdout`. You must specify the command and any necessary arguments to choose the model.
  * **`aichat`**: A command-line interface for language models.
    * Example: `["aichat", "--model", "MODEL_NAME"]` or simply `["aichat"]` (uses the default model).
    * Docs: [https://github.com/sigoden/aichat](https://github.com/sigoden/aichat)
  * **`llm`**: Another CLI for interacting with language models.
    * Example: `["llm", "--model", "MODEL_NAME"]`, `["llm", "-m", "MODEL_NAME"]`, or `["llm"]` (uses the default model).
    * Docs: [https://github.com/simonw/llm](https://github.com/simonw/llm)
  * **`sgpt` (ShellGPT)**: A CLI wrapper around LLMs.
    * Example: `["sgpt", "--model", "MODEL_NAME"]` or `["sgpt"]` (uses the default model).
    * Docs: [https://github.com/tbckr/sgpt](https://github.com/tbckr/sgpt)
  * **`ollama`**: A CLI for running models locally via Ollama.
    * Example: `["ollama", "run", "MODEL_NAME"]` (a model name is required).
    * Docs: [https://github.com/ollama/ollama](https://github.com/ollama/ollama)
* **Configuration checks**: For `query::gemini`, `knitmit` checks for a `query::gemini::is_configured` function that verifies whether `GEMINI_API_KEY` is set. You can define similar `<command>::is_configured` bash functions for custom tools. If such a function exists and returns a non-zero status, `knitmit` will skip using that model.

### Commit message guidelines and prompt construction

`knitmit` generates high-quality Git commit messages by creating a structured prompt for your configured LLM. This prompt begins with a concise summary, followed by a detailed explanation formatted using Markdown (limited to bulleted lists prefixed with `*`).

The instructions for commit formatting are drawn from one of two sources:

* If a `knitmit-commit-instructions.txt` file exists in your [configuration directory](#configuration), its contents are used.
* Otherwise, default built-in instructions are applied. You can view these using:
  ```bash
  knitmit commit-instructions
  ```

These instructions cover:

* Summary formatting — character limits, style, and punctuation.
* Body formatting — when to include a body, how to structure it, and Markdown usage.
* Writing principles — clarity, tone, and relevance.
* Output requirements — plain text only, no extra formatting.

To customize these rules, redirect the output to a file and edit as needed:

```bash
knitmit commit-instructions > ~/.config/knitmit-commit-instructions.txt
```

> This example is for Linux. On macOS or Windows, use the appropriate configuration path listed [here](#configuration).

This file will override the built-in instructions whenever prompts are generated.

The full prompt constructed by `knitmit` also includes:

* A summary of recent commit messages to convey project tone and history:
  * Defaults to the last 12 messages, or 4 if using `short`.
* The full staged diff, using:
  * `git diff --diff-algorithm=histogram --cached --function-context -U15` (default)
  * Or `git diff --diff-algorithm=minimal --cached` when `short` is specified

This contextual information ensures the generated commit message is both descriptive and consistent with your project’s style.

## Troubleshooting

* **"The GEMINI_API_KEY environment variable is not set."**  
  Set this environment variable if you intend to use the `query::gemini` function.
* **"No changes have been staged for commit."**  
  Stage your changes using `git add <files>` before running `knitmit`.
* **"The current directory is not part of a Git repository."**  
  Make sure you're inside a valid Git repository. Use `git init` to create one if needed.
* **"jq: command not found" / "curl: command not found"**  
  Install `jq` and `curl` using your system's package manager (e.g., `apt`, `brew`, or `pacman`).
* **"Could not access the clipboard using..."**  
  Ensure a supported clipboard utility is installed and functional — such as `wl-copy`, `xclip`, `pbcopy`, or `clip`.
* **Model command issues ("command not available", "not configured", "failed with a non-zero exit status")**  
  * Confirm the tools listed in your `model_preferences` (e.g., `aichat`, `llm`, `ollama run`) are correctly installed, configured, and available in your `PATH`.
  * Make sure they accept input via `stdin` and return output via `stdout`.
  * Review specific setup steps for each tool — such as downloading models for `ollama`, or setting API keys for cloud-based CLIs.
  * Optionally, enable `report_unavailable_commands: true` in your configuration file to receive immediate feedback when tools are unavailable.

## License

This script is licensed under the Apache License, Version 2.0.
