# Knitmit

**`Knitmit`** is a portable, self-contained, configurable, and hackable Bash script that streamlines your Git workflow by automatically generating commit messages using Large Language Models (LLMs). It analyzes your staged changes, builds a detailed prompt, queries your configured LLM, and uses the response to craft a commit message — ready to be reviewed or committed as-is.

The script is also intended to be used as a Git subcommand `git knitmit`.

## Features

* **LLM-Powered Commit Messages:** Leverages LLMs to suggest well-formatted and descriptive commit messages.
* **Contextual Prompts:** Generates prompts based on staged Git diffs and recent commit history.
* **Customizable Prompts:** Option to generate a "short" prompt with minimal diff context, useful for large changes or token limits.
* **Flexible LLM Configuration:**
    * Supports multiple LLM backends through a configuration file (`model_preferences`).
    * Tries configured models in order until one succeeds.
    * **Built-in support for Gemini models** via `query::gemini` (uses `curl` and `GEMINI_API_KEY`).
    * **Support for local CLI LLM tools:** Easily integrate with command-line tools that accept a prompt via `stdin` and output the result to `stdout`.
* **Configuration File:** Manage settings via `knitmit.json` in your platform's configuration directory.
* **Clipboard Integration:**
    * Copy the generated prompt to the clipboard.
    * Copy the LLM's response to the clipboard.
* **Platform Aware:** Automatically determines the correct configuration directory and uses platform-specific clipboard tools (`wl-copy`, `xclip`, `pbcopy`, `clip`).
* **Git Integration:**
    * Can directly use the LLM's response as a template for `git commit`.
    * Checks for staged changes and if inside a Git repository.
* **Robust Error Handling:** Provides clear error messages and stack traces for easier debugging.
* **Informative Output:** Keeps the user informed about the steps being taken (e.g., model being queried, clipboard actions).

## Requirements

* **Bash:** The script is written in Bash.
* **Git:** Must be installed and the script must be run within a Git repository.
* **jq:** For parsing JSON responses and configuration. (Install via `sudo apt install jq`, `brew install jq`, etc.)
* **curl:** Used to make requests to the Gemini API when using the built-in `query::gemini`. It is usually pre-installed on most systems.
* **LLM Access & Tools:**
    * For **Gemini models (via `query::gemini`)**:
        * A Google Gemini API key.
        * The `GEMINI_API_KEY` environment variable must be set.
    * For **other local CLI LLM tools** listed in `model_preferences` (e.g., `aichat`, `llm`, `sgpt`, `ollama`):
        * These tools must be installed, configured, and accessible in your system's `PATH`.
        * They must accept the prompt input via `stdin` and output the generated text to `stdout`.
* **Clipboard Tools (Optional but Recommended):**
    * On Linux (Wayland): `wl-copy`
    * On Linux (X11): `xclip`
    * On macOS: `pbcopy`
    * On Windows (Git Bash/MSYS/Cygwin): `clip`
    If no clipboard tool is found, the script will print the content to standard output.

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

If the configuration file is not found, `knitmit` will use the following default configuration:

```json
{
  "commit_with_template": true,
  "copy_prompt": false,
  "copy_response": false,
  "interactive_prompt_limit": 139000,
  "query_language_model": true,
  "report_unavailable_commands": false,
  "model_preferences": [
    ["query::gemini", "gemini-2.5-flash-preview-04-17"],
    ["query::gemini", "gemini-2.0-flash"],
    ["query::gemini", "gemini-2.0-flash-lite"],
    ["aichat"],
    ["llm"],
    ["sgpt"]
  ]
}
```

### Configuration Options:

* `commit_with_template` (boolean, default: `true`): If `true`, after getting the LLM response, `knitmit` will run `git commit --template <(response)` to open your editor with the suggested message.
* `copy_prompt` (boolean, default: `false`): If `true`, the generated prompt will be copied to the clipboard *before* querying the LLM.
* `copy_response` (boolean, default: `false`): If `true`, the LLM's response will be copied to the clipboard.
* `interactive_prompt_limit` (integer, default: `139000`): Maximum length of the prompt considered suitable for interactive use (e.g., pasting into a web UI). A warning is shown if this limit is exceeded when `copy_prompt` is active or the `copy` command is used.
* `query_language_model` (boolean, default: `true`): If `false`, the script will not query any LLM. Useful if you only want to generate and copy the prompt.
* `report_unavailable_commands` (boolean, default: `false`): If `true`, `knitmit` will immediately report if a model command in `model_preferences` is not found or not configured. If `false`, these messages are deferred until after all models have been attempted.
* `model_preferences` (array of arrays): A list of commands to try for querying an LLM.
    * `knitmit` will attempt to use these models in the order they are listed. The first successful response is used.
    * Each item in the array represents a command and its arguments. The script will pipe the generated prompt to the `stdin` of this command and expect the commit message suggestion on `stdout`.
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

### Supported Model Types/Commands:

* **`query::gemini`**: This is a special built-in function that sends API calls directly to Google's Gemini models using `curl`. It requires `GEMINI_API_KEY` to be set.
    * Example: `["query::gemini", "gemini-2.0-flash"]`
    * Available Gemini model names: [https://ai.google.dev/gemini-api/docs/models](https://ai.google.dev/gemini-api/docs/models)
* **Local CLI Tools (e.g., `aichat`, `llm`, `sgpt`, `ollama`):** Many command-line LLM tools can be used if they adhere to the `stdin` for prompt and `stdout` for response convention. You just need to specify the command and any necessary arguments to select the model.
    * **`aichat`**: Command-line interface for language models.
        * Example: `["aichat", "--model", "MODEL_NAME"]` or `["aichat"]` (for default).
        * More details: [https://github.com/sigoden/aichat](https://github.com/sigoden/aichat)
    * **`llm`**: Command-line interface for language models.
        * Example: `["llm", "--model", "MODEL_NAME"]` or `["llm", "-m", "MODEL_NAME"]` or `["llm"]` (for default).
        * More details: [https://github.com/simonw/llm](https://github.com/simonw/llm)
    * **`sgpt` (ShellGPT)**: Command-line interface for language models.
        * Example: `["sgpt", "--model", "MODEL_NAME"]` or `["sgpt"]` (for default).
        * More details: [https://github.com/tbckr/sgpt](https://github.com/tbckr/sgpt)
    * **`ollama`**: Command-line interface for running models locally with Ollama.
        * Example: `["ollama", "run", "MODEL_NAME"]` (A model name must be specified).
        * More details: [https://github.com/ollama/ollama](https://github.com/ollama/ollama)
* For `query::gemini`, `knitmit` also checks for the `query::gemini::is_configured` function (which checks `GEMINI_API_KEY`). You can create similar `<command>::is_configured` bash functions in your environment if your custom tools require specific setup checks before being called. If this function exists and returns a non-zero status, `knitmit` will skip that model.

## Usage

Run `knitmit` (or `git knitmit`) from within a Git repository with staged changes.

```bash
# Basic usage: Generate message and open commit editor with the suggestion
knitmit

# Generate a shorter prompt (minimal diff), then generate message and open commit editor
knitmit short

# Generate the full prompt and copy it to the clipboard, then exit (no LLM query)
knitmit copy

# Generate a short prompt, copy it to clipboard, then exit
knitmit short copy

# Generate prompt, query LLM, copy the LLM response to clipboard, then exit (no commit)
knitmit result

# Generate short prompt, query LLM, copy response to clipboard, then exit
knitmit short result

# Display help message
knitmit help
knitmit -h
knitmit --help
```

## How the Prompt is Constructed

The `knitmit::make_prompt` function generates a detailed prompt for the LLM, beginning with a concise summary line followed by a more in-depth explanation. The body of the prompt uses Markdown formatting, but restricts it to bulleted lists prefixed with `*`.

The prompt provides context on the project's current state and style:

* A summary of recent commit messages to establish project history and tone:
  * By default, includes the last 12 commit messages.
  * If `short` is specified, includes the last 4 commit messages.
* The full output of `git diff --cached` to capture staged changes:
  * If `short` is used: `git diff --diff-algorithm=minimal --cached`.
  * Otherwise: `git diff --diff-algorithm=histogram --cached --function-context -U15`.

## Troubleshooting

* **"The GEMINI_API_KEY environment variable is not set."**: Make sure you have set this environment variable if you intend to use the `query::gemini` function.
* **"No changes have been staged for commit."**: Stage your changes using `git add <files>` before running `knitmit`.
* **"The current directory is not part of a Git repository."**: Navigate to a Git repository.
* **"jq: command not found" / "curl: command not found"**: Install `jq` and `curl` using your system's package manager.
* **"Could not access the clipboard using..."**: Ensure one of the supported clipboard utilities (`wl-copy`, `xclip`, `pbcopy`, `clip`) is installed and working.
* **Model command issues ("command not available", "not configured", "failed with a non-zero exit status")**:
    * Verify that the LLM tools specified in your `model_preferences` (e.g., `aichat`, `llm`, `ollama run`) are installed correctly, configured, and are in your `PATH`.
    * Ensure they accept prompts via `stdin` and output to `stdout`.
    * Check their specific configuration requirements (e.g., model downloads for `ollama`, API keys for cloud-based tools wrapped by these CLIs).
    * You can enable `report_unavailable_commands: true` in your config for more immediate feedback.

## License

This script is licensed under the Apache License, Version 2.0.
