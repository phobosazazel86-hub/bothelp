{
  "name": "Telegram JS Bot Ready",
  "nodes": [
    {
      "parameters": {
        "updates": [
          "message",
          "callback_query"
        ],
        "additionalFields": {}
      },
      "id": "1",
      "name": "Telegram Trigger",
      "type": "n8n-nodes-base.telegramTrigger",
      "typeVersion": 1,
      "position": [
        260,
        300
      ],
      "credentials": {
        "telegramApi": {
          "id": "",
          "name": "Telegram account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "jsCode": "const replyKeyboard = {\n  keyboard: [\n    [{ text: 'Меню' }, { text: 'Помощь' }],\n    [{ text: 'Калькулятор' }, { text: 'Ссылки' }],\n  ],\n  resize_keyboard: true,\n};\n\nconst inlineKeyboard = {\n  inline_keyboard: [\n    [\n      { text: 'Нажми меня', callback_data: 'press_me' },\n      { text: 'Открыть сайт', url: 'https://telegram.org' },\n    ],\n    [{ text: 'Назад в меню', callback_data: 'back_to_menu' }],\n  ],\n};\n\nfunction evaluateExpression(expression) {\n  const normalized = (expression ?? '').replace(/\\s+/g, '');\n\n  if (!normalized) {\n    throw new Error('Empty expression');\n  }\n\n  if (!/^[0-9+\\-*/().]+$/.test(normalized)) {\n    throw new Error('Unsupported characters');\n  }\n\n  if (!/[+\\-*/]/.test(normalized)) {\n    throw new Error('No operator');\n  }\n\n  const result = Function(`\"use strict\"; return (${normalized});`)();\n  if (!Number.isFinite(result)) {\n    throw new Error('Invalid result');\n  }\n\n  return result;\n}\n\nreturn items.map((item) => {\n  const update = item.json.body ?? item.json;\n  const message = update.message ?? null;\n  const callbackQuery = update.callback_query ?? null;\n  const chatId = message?.chat?.id ?? callbackQuery?.message?.chat?.id ?? null;\n  const userId = message?.from?.id ?? callbackQuery?.from?.id ?? null;\n  const messageId = message?.message_id ?? callbackQuery?.message?.message_id ?? null;\n  const text = (message?.text ?? callbackQuery?.data ?? '').trim();\n  const lowerText = text.toLowerCase();\n\n  const output = {\n    chatId,\n    userId,\n    messageId,\n    originalText: text,\n    shouldCallAi: false,\n    skipTelegram: false,\n    telegramMethod: '',\n    telegramPayload: {},\n    aiPrompt: '',\n  };\n\n  if (!chatId) {\n    output.skipTelegram = true;\n    return { json: output };\n  }\n\n  if (callbackQuery) {\n    if (text === 'press_me') {\n      output.telegramMethod = 'editMessageText';\n      output.telegramPayload = {\n        chat_id: chatId,\n        message_id: messageId,\n        text: 'Ты нажал inline-кнопку.',\n      };\n      return { json: output };\n    }\n\n    if (text === 'back_to_menu') {\n      output.telegramMethod = 'sendMessage';\n      output.telegramPayload = {\n        chat_id: chatId,\n        text: 'Возврат в меню. Жми кнопки снизу.',\n        reply_markup: replyKeyboard,\n      };\n      return { json: output };\n    }\n\n    output.telegramMethod = 'answerCallbackQuery';\n    output.telegramPayload = {\n      callback_query_id: callbackQuery.id,\n      text: 'Неизвестная кнопка',\n      show_alert: false,\n    };\n    return { json: output };\n  }\n\n  if (lowerText === '/start') {\n    output.telegramMethod = 'sendMessage';\n    output.telegramPayload = {\n      chat_id: chatId,\n      text: 'Привет! Я бот с кнопками.\\nВыбери действие:',\n      reply_markup: replyKeyboard,\n    };\n    return { json: output };\n  }\n\n  if (lowerText === '/help' || lowerText === 'помощь') {\n    output.telegramMethod = 'sendMessage';\n    output.telegramPayload = {\n      chat_id: chatId,\n      text: 'Команды:\\n/start - запуск\\n/menu - меню\\n/buttons - inline-кнопки\\n\\nИли нажимай кнопки внизу.',\n    };\n    return { json: output };\n  }\n\n  if (lowerText === '/menu' || lowerText === 'меню') {\n    output.telegramMethod = 'sendMessage';\n    output.telegramPayload = {\n      chat_id: chatId,\n      text: 'Вот меню:',\n      reply_markup: replyKeyboard,\n    };\n    return { json: output };\n  }\n\n  if (lowerText === '/buttons' || lowerText === 'ссылки') {\n    output.telegramMethod = 'sendMessage';\n    output.telegramPayload = {\n      chat_id: chatId,\n      text: 'Ссылки с inline-кнопками:',\n      reply_markup: inlineKeyboard,\n    };\n    return { json: output };\n  }\n\n  if (lowerText === 'калькулятор') {\n    output.telegramMethod = 'sendMessage';\n    output.telegramPayload = {\n      chat_id: chatId,\n      text: 'Напиши пример в формате 2+2, 10/5 или 12*3.',\n    };\n    return { json: output };\n  }\n\n  try {\n    const result = evaluateExpression(text);\n    output.telegramMethod = 'sendMessage';\n    output.telegramPayload = {\n      chat_id: chatId,\n      text: `Ответ: ${result}`,\n    };\n    return { json: output };\n  } catch (error) {\n    output.shouldCallAi = true;\n    output.aiPrompt = `Ты Telegram-бот. Ответь коротко, понятно и по-русски. Пользователь написал: ${text}`;\n    return { json: output };\n  }\n});"
      },
      "id": "2",
      "name": "Code Router",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        520,
        300
      ]
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            {
              "value1": "={{ $json.shouldCallAi }}",
              "value2": true
            }
          ]
        }
      },
      "id": "3",
      "name": "IF AI",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [
        760,
        300
      ]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$env.TELEGRAM_BOT_TOKEN}}/{{$json.telegramMethod}}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ $json.telegramPayload }}",
        "options": {}
      },
      "id": "4",
      "name": "Telegram API",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1010,
        420
      ]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://api.openai.com/v1/responses",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Authorization",
              "value": "={{ 'Bearer ' + $env.OPENAI_API_KEY }}"
            },
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ { model: 'gpt-4.1-mini', input: $json.aiPrompt } }}",
        "options": {}
      },
      "id": "5",
      "name": "OpenAI Response",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1010,
        180
      ]
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const answer = $json.output?.[0]?.content?.[0]?.text ?? 'Не удалось подготовить ответ.';\nconst source = $item(0).$node['Code Router'].json;\n\nreturn {\n  chatId: source.chatId,\n  telegramMethod: 'sendMessage',\n  telegramPayload: {\n    chat_id: source.chatId,\n    text: answer,\n  },\n};"
      },
      "id": "6",
      "name": "Build AI Reply",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1260,
        180
      ]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "=https://api.telegram.org/bot{{$env.TELEGRAM_BOT_TOKEN}}/{{$json.telegramMethod}}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ $json.telegramPayload }}",
        "options": {}
      },
      "id": "7",
      "name": "Telegram AI Reply",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [
        1510,
        180
      ]
    }
  ],
  "connections": {
    "Telegram Trigger": {
      "main": [
        [
          {
            "node": "Code Router",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Code Router": {
      "main": [
        [
          {
            "node": "IF AI",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "IF AI": {
      "main": [
        [
          {
            "node": "OpenAI Response",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Telegram API",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "OpenAI Response": {
      "main": [
        [
          {
            "node": "Build AI Reply",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build AI Reply": {
      "main": [
        [
          {
            "node": "Telegram AI Reply",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "settings": {
    "executionOrder": "v1"
  },
  "active": false,
  "versionId": "1"
}
# bothelp
