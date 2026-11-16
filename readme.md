# Generative AI Chat App with Microsoft Foundry

A console chat application, written in Python, that talks to a large language model deployed in a Microsoft Foundry project. It uses the OpenAI SDK and the Responses API, signs in with Microsoft Entra ID instead of an API key, remembers the conversation, and streams answers as they are generated.

I built it by working through the [Create a generative AI chat app](Instructions/Exercises/03-foundry-sdk.md) exercise from Microsoft Learn as part of my AI-103 studies. This repository is my fork of [MicrosoftLearning/mslearn-ai-studio](https://github.com/MicrosoftLearning/mslearn-ai-studio). The completed code is in [labfiles/foundry-chat/python/chat-app](labfiles/foundry-chat/python/chat-app).

## What it does

- **Chats with a deployed model from the terminal.** You type a prompt, the model answers, and the loop continues until you type `quit`.
- **Signs in with Microsoft Entra ID.** There is no API key anywhere in the code or the configuration.
- **Uses the Responses API**, the newer alternative to Chat Completions.
- **Remembers the conversation.** Each request carries the ID of the previous response, so a follow-up such as "How does it compare to modern LLMs?" is understood.
- **Streams the answer.** Text appears piece by piece instead of after a long pause.
- **Includes an asynchronous version** built on `AsyncOpenAI`.

## Project files

All of these are in [labfiles/foundry-chat/python/chat-app](labfiles/foundry-chat/python/chat-app).

| File | Purpose |
| --- | --- |
| [chat-app.py](labfiles/foundry-chat/python/chat-app/chat-app.py) | The main app: Responses API, conversation tracking and streaming |
| [chat-async.py](labfiles/foundry-chat/python/chat-app/chat-async.py) | The same chat using the asynchronous client |
| [requirements.txt](labfiles/foundry-chat/python/chat-app/requirements.txt) | Python packages: `openai`, `azure-identity`, `python-dotenv`, `aiohttp` |
| [.env.example](labfiles/foundry-chat/python/chat-app/.env.example) | Template for the two settings the app needs |

## How it works

### 1. Configuration

The app reads two settings from a `.env` file with `python-dotenv`: the endpoint to call and the name of the model deployment.

```python
load_dotenv()
azure_openai_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
model_deployment = os.getenv("MODEL_DEPLOYMENT")
```

### 2. Signing in without an API key

`DefaultAzureCredential` picks up the identity you signed in with through `az login`. `get_bearer_token_provider` turns that into a function that fetches a fresh access token whenever one is needed, and the OpenAI client accepts that function in place of an API key.

```python
token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://ai.azure.com/.default"
)

openai_client = OpenAI(
    base_url=azure_openai_endpoint,
    api_key=token_provider
)
```

### 3. Asking the model

The Responses API takes the system message as `instructions` and the user's prompt as `input`. This is simpler than Chat Completions, which needs a list of role-tagged messages.

### 4. Remembering the conversation

On its own, each request is independent, so the model has no idea what "it" refers to in a follow-up question. The fix is to keep the ID of the last response and send it with the next request:

```python
last_response_id = None   # before the loop
...
previous_response_id=last_response_id,
```

The memory lasts for one run of the app. Starting the app again starts a new conversation.

### 5. Streaming the answer

With `stream=True` the call returns a series of events instead of one finished response. The app prints each chunk of text as it arrives, and saves the response ID from the final event for the next turn.

```python
stream = openai_client.responses.create(
    model=model_deployment,
    instructions="You are a helpful AI assistant that answers questions and provides information.",
    input=input_text,
    previous_response_id=last_response_id,
    stream=True
)
for event in stream:
    if event.type == "response.output_text.delta":
        print(event.delta, end="")
    elif event.type == "response.completed":
        last_response_id = event.response.id
print()
```

### 6. The asynchronous version

[chat-async.py](labfiles/foundry-chat/python/chat-app/chat-async.py) does the same job with `AsyncOpenAI` and the async credential from `azure.identity.aio`. The request is awaited, the whole answer is printed at once, and the credential is closed in a `finally` block when the app ends.

## How the app was built up

The exercise builds the app in stages, each replacing the one before. The repository holds the final stage.

| Stage | Change | Result |
| --- | --- | --- |
| 1. Chat Completions | `chat.completions.create` with a system and a user message | One question, one answer |
| 2. Responses API | `responses.create` with `instructions` and `input` | Same answer from a simpler call, but follow-up questions lose their context |
| 3. Conversation tracking | Add `previous_response_id` | Follow-up questions work |
| 4. Streaming | Add `stream=True` and handle the events | The answer appears as it is generated |
| 5. Async | `AsyncOpenAI` with `await` | The same chat on the asynchronous client |

## Run it yourself

### Prerequisites

- An Azure subscription
- A Microsoft Foundry project with a chat model deployed (create both at <https://ai.azure.com>)
- Python 3.13
- Git
- Azure CLI

### Steps

1. Clone the repository and go to the app folder:

   ```
   git clone https://github.com/RafidSarowar/mslearn-ai-studio.git
   cd mslearn-ai-studio/labfiles/foundry-chat/python/chat-app
   ```

2. Install the packages:

   ```
   pip install -r requirements.txt
   ```

3. Create your own `.env` from the template:

   ```
   cp .env.example .env
   ```

   On Windows PowerShell use `Copy-Item .env.example .env`. Then open `.env` and fill in the two values described under [Configuration](#configuration).

4. Sign in to Azure:

   ```
   az login
   ```

5. Start the app:

   ```
   python chat-app.py
   ```

6. Wait for the `Enter a prompt` line, then type your questions there. Type `quit` to stop.

To try the asynchronous version, run `python chat-async.py` instead.

### Example session

```
Enter a prompt (or type "quit" to exit): Tell me about the ELIZA chatbot.
ELIZA is an early natural-language computer program created in 1966 by Joseph
Weizenbaum at MIT that simulated conversation using pattern-matching and simple
substitution rules. ...

Enter a prompt (or type "quit" to exit): How does it compare to modern LLMs?
Whereas ELIZA used simple pattern-matching and scripted rules with no learning
or real understanding, modern LLMs are large neural-network models trained on
vast text corpora ...

Enter a prompt (or type "quit" to exit): quit
```

## Configuration

| Setting | What to put in it |
| --- | --- |
| `AZURE_OPENAI_ENDPOINT` | The **Azure OpenAI endpoint** from the home page of your Foundry project, not the project endpoint. It must end in `/openai/v1/`. |
| `MODEL_DEPLOYMENT` | The exact name of your model deployment, as shown in the Foundry portal. |

## Keeping the endpoint private

- `.env` is listed in [.gitignore](.gitignore), so the file holding a real endpoint is never committed. Only `.env.example`, which contains placeholders, is in the repository.
- The app has no API key to leak. Access is granted to the Azure identity you sign in with, and tokens are fetched at run time.
- No endpoint, resource name or subscription detail appears in this README or in the code.

## Troubleshooting

These are the problems I ran into while building it.

| What you see | Cause | Fix |
| --- | --- | --- |
| `DefaultAzureCredential failed to retrieve a token` | Azure CLI is not installed, or you are not signed in | Install Azure CLI, then run `az login` |
| `az` is not recognized straight after installing it | The terminal was opened before the install | Close and reopen VS Code or the terminal |
| The request fails although sign-in worked | The endpoint ends in `/responses`. The SDK adds that part itself, so the address becomes `/responses/responses` | End the endpoint at `/openai/v1/` |
| `SyntaxError: expected 'except' or 'finally' block`, or an `IndentationError` | Pasted code is not indented to match the block it belongs to | Select the pasted lines and press Tab until they line up with the `# Get a response` comment |
| `The term 'Tell' is not recognized` | The prompt was typed into PowerShell, not into the app | Start the app first, then type at the `Enter a prompt` line |
| A follow-up question gets "I don't know what 'it' refers to" | It was the first question of a new run, or `previous_response_id` is missing | Ask the first question in the same run, and check stage 3 above |
| `'charmap' codec can't encode character` | Output is being piped or redirected on Windows | Set `PYTHONIOENCODING=utf-8`, or run in a normal terminal |

## What I learned

- How to connect to a model in Microsoft Foundry with the OpenAI SDK by pointing `base_url` at the Azure OpenAI endpoint.
- How Entra ID authentication replaces API keys, using `DefaultAzureCredential` and a bearer token provider.
- The difference between the Chat Completions API and the Responses API.
- How `previous_response_id` carries context from one turn to the next.
- How to handle streamed events so that long answers appear immediately.
- How the asynchronous client differs from the synchronous one, including closing the credential at the end.
- That indentation is part of Python's syntax, and that a prompt for the app and a command for the terminal are typed in different places.

## Cleaning up

The model deployment is billed per use, so it costs nothing while idle. When you have finished, delete the Foundry resource in the [Azure portal](https://portal.azure.com) to remove the project and the deployment.

## Tested with

Python 3.13.7, `openai` 3.51.0, `azure-identity` 1.27.0 and `python-dotenv` 1.2.4 on Windows 11, against a `gpt-5-mini` deployment.

## About the original repository

This is a fork of [MicrosoftLearning/mslearn-ai-studio](https://github.com/MicrosoftLearning/mslearn-ai-studio), *Develop Generative AI Solutions in Azure*: a set of hands-on exercises covering common tasks in building generative AI solutions on Microsoft Azure. The exercise instructions and starter files in this repository are Microsoft's, and the rest of the labs are still here unchanged.

- Read the exercises on the [GitHub Pages site for the original repo](https://go.microsoft.com/fwlink/?linkid=2310724).
- The exercises complement the modules on [Microsoft Learn](https://aka.ms/mslearn-generative-ai).
- Problems with the exercises themselves should be reported as issues in the original repository.

The repository is released under the [MIT License](LICENSE).
