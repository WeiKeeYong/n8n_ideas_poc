# n8n Proof of Concept: Telegram-Controlled Automated Gate System with Raspberry Pi

This Proof of Concept (PoC) showcases an innovative automation workflow that integrates **n8n**, **Telegram**, and a **Raspberry Pi** to remotely control a gate system via text or voice commands. By leveraging n8n's powerful workflow automation, Telegram's messaging capabilities, and the Raspberry Pi's hardware control, this project enables seamless gate operation for pedestrians, cyclists, or vehicles. Whether you're opening a side gate for a bike or both gates for a car, this system provides a flexible and user-friendly solution.

Before diving into the setup, ensure you have the following prerequisites:

- **Basic Knowledge**: Familiarity with electronics, Python programming, and Raspberry Pi configuration.
- **OpenAI API Account**: Required for AI-powered text and voice message processing.
- **Required Libraries**: Install necessary Python libraries on your Raspberry Pi (e.g., `RPi.GPIO`, `Flask`).
- **Hardware**: A Raspberry Pi, relays (e.g., Generic HL-52S), and a gate remote system wired to the relays.

Special thanks to [this YouTube tutorial](https://www.youtube.com/watch?v=-dJOa_MSwsk) for inspiration on handling voice and text inputs.

---

## System Overview

This PoC enables users to send commands via Telegram (e.g., "Open gate for my car" or "Close gate") to control a gate system. The workflow interprets these commands using AI, triggers the appropriate Raspberry Pi relay, and sends a confirmation back to the user. The system supports:

- **Text and Voice Inputs**: Users can send text messages or voice notes via Telegram.
- **Context-Aware Gate Control**: The AI distinguishes between requests for side gate (for bikes, pedestrians) or both gates (for cars, lorries).
- **Polite Error Handling**: Non-gate-related requests (e.g., "What's the weather?") receive a friendly response redirecting focus to gate control.

The workflow is visualized below:

![Gate Control Workflow](https://github.com/WeiKeeYong/n8n_ideas_poc/raw/main/images/n8n_open_gate.jpg)

---

## Workflow Details

The n8n workflow orchestrates the entire process, from receiving Telegram messages to triggering the Raspberry Pi relays. Below is a detailed breakdown of the workflow, followed by the JSON configuration and the Python script for the Raspberry Pi.

### Key Components

1. **Telegram Trigger**: Listens for incoming messages (text or voice) from the Telegram bot.
2. **Switch Node**: Routes messages based on whether they are text or voice inputs.
3. **Voice Processing**: Converts voice messages to text using OpenAI's transcription service.
4. **AI Agent**: Analyzes user requests, selects the appropriate gate control action, and responds in the user's language.
5. **HTTP Requests**: Sends commands to the Raspberry Pi to activate relays for opening/closing gates.
6. **Memory Buffer**: Maintains a short conversation history for context-aware responses.
7. **Telegram Response**: Sends confirmation messages back to the user (e.g., "The gate is now open for your car.").

---


<details>
  <summary>n8n Workflow JSON Configuration</summary>

```json
{
  "name": "Telegram Open Gate",
  "nodes": [
    {
      "parameters": {
        "updates": [
          "message"
        ],
        "additionalFields": {}
      },
      "type": "n8n-nodes-base.telegramTrigger",
      "typeVersion": 1.2,
      "position": [
        -780,
        40
      ],
      "id": "f51970ae-137e-4c33-83a6-3a3915f4e079",
      "name": "Telegram Trigger",
      "webhookId": "7e48c5d1-df3c-490a-97de-d86305ba56b0",
      "credentials": {
        "telegramApi": {
          "id": "iDfndOKRyeylXBA6",
          "name": "Open_Sesame_bot"
        }
      }
    },
    {
      "parameters": {
        "rules": {
          "values": [
            {
              "conditions": {
                "options": {
                  "caseSensitive": true,
                  "leftValue": "",
                  "typeValidation": "strict",
                  "version": 2
                },
                "conditions": [
                  {
                    "leftValue": "={{ $json.message.voice.file_id }}",
                    "rightValue": "",
                    "operator": {
                      "type": "string",
                      "operation": "notExists",
                      "singleValue": true
                    },
                    "id": "23181306-8f9f-430c-8d13-2d80130f6aef"
                  }
                ],
                "combinator": "and"
              },
              "renameOutput": true,
              "outputKey": "Text"
            },
            {
              "conditions": {
                "options": {
                  "caseSensitive": true,
                  "leftValue": "",
                  "typeValidation": "strict",
                  "version": 2
                },
                "conditions": [
                  {
                    "id": "90b49fad-92b1-406c-9953-1cc165511595",
                    "leftValue": "={{ $json.message.voice.file_id }}",
                    "rightValue": "",
                    "operator": {
                      "type": "string",
                      "operation": "exists",
                      "singleValue": true
                    }
                  }
                ],
                "combinator": "and"
              },
              "renameOutput": true,
              "outputKey": "Voice"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.switch",
      "typeVersion": 3.2,
      "position": [
        -560,
        40
      ],
      "id": "2ac8a89f-f3d6-4274-a643-5558148be239",
      "name": "Switch"
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "={{ $json.message?.text || $json.text }}",
        "options": {
          "systemMessage": "=<prompt>\n    <current_date_time>{{ $now }}</current_date_time>\n    <state>\n        You control the auto gate and execute user requests to open or close the gates.\n    </state>\n    <instructions>\n        1. Analyze the user's request for gate actions.\n        2. Select the correct tool:\n           - Use HTTP_Request_channel01 to open the side gate. (For bikes, humans, bicycles, or if not specified)\n           - Use HTTP_Request_channel02 to open both gates. (For cars, lorries, or if the user explicitly requests both gates open)\n        3. When closing any gate, always use HTTP_Request_channel01 (closing function is always on channel01).\n        4. Only respond to gate control requests. If the user asks for something unrelated (e.g., weather, news), politely reply you can only help with the gate.\n        5. Respond in the user's language.\n        6. Respond clearly and concisely.\n    </instructions>\n    <examples>\n      <example>\n        <input>Hi</input>\n        <output>Hello! Do you want to open or close the gate?</output>\n      </example>\n      <example>\n        <input>Open gate for my bike</input>\n        <action>\n          <tool>HTTP_Request_channel01</tool>\n          <message>The gate is now open for your bike.</message>\n        </action>\n      </example>\n      <example>\n        <input>Open me gate</input>\n        <action>\n          <tool>HTTP_Request_channel01</tool>\n          <message>The gate is now open.</message>\n        </action>\n      </example>\n      <example>\n        <input>Close gate</input>\n        <action>\n          <tool>HTTP_Request_channel01</tool>\n          <message>The gate is now closed.</message>\n        </action>\n      </example>\n      <example>\n        <input>Open gate for my car</input>\n        <action>\n          <tool>HTTP_Request_channel02</tool>\n          <message>The gate is now open for your car.</message>\n        </action>\n      </example>\n      <example>\n        <input>I have a lorry coming, open the gate</input>\n        <action>\n          <tool>HTTP_Request_channel02</tool>\n          <message>The gate is now open for your lorry.</message>\n        </action>\n      </example>\n      <example>\n        <input>What’s the weather?</input>\n        <output>I can only help you open or close the gate.</output>\n      </example>\n      <example>\n        <input>What is today news?</input>\n        <output>I can only help you open or close the gate.</output>\n      </example>\n      <example>\n        <input>Do you like to eat?</input>\n        <output>Hi, let's focus on gate control. Do you want to open or close the gate?</output>\n      </example>\n    </examples>\n</prompt>"
        }
      },
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.8,
      "position": [
        240,
        -100
      ],
      "id": "76c4089b-54c5-400c-bb59-e47d5bdbc8b8",
      "name": "AI Agent"
    },
    {
      "parameters": {
        "resource": "audio",
        "operation": "transcribe",
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.openAi",
      "typeVersion": 1.8,
      "position": [
        -80,
        200
      ],
      "id": "7ccda183-317d-4d0f-b04c-4882028b3eef",
      "name": "OpenAI",
      "credentials": {
        "openAiApi": {
          "id": "xmvnrIlp6k6U5Eyt",
          "name": "OpenAi account"
        }
      }
    },
    {
      "parameters": {
        "resource": "file",
        "fileId": "={{ $json.message.voice.file_id }}"
      },
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        -300,
        200
      ],
      "id": "b36bdd39-9208-4342-86c6-9f42b63179a3",
      "name": "Telegram",
      "webhookId": "9dfbf0e8-ca74-48f6-828a-84d6ad6fd83a",
      "credentials": {
        "telegramApi": {
          "id": "iDfndOKRyeylXBA6",
          "name": "Open_Sesame_bot"
        }
      }
    },
    {
      "parameters": {
        "model": {
          "__rl": true,
          "mode": "list",
          "value": "gpt-4o-mini"
        },
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1.2,
      "position": [
        200,
        180
      ],
      "id": "a4ce9573-0897-40ad-9273-89113b1f5c5d",
      "name": "OpenAI Chat Model",
      "credentials": {
        "openAiApi": {
          "id": "xmvnrIlp6k6U5Eyt",
          "name": "OpenAi account"
        }
      }
    },
    {
      "parameters": {
        "chatId": "={{ $('Telegram Trigger').item.json.message.chat.id }}",
        "text": "={{ $json.output }}",
        "additionalFields": {
          "appendAttribution": false
        }
      },
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        600,
        -100
      ],
      "id": "da13465e-dd09-4e81-9b3c-98b068ab3eb2",
      "name": "Telegram1",
      "webhookId": "78255e50-4214-4899-bd71-3f55406b8f48",
      "credentials": {
        "telegramApi": {
          "id": "iDfndOKRyeylXBA6",
          "name": "Open_Sesame_bot"
        }
      }
    },
    {
      "parameters": {
        "sessionIdType": "customKey",
        "sessionKey": "={{ $('Telegram Trigger').item.json.message.from.id }}",
        "contextWindowLength": 2
      },
      "type": "@n8n/n8n-nodes-langchain.memoryBufferWindow",
      "typeVersion": 1.3,
      "position": [
        340,
        120
      ],
      "id": "2488df91-5fd2-402b-b631-abe6db2d4b75",
      "name": "Simple Memory"
    },
    {
      "parameters": {
        "toolDescription": "Call this tool for Open side gate or close gate",
        "method": "POST",
        "url": "http://127.0.0.1:881/1"
      },
      "type": "@n8n/n8n-nodes-langchain.toolHttpRequest",
      "typeVersion": 1.1,
      "position": [
        840,
        240
      ],
      "id": "8256dee1-41f1-4389-9ded-eb17b466c617",
      "name": "HTTP_Request_channel01"
    },
    {
      "parameters": {
        "toolDescription": "Call this tool to open Both the gate or for car, lorry, large vehicles",
        "method": "POST",
        "url": "http://127.0.0.1:881/2"
      },
      "type": "@n8n/n8n-nodes-langchain.toolHttpRequest",
      "typeVersion": 1.1,
      "position": [
        480,
        300
      ],
      "id": "49363f48-45ae-4c53-a154-0c34426c432c",
      "name": "HTTP_Request_channel02"
    }
  ],
  "pinData": {},
  "connections": {
    "Telegram Trigger": {
      "main": [
        [
          {
            "node": "Switch",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Switch": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Telegram",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Telegram": {
      "main": [
        [
          {
            "node": "OpenAI",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "OpenAI": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "OpenAI Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Telegram1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Simple Memory": {
      "ai_memory": [
        [
          {
            "node": "AI Agent",
            "type": "ai_memory",
            "index": 0
          }
        ]
      ]
    },
    "HTTP_Request_channel01": {
      "ai_tool": [
        [
          {
            "node": "AI Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    },
    "HTTP_Request_channel02": {
      "ai_tool": [
        [
          {
            "node": "AI Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "versionId": "d1f669fa-2f58-4e61-908d-f27fecb2e973",
  "meta": {
    "templateCredsSetupCompleted": true,
    "instanceId": "205c45a2735bdbcb74d87e3d381222fbdd19e110964a53d0d0ea9af4012c5a91"
  },
  "id": "m07FLP16C6wOMRYz",
  "tags": []
}
```
</details>
  
<details>
<summary>Raspberry Pi Python Script</summary>
This Python script runs on the Raspberry Pi to control the gate relays via HTTP requests. It uses the Flask framework to create a simple web server and the RPi.GPIO library to manage the relays. Ensure you have the required libraries installed and basic Python knowledge.


```python
#using Pin 11 and Pin 13 in rasberry. For some reason, High is off. And Relay i am using Generic HL-52s Google for schematic. 
#Gate remote are basic remote that i tap the button switch to the relay. Please adjust the duration, depending your relay sensitivity.

import time
import threading
import RPi.GPIO as GPIO
from flask import Flask

app = Flask(__name__)
GPIO.setmode(GPIO.BCM)

GPIO.setup(17, GPIO.OUT)  # Relay 1
GPIO.setup(27, GPIO.OUT)  # Relay 2

GPIO.output(17, GPIO.HIGH)
GPIO.output(27, GPIO.HIGH)

duration = 1.2

def activate_relay(pin):
    GPIO.output(pin, GPIO.LOW)  # Turn the relay on
    time.sleep(duration)        # Wait for the specified time
    GPIO.output(pin, GPIO.HIGH)  # Turn the relay off

@app.route('/1', methods=['POST','GET'])
def relay1_on():
    thread = threading.Thread(target=activate_relay, args=(17,))
    thread.daemon = True  # Make the thread
    thread.start()
    return "Relay 1 activated", 200

@app.route('/2', methods=['POST','GET'])
def relay2_on():
    thread = threading.Thread(target=activate_relay, args=(27,))
    thread.daemon = True  # Make as thread
```
</details>
