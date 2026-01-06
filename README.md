# Chat Window Widget for FlowFuse Dashboard (Node-RED Dashboard 2.0)

This repository contains a third-party, node for the Node-RED Dashboard to provide an interactive chat window widget.

Easily build a chat-based user interface, and seamlessly integrate it with the vast collection of other Node-RED nodes available, including many for well-known AI services and offerings like [OpenAI](https://flows.nodered.org/node/@inductiv/node-red-openai-api) or [ollama](https://flows.nodered.org/node/node-red-contrib-ollama).

Below is a demonstration of the chat widget in action, integrated with an OpenAI node to provide a chatbot interface, one trained on 

## Examples

### OpenAI Chatbot

Here, an OpenAI agent was prompted that it is an expert in Node-RED, and should assist users with flow-building and answering general questions:

<img style="max-width:600px; margin: auto;" alt="Screenshot of a FlowFuse Dashboard built to enable interaction with an LLM Agent that can answer questions about Node-RED" src="./docs/screenshot-open-ai.png" />

### Chat & Worldmap Integration

This agent was provisioned to provide coordinate data alongside it's text-based answers, which are then rendered onto a World Map, also rendered within FlowFuse Dashboard.

<img alt="Screenshot of a FlowFuse Dashboard built to enable interaction with an LLM Agent that can answer questions and provide coordinate data, which is then rendered on a World Map" src="./docs/screenshot-worldmap.png" />

## Usage

### Installation

#### Install via Node-RED UI

1. Navigate to a Node-RED Editor
2. Click on the "Manage Palette" option int eh Node-RED menu
3. Switch to the "Install" tab
4. Search for "@flowfuse/node-red-dashboard-2-ui-chat"
5. Click on the "Install" button

#### Install via NPM

```bash
npm install @flowfuse/node-red-dashboard-2-ui-chat
```

### Using in a Flow

<img width="964" alt="Image" src="https://github.com/user-attachments/assets/a6597ab4-bfc8-4358-8c2a-8c4b69ecbeda" />

Anything sent into the node will be considered as a _received_ message. Any messages typed into the chat will be sent out of the node as a _sent_ message.

#### Predefined Input Types

You can create placeholder items in your chat, like a "Typing" message by using different `msg.topic` values.

##### "Typing" Placeholder

By assigning a `msg.topic` of `_typing`, then a placeholder "Typing..." message will be shown in the chat. This is automatically removed when the next message is received.


##### Defining Message Authors

Any other `msg.topic` values used are assumed to be the name of the "author" of the message.

### Message API (inputs and outputs)

The node accepts standard Node-RED messages on its single input and emits messages on its single output when a dashboard user sends a chat message.

#### Inputs (messages into the node)

| Property | Type | Description |
| --- | --- | --- |
| `msg.payload` | `string` | Text to render as a received message. Markdown is supported and sanitized before rendering. |
| `msg.topic` | `string` | Optional author label shown for received messages when **Show author name** is enabled. The special value `_typing` shows a temporary typing indicator until the next message arrives. |

To preload history, send an **array** in `msg.payload` instead of a string. Each array entry should look like:

```json
{
  "text": "Hello there",
  "author": "Bot",
  "sent": false,
  "timestamp": 1715081130000
}
```

- `text` (string) is the message content and supports Markdown formatting.
- `sent` (boolean) controls alignment: `true` shows as a message sent by the user, `false` as received.
- `author` (string, optional) labels the sender when author display is enabled.
- `timestamp` (number, optional) sets the displayed time; defaults to the current time when omitted.

Incoming messages are **not** forwarded to the output—they are stored and displayed in the widget only.

#### Outputs (messages from the dashboard)

When a dashboard user submits a message via the chat input, the node emits:

```json
{
  "topic": "user-message",
  "payload": "text the user typed"
}
```

The `topic` value is fixed to `user-message`; `payload` contains the trimmed text entered by the user. This message will also appear immediately in the widget as a sent message.

## Development

To get started, clone this repository:

```bash
# if using HTTPS:
git clone https://github.com/FlowFuse/node-red-dashboard-2-ui-chat.git

# if using SSH:
git clone git@github.com:FlowFuse/node-red-dashboard-2-ui-chat.git
```

Install dependencies:

```bash
npm install
```

Install the node to your Node-RED instance for development:

```bash
# Change directory to your Node-RED user directory - typically ~/.node-red or c:\Users\<username>\.node-red
cd ~/.node-red
# Install the cloned node package - using the path to where you cloned the repo
npm install /path/to/cloned/repo/node-red-dashboard-2-ui-chat
```

During development, you can use the following command to build the frontend code:

```bash
npm run build:dev
```

For production builds, use:

```bash
npm run build
```

After running a build for front-end changes, you can simply refresh the Node-RED Dashboard to see your changes.

If changes to the Node-RED backend code or the nodes HTML, then you will need to restart Node-RED itself
