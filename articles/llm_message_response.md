# openai
## messages: 
```
[ { role: 'user', content: 'go to white house' } ]
```

## functions

```
[
  {
    type: "function",
    function: {
      name: "setCenter",
      description: "set center location",
      parameters: {
        type: "object",
        properties: {
          lat: {
            type: "number",
            description: "latitude of center",
          },
          lng: {
            type: "number",
            description: "longtitude of center",
          },
        },
        required: ["lat", "lng"],
      },
    },
  },
]


```
  [
    {
      "role": "system",
      "content": "You are an operator for Google Maps. Follow the user's instructions and call the necessary functions accordingly."
    },
    {
      "role": "user",
      "content": "go to white house"
    },
    {
      "role": "assistant",
      "tool_calls": [
        {
          "id": "call_sY4xCdScn5R62FMSFtzitC2k",
          "type": "function",
          "function": {
            "name": "googleMapAgent--movePlaces",
            "arguments": "{\"textQuery\":\"White House\",\"includedType\":\"government_office\"}"
          }
        }
      ]
    }
  ]
```

```
    {
      "role": "tool",
      "tool_call_id": "call_sY4xCdScn5R62FMSFtzitC2k",
      "name": "googleMapAgent--movePlaces",
      "content": "success"
    }
```

```
 [
    {
      "role": "user",
      "content": "find cafe"
    },
    {
      "role": "assistant",
      "tool_calls": [
        {
          "id": "call_oYbSpKGx9tiD8ECh8ueE8wTm",
          "type": "function",
          "function": {
            "name": "googleMapAgent--findPlaces",
            "arguments": "{\"textQuery\":\"cafe\",\"includedType\":\"cafe\"}"
          }
        }
      ]
    },
    {
      "role": "tool",
      "tool_call_id": "call_oYbSpKGx9tiD8ECh8ueE8wTm",
      "name": "googleMapAgent--findPlaces",
      "content": "[{\"name\":\"Uptown Cafe\",\"address\":\"One NoMA Station, 131 M St NE, Washington, DC 20002 アメリカ合衆国\"},{\"name\":\"パネラ・ブレッド\",\"address\":\"601 13th St NW, Washington, DC 20005 アメリカ合衆国\"},{\"name\":\"Corner Bakery\",\"address\":\"500 North Capitol St NW Ste 110, Washington, DC 20001 アメリカ合衆国\"},{\"name\":\"Uptowner Cafe\",\"address\":\"1609 King St, Alexandria, VA 22314 アメリカ合衆国\"},{\"name\":\"Corner Bakery\",\"address\":\"529 14th St NW, Washington, DC 20045 アメリカ合衆国\"},{\"name\":\"Panera Bread\",\"address\":\"1750 H St NW Suite G1, Washington, DC 20006 アメリカ合衆国\"}]"
    },
    {
      "content": "Here are some cafes you might be interested in:\n\n1. **Uptown Cafe**\n   - Address: One NoMA Station, 131 M St NE, Washington, DC 20002, USA\n\n2. **パネラ・ブレッド (Panera Bread)**\n   - Address: 601 13th St NW, Washington, DC 20005, USA\n\n3. **Corner Bakery**\n   - Address: 500 North Capitol St NW Ste 110, Washington, DC 20001, USA\n\n4. **Uptowner Cafe**\n   - Address: 1609 King St, Alexandria, VA 22314, USA\n\n5. **Corner Bakery**\n   - Address: 529 14th St NW, Washington, DC 20045, USA\n\n6. **Panera Bread**\n   - Address: 1750 H St NW Suite G1, Washington, DC 20006, USA\n\nLet me know if you need more information about any of these places!",
      "role": "assistant"
    }
  ]
```  

# anthropic

## messages: 
```
[ { role: 'user', content: 'go to white house' } ]
```

## functions

```
[
  {
    name: 'setCenter',
    description: 'set center location',
    input_schema: { type: 'object', properties: [Object], required: [Array] }
  }
]
```

## response

```
{
  id: 'msg_01VMupbikDVeNZ4vxfkYjtRq',
  type: 'message',
  role: 'assistant',
  model: 'claude-3-5-sonnet-20241022',
  content: [
    {
      type: 'text',
      text: "I'll help you set the center of the map to the White House location in Washington, DC. The White House is located at approximately 38.8977° N, 77.0365° W."
    },
    {
      type: 'tool_use',
      id: 'toolu_01XnRmo6115jYBWpWuhXBxnz',
      name: 'setCenter',
      input: { lat: 38.8977, lng: -77.0365 }
    }
  ],
  stop_reason: null,
  stop_sequence: null,
  usage: {
    input_tokens: 412,
    cache_creation_input_tokens: 0,
    cache_read_input_tokens: 0,
    output_tokens: 2
  }
}
```

https://docs.anthropic.com/en/docs/build-with-claude/tool-use#example-of-successful-tool-result
```
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01XnRmo6115jYBWpWuhXBxnz",
      "content": "success"
    }
  ]
}
```