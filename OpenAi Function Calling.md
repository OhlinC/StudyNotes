# OpenAi Function Calling

`gpt-4-0613`开发人员现在可以向和描述函数`gpt-3.5-turbo-0613`，并让模型智能地选择输出包含调用这些函数的参数的 JSON 对象。这是一种更可靠地将 GPT 功能与外部工具和 API 连接的新方法。

这些模型已经过微调，可以检测何时需要调用函数（取决于用户的输入）并使用符合函数签名的 JSON 进行响应。函数调用允许开发人员更可靠地从模型中获取结构化数据。例如，开发人员可以：

- 创建通过调用外部工具（例如 ChatGPT 插件）来回答问题的聊天机器人

将诸如“给 Anya 发电子邮件，看看她下周五是否想喝咖啡”之类的查询转换为函数调用`send_email(to: string, body: string)`，例如“波士顿的天气怎么样？” 到`get_current_weather(location: string, unit: 'celsius' | 'fahrenheit')`。

- 将自然语言转换为 API 调用或数据库查询

转换“谁是本月我的前十名客户？” 到内部 API 调用，例如`get_customers_by_revenue(start_date: string, end_date: string, limit: int)`，或“Acme, Inc. 上个月下了多少订单？” 到一个 SQL 查询使用`sql_query(query: string)`。

- 从文本中提取结构化数据

定义一个名为 的函数`extract_people_data(people: [{name: string, birthday: string, location: string}])`，以提取维基百科文章中提到的所有人。

`/v1/chat/completions`这些用例由我们端点中的新 API 参数启用，`functions`并且`function_call`允许开发人员通过 JSON 模式向模型描述函数，并可选择要求它调用特定函数。

示例：

```
'{
  "model": "gpt-3.5-turbo-0613",
//用户输入
  "messages": [
    {"role": "user", "content": "What is the weather like in Boston?"}
  ],
//声明函数
  "functions": [
    {
//函数名
      "name": "get_current_weather",
//触发函数描述
      "description": "Get the current weather in a given location",
//函数参数
      "parameters": {
        "type": "object",
//对函数参数的描述，假设location写成，体验结束那第一次调用回来的functio_call里的arguments第一个值就是体验结束，如果是模糊概念openai会智能识别填参数
        "properties": {
          "location": {
            "type": "string",
            "description": "The city and state, e.g. San Francisco, CA"
          },
//第二个参数 看情况可不要
          "unit": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"]
          }
        },
        "required": ["location"]
      }
    }
  ]
}'
```

第一次访问openai接口后返回：

```
{
  "id": "chatcmpl-123",
  ...
  "choices": [{
    "index": 0,
//这里我们取message = response["choices"][0]["message"]，第二次请求主体就是这个
    "message": {
      "role": "assistant",
      "content": null,
      "function_call": {
        "name": "get_current_weather",
        "arguments": "{ \"location\": \"Boston, MA\"}"
      }
    },
    "finish_reason": "function_call"
  }]
}
```

返回参数"finish_reason": "function_call"中，当finish_reason为stop时代表无再可用function也就是未匹配到函数，那就说明是最后的结果了，去content就行，这里我们拿到了方法名也就是function_call里面的name属性，加上返回回来的参数，拿着参数做自己的处理，可以访问自己的方法也可以用来发外部api请求，处理完数据存在`function_respons`用于第二次发请求做总结。处理完发第二次请求：

```
model="gpt-3.5-turbo-0613",
            messages=[
//用户的原问题
                {"role": "user", "content": "What is the weather like in boston?"},
//上文中拿到的message           
                    message,
//再次匹配function，直到finish_reason属性为：stop
                {
                    "role": "function",
                    "name": function_name,
                    "content": function_response,
                },
            ],
```

最后拿到content而finish_reason也为stop,ok结束



举例几个固定prompt的function

```
{
  "model": "gpt-3.5-turbo-0613",
  "messages": [
    {"role": "user", "content": "體驗已結束，再見Metis"}
  ],
  "functions": [
    {
      "name": "reply_regular_answer_contact",
      "description": "If anyone wants any contact information about you.",
      "parameters": {
        "type": "object",
        "properties": {
          "question": {
            "type": "string",
            "description": "contact information about you"
          }
        },
        "required": ["question"]
      }
    },
    {
      "name": "reply_regular_answer_interview",
      "description": "If the anyone has any intention of ending this interview.",
      "parameters": {
        "type": "object",
        "properties": {
          "question": {
            "type": "string",
            "description": "At the end of the interview"
          }
        },
        "required": ["question"]
        }
    },
    {
      "name": "reply_regular_answer_experience",
      "description": "If the anyone say:'體驗已結束，再見Metis'",
      "parameters": {
        "type": "object",
        "properties": {
          "question": {
            "type": "string",
            "description": "It has' The experience is over, goodbye Metis' in it"
          }
        },
        "required": ["question"]
        }
    }
  ]
}
```

收到的回答：

```
{
    "id": "chatcmpl-7RgFBReoYIjPDR2INB5KlvJD8t91W",
    "object": "chat.completion",
    "created": 1686831597,
    "model": "gpt-3.5-turbo-0613",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": null,
                "function_call": {
                    "name": "reply_regular_answer_experience",
                    "arguments": "{\n\"answer\": \"體驗已結束，再見Metis\"\n}"
                }
            },
            "finish_reason": "function_call"
        }
    ],
    "usage": {
        "prompt_tokens": 162,
        "completion_tokens": 29,
        "total_tokens": 191
    }
}
```

我们拿到参数因为固定回答暂时不加处理直接声明返回就好了

```
{
  "model": "gpt-3.5-turbo-0613",
  "messages": [
    {"role": "user", "content": "體驗已結束，再見Metis"},
    {"role": "assistant", "content": null, "function_call": {"name": "reply_regular_answer_experience", "arguments": "{\"question\": \"It has' The experience is over, goodbye Metis' in it\"}"}},
    {"role": "function", "name": "reply_regular_answer_experience", "content": "{\"answer\": \"You MUST answer：感謝你參加了今天的體驗，希望你度過了愉快的時光，如果您對AI小助理有任何建議或意見，歡迎隨時聯繫我們Have a nice day！💐\"}"}
  ]
}
```

上面的role是function当中我们假设通过函数拿到了content，通过这种json发送给openai就是完整的function calling的流程，最后我们拿到响应：

```
{
    "id": "chatcmpl-7RgEpR1q9tn0DpBk2V7ZLgeSCjOHs",
    "object": "chat.completion",
    "created": 1686831575,
    "model": "gpt-3.5-turbo-0613",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "感謝你參加了今天的體驗，希望你度過了愉快的時光。如果您對AI小助理有任何建議或意見，歡迎隨時聯繫我們。祝你有美好的一天！💐"
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 144,
        "completion_tokens": 83,
        "total_tokens": 227
    }
}
```




