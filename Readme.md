# Code Search Agent

A LangChain agent that answers natural-language questions about a GitHub repository's code — built with Google Gemini and LangChain's GitHub toolkit, running in Google Colab.

This notebook is configured to explore [`kavishannip/Ai-Diagrams`](https://github.com/kavishannip/Ai-Diagrams), but works with any repo your GitHub App is installed on.

## What it does

Ask questions like:

- "Search the repo for where `DiagramResult` is defined and tell me the file path."
- "How does user authentication work in this repo? Explain the process."
- "How does the diagram generation process happen, from user prompt to generated diagram?"

The agent reasons over the codebase step by step — searching for relevant files, reading their contents, and synthesizing an explanation grounded in what it actually found (not guessed).

## Stack

- **LLM**: Google Gemini (`gemini-3.1-flash-lite`) via `langchain-google-genai`
- **Agent framework**: `langchain.agents.create_agent`
- **GitHub access**: `langchain_community`'s `GitHubToolkit`, authenticated via a GitHub App
- **Runtime**: Google Colab

## Tools available to the agent

- **`search_code`** — searches code in the repo via GitHub's Search API
- **`read_file`** — reads the full contents of a specific file

(Renamed from the toolkit's defaults `"Search code"` / `"Read File"` — Gemini's function-calling API rejects tool names containing spaces.)

## Setup

1. **Create a GitHub App** at [github.com/settings/apps/new](https://github.com/settings/apps/new)
   - Give it Read-only access to **Contents** (and optionally Issues, Pull requests)
   - Generate a private key (`.pem` file) from the app's settings page
   - Install the app on your account (choose "All repositories")

2. **Get a Gemini API key** at [aistudio.google.com/apikey](https://aistudio.google.com/apikey)

3. **In Colab**, set these as secrets (🔑 icon in the sidebar):
   - `GOOGLE_API_KEY`
   - `GITHUB_APP_ID`

4. Run the upload cell and select your `.pem` private key file when prompted.

5. Set `GITHUB_REPOSITORY` to any repo your GitHub App is installed on — this can be a repo you own, or your own fork of a public repo.

## Notes / limitations

- A GitHub App can only access repos it's installed on — to explore someone else's public repo, fork it into your own account first.
- `search_code` returns at most the top 5 results per query — it's a text search, not a call-graph or AST-aware search, so questions like "find every caller of X" may be incomplete.
- LangChain v1.0 removed `hub.pull()` and the classic `create_react_agent` + `AgentExecutor` pattern; this notebook uses the current `langchain.agents.create_agent` API instead, with a plain `system_prompt` string.

## Example

```python
response = agent.invoke({"messages": [("user", "How Diagram generation process is happened, from user prompt to generate diagram")]})

print(response["messages"][-1].content)
```

**Output:**

> The diagram generation process in this application follows a structured flow from user input to the final rendered image. Based on the codebase, here is the step-by-step breakdown:
>
> ### 1. User Input
>
> The process begins when a user provides input through one of two primary methods:
>
> - **Text Prompt:** The user describes a diagram in natural language (e.g., "Create a sequence diagram for user login").
> - **Image Upload:** The user uploads an image file that contains a diagram.
>
> ### 2. AI Processing (Generation)
>
> The application uses Google's Gemini AI models to interpret the user's input and generate the necessary code to create the diagram.
>
> - **Text-to-PlantUML:** The text prompt is sent to an API endpoint (e.g., `/api/text-to-plantuml`) where the Gemini model (typically Gemini 2.5 Flash for speed) analyzes the request and generates the corresponding **PlantUML code**.
> - **Image-to-PlantUML:** The uploaded image is sent to an API endpoint (e.g., `/api/img-to-plantuml`) where the Gemini model (typically Gemini 2.5 Pro for better visual understanding) analyzes the image and converts it into **PlantUML code**.
>
> ### 3. Code Handling & Refinement
>
> Once the PlantUML code is generated, it is passed to the frontend:
>
> - **Event-Driven Updates:** The application uses custom browser events (`diagramGenerated`, `diagramUpdated`) to pass the generated PlantUML code between components (e.g., from the upload/prompt component to the `DiagramResult` or `ImgDiagramResult` display components).
> - **AI Fix:** If the generated code contains syntax errors, the user can trigger an "AI Fix" feature, which sends the faulty code to an API endpoint (`/api/plantuml-ai-fix`) to have the AI repair the syntax.
>
> ### 4. Diagram Rendering
>
> The frontend is responsible for turning the PlantUML code into a visual diagram:
>
> - **Theme Application:** The application prepends a theme directive (e.g., `!theme cyborg`) to the PlantUML code based on the user's selected theme.
> - **Encoding:** The PlantUML code is encoded using a specific algorithm (deflate compression followed by a base64-like encoding) to make it suitable for transmission.
> - **Image Generation:** The encoded code is sent to the `/api/plantuml-to-img` endpoint. This endpoint interacts with a PlantUML server to render the code into an image format (SVG or PNG).
> - **Display:** The resulting image (usually an SVG) is returned to the frontend, converted into a `Blob` URL, and displayed in the UI.
>
> ### 5. Interaction & Management
>
> Once the diagram is rendered, the user can:
>
> - **Manipulate:** Zoom, pan, and change themes (which triggers a re-render).
> - **Save:** Use the `SaveButton` component to persist the diagram (code and title) to a MongoDB database via server actions.
> - **Download:** Export the diagram as an SVG or PNG file.
>
> ### Summary Flowchart
>
> 1. **User Prompt/Image** → **Gemini AI** (via API)
> 2. **Gemini AI** → **PlantUML Code**
> 3. **PlantUML Code** → **Frontend** (via custom events)
> 4. **Frontend** → **PlantUML Server** (via `/api/plantuml-to-img`)
> 5. **PlantUML Server** → **SVG/PNG Image** → **User UI**

This shows the agent doing more than keyword lookup — it searched multiple files, cross-referenced API routes and component names, and synthesized an accurate end-to-end explanation of a multi-step feature.

## Seeing the agent's reasoning

```python
for step in agent.stream(
    {"messages": [("user", "How does user authentication work in this repo?")]},
    stream_mode="values"
):
    step["messages"][-1].pretty_print()
```

This streams each intermediate step — tool calls, tool results, and the final answer — instead of only the final response.
