---
title: "GraphAI - シンプルなサンプル"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [agent, AI, LLM, Tech, GraphAI]
published: true
publication_name: "singularity"
---

vanilla agentのunit test用のコードを、sampleのGraphDataにしました。
それぞれのgraphをjsonに保存すれば、cli、もしくはTypeScript/JavaScriptでテストが可能です。

cliの場合は、
https://www.npmjs.com/package/@receptron/graphai_cli
こちらをインストール。

TypeScriptで実行する場合は以下のコードを参考にしてください。
```typescript
import * as vanilla_agent from "@graphai/vanilla";
import { GraphAI } from "graphai";

import { graphData } from "./data";

const main = async () => {
  const graph = new GraphAI(graphData, vanilla_agent);
  const result = await graph.run();
};

main();

```


```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "text": "Here's to the crazy ones, the misfits, the rebels, the troublemakers, the round pegs in the square holes ... the ones who see things differently -- they're not fond of rules, and they have no respect for the status quo. ... You can quote them, disagree with them, glorify or vilify them, but the only thing you can't do is ignore them because they change things. ... They push the human race forward, and while some may see them as the crazy ones, we see genius, because the people who are crazy enough to think that they can change the world, are the ones who do."
      }
    },
    "node": {
      "isResult": true,
      "agent": "stringSplitterAgent",
      "params": {
        "chunkSize": 64
      },
      "inputs": {
        "text": ":sampleInput.text"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "hello"
    },
    "sampleInput1": {
      "value": "test"
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": "${m1}: ${m2}"
      },
      "inputs": {
        "m1": ":sampleInput0",
        "m2": ":sampleInput1"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "hello"
    },
    "sampleInput1": {
      "value": "test"
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": [
          "${m1}: ${m2}",
          "${m2}: ${m1}"
        ]
      },
      "inputs": {
        "m1": ":sampleInput0",
        "m2": ":sampleInput1"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "hello"
    },
    "sampleInput1": {
      "value": "test"
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": {
          "apple": "${m1}",
          "lemon": "${m2}"
        }
      },
      "inputs": {
        "m1": ":sampleInput0",
        "m2": ":sampleInput1"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "hello"
    },
    "sampleInput1": {
      "value": "test"
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": [
          {
            "apple": "${m1}",
            "lemon": "${m2}"
          }
        ]
      },
      "inputs": {
        "m1": ":sampleInput0",
        "m2": ":sampleInput1"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "hello"
    },
    "sampleInput1": {
      "value": "test"
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": {
          "apple": "${m1}",
          "lemon": [
            "${m2}"
          ]
        }
      },
      "inputs": {
        "m1": ":sampleInput0",
        "m2": ":sampleInput1"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "message1": "hello",
        "message2": "test"
      }
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": "${message1}: ${message2}"
      },
      "inputs": {
        "message1": ":sampleInput.message1",
        "message2": ":sampleInput.message2"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "message1": "hello",
        "message2": "test"
      }
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": [
          "${message1}: ${message2}",
          "${message2}: ${message1}"
        ]
      },
      "inputs": {
        "message1": ":sampleInput.message1",
        "message2": ":sampleInput.message2"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "message1": "hello",
        "message2": "test"
      }
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": {
          "apple": "${message1}",
          "lemon": "${message2}"
        }
      },
      "inputs": {
        "message1": ":sampleInput.message1",
        "message2": ":sampleInput.message2"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "message1": "hello",
        "message2": "test"
      }
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": [
          {
            "apple": "${message1}",
            "lemon": "${message2}"
          }
        ]
      },
      "inputs": {
        "message1": ":sampleInput.message1",
        "message2": ":sampleInput.message2"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "message1": "hello",
        "message2": "test"
      }
    },
    "node": {
      "isResult": true,
      "agent": "stringTemplateAgent",
      "params": {
        "template": {
          "apple": "${message1}",
          "lemon": [
            "${message2}"
          ]
        }
      },
      "inputs": {
        "message1": ":sampleInput.message1",
        "message2": ":sampleInput.message2"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "apple": "red",
        "lemon": "yellow"
      }
    },
    "node": {
      "isResult": true,
      "agent": "jsonParserAgent",
      "inputs": {
        "text": ":sampleInput0"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "{\n  \"apple\": \"red\",\n  \"lemon\": \"yellow\"\n}"
    },
    "node": {
      "isResult": true,
      "agent": "jsonParserAgent",
      "params": {},
      "inputs": {
        "text": ":sampleInput0"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "```\n{\"apple\":\"red\",\"lemon\":\"yellow\"}\n```"
    },
    "node": {
      "isResult": true,
      "agent": "jsonParserAgent",
      "params": {},
      "inputs": {
        "text": ":sampleInput0"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "```json\n{\"apple\":\"red\",\"lemon\":\"yellow\"}\n```"
    },
    "node": {
      "isResult": true,
      "agent": "jsonParserAgent",
      "params": {},
      "inputs": [
        ":sampleInput0"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "```JSON\n{\"apple\":\"red\",\"lemon\":\"yellow\"}\n```"
    },
    "node": {
      "isResult": true,
      "agent": "jsonParserAgent",
      "params": {},
      "inputs": [
        ":sampleInput0"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          1,
          2
        ],
        "item": 3
      }
    },
    "node": {
      "isResult": true,
      "agent": "pushAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array",
        "item": ":sampleInput.item"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          {
            "apple": 1
          }
        ],
        "item": {
          "lemon": 2
        }
      }
    },
    "node": {
      "isResult": true,
      "agent": "pushAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array",
        "item": ":sampleInput.item"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          1,
          2,
          3
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "popAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          "a",
          "b",
          "c"
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "popAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          1,
          2,
          3
        ],
        "array2": [
          "a",
          "b",
          "c"
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "popAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array",
        "array2": ":sampleInput.array2"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          1,
          2,
          3
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "shiftAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          "a",
          "b",
          "c"
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "shiftAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          [
            1
          ],
          [
            2
          ],
          [
            3
          ]
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "arrayFlatAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          [
            1
          ],
          [
            2
          ],
          [
            [
              3
            ]
          ]
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "arrayFlatAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          [
            1
          ],
          [
            2
          ],
          [
            [
              3
            ]
          ]
        ],
        "depth": 2
      }
    },
    "node": {
      "isResult": true,
      "agent": "arrayFlatAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array",
        "depth": ":sampleInput.depth"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          [
            "a"
          ],
          [
            "b"
          ],
          [
            "c"
          ]
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "arrayFlatAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "matrix": [
          [
            1,
            2
          ],
          [
            3,
            4
          ],
          [
            5,
            6
          ]
        ],
        "vector": [
          3,
          2
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "dotProductAgent",
      "params": {},
      "inputs": {
        "matrix": ":sampleInput.matrix",
        "vector": ":sampleInput.vector"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "matrix": [
          [
            1,
            2
          ],
          [
            2,
            3
          ]
        ],
        "vector": [
          1,
          2
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "dotProductAgent",
      "params": {},
      "inputs": {
        "matrix": ":sampleInput.matrix",
        "vector": ":sampleInput.vector"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          "banana",
          "orange",
          "lemon",
          "apple"
        ],
        "values": [
          2,
          5,
          6,
          4
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "sortByValuesAgent",
      "params": {},
      "inputs": {
        "array": ":sampleInput.array",
        "values": ":sampleInput.values"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "array": [
          "banana",
          "orange",
          "lemon",
          "apple"
        ],
        "values": [
          2,
          5,
          6,
          4
        ]
      }
    },
    "node": {
      "isResult": true,
      "agent": "sortByValuesAgent",
      "params": {
        "assendant": true
      },
      "inputs": {
        "array": ":sampleInput.array",
        "values": ":sampleInput.values"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "node": {
      "isResult": true,
      "agent": "echoAgent",
      "params": {
        "message": "this is test"
      },
      "inputs": []
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "node": {
      "isResult": true,
      "agent": "echoAgent",
      "params": {
        "message": "If you add filterParams option, it will respond to filterParams",
        "filterParams": true
      },
      "inputs": []
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "a": "123"
      }
    },
    "node": {
      "isResult": true,
      "agent": "bypassAgent",
      "params": {},
      "inputs": [
        ":sampleInput0"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "a": "123"
        },
        {
          "b": "abc"
        }
      ]
    },
    "sampleInput1": {
      "value": [
        {
          "c": "987"
        },
        {
          "d": "xyz"
        }
      ]
    },
    "node": {
      "isResult": true,
      "agent": "bypassAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "a": "123"
        },
        {
          "b": "abc"
        }
      ]
    },
    "sampleInput1": {
      "value": [
        {
          "c": "987"
        },
        {
          "d": "xyz"
        }
      ]
    },
    "node": {
      "isResult": true,
      "agent": "bypassAgent",
      "params": {
        "firstElement": true
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "a": "123"
        },
        {
          "b": "abc"
        }
      ]
    },
    "sampleInput1": {
      "value": [
        {
          "c": "987"
        },
        {
          "d": "xyz"
        }
      ]
    },
    "node": {
      "isResult": true,
      "agent": "bypassAgent",
      "params": {
        "flat": 1
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "node": {
      "isResult": true,
      "agent": "countingAgent",
      "params": {
        "count": 4
      },
      "inputs": []
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "node": {
      "isResult": true,
      "agent": "copyMessageAgent",
      "params": {
        "count": 4,
        "message": "hello"
      },
      "inputs": []
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "message": "hello"
      }
    },
    "node": {
      "isResult": true,
      "agent": "copy2ArrayAgent",
      "params": {
        "count": 10
      },
      "inputs": [
        ":sampleInput0"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "node": {
      "isResult": true,
      "agent": "streamMockAgent",
      "params": {
        "message": "this is params test"
      },
      "inputs": []
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "message": "this is named inputs test"
      }
    },
    "node": {
      "isResult": true,
      "agent": "streamMockAgent",
      "params": {},
      "inputs": {
        "message": ":sampleInput.message"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "a": 1
      }
    },
    "sampleInput1": {
      "value": {
        "a": 2
      }
    },
    "sampleInput2": {
      "value": {
        "a": 3
      }
    },
    "node": {
      "isResult": true,
      "agent": "totalAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1",
        ":sampleInput2"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "a": 1,
          "b": -1
        },
        {
          "c": 10
        }
      ]
    },
    "sampleInput1": {
      "value": [
        {
          "a": 2,
          "b": -1
        }
      ]
    },
    "sampleInput2": {
      "value": [
        {
          "a": 3,
          "b": -2
        },
        {
          "d": -10
        }
      ]
    },
    "node": {
      "isResult": true,
      "agent": "totalAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1",
        ":sampleInput2"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "a": 1
      }
    },
    "node": {
      "isResult": true,
      "agent": "totalAgent",
      "params": {},
      "inputs": [
        ":sampleInput0"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "a": 1
      }
    },
    "sampleInput1": {
      "value": {
        "a": 2
      }
    },
    "node": {
      "isResult": true,
      "agent": "totalAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "a": 1
      }
    },
    "sampleInput1": {
      "value": {
        "a": 2
      }
    },
    "sampleInput2": {
      "value": {
        "a": 3
      }
    },
    "node": {
      "isResult": true,
      "agent": "totalAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1",
        ":sampleInput2"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "a": 1,
        "b": 1
      }
    },
    "sampleInput1": {
      "value": {
        "a": 2,
        "b": 2
      }
    },
    "sampleInput2": {
      "value": {
        "a": 3,
        "b": 0
      }
    },
    "node": {
      "isResult": true,
      "agent": "totalAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1",
        ":sampleInput2"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "a": 1
      }
    },
    "sampleInput1": {
      "value": {
        "a": 2,
        "b": 2
      }
    },
    "sampleInput2": {
      "value": {
        "a": 3,
        "b": 0
      }
    },
    "node": {
      "isResult": true,
      "agent": "totalAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1",
        ":sampleInput2"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": 1
    },
    "node": {
      "isResult": true,
      "agent": "dataSumTemplateAgent",
      "params": {},
      "inputs": [
        ":sampleInput0"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": 1
    },
    "sampleInput1": {
      "value": 2
    },
    "node": {
      "isResult": true,
      "agent": "dataSumTemplateAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": 1
    },
    "sampleInput1": {
      "value": 2
    },
    "sampleInput2": {
      "value": 3
    },
    "node": {
      "isResult": true,
      "agent": "dataSumTemplateAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1",
        ":sampleInput2"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "color": "red",
        "model": "Model 3",
        "type": "EV",
        "maker": "Tesla",
        "range": 300
      }
    },
    "node": {
      "isResult": true,
      "agent": "propertyFilterAgent",
      "params": {
        "include": [
          "color",
          "model"
        ]
      },
      "inputs": [
        ":sampleInput0"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "color": "red",
          "model": "Model 3",
          "type": "EV",
          "maker": "Tesla",
          "range": 300
        },
        {
          "color": "blue",
          "model": "Model Y",
          "type": "EV",
          "maker": "Tesla",
          "range": 400
        }
      ]
    },
    "sampleInput1": {
      "value": "Tesla Motors"
    },
    "node": {
      "isResult": true,
      "agent": "propertyFilterAgent",
      "params": {
        "include": [
          "color",
          "model"
        ]
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "color": "red",
          "model": "Model 3",
          "type": "EV",
          "maker": "Tesla",
          "range": 300
        },
        {
          "color": "blue",
          "model": "Model Y",
          "type": "EV",
          "maker": "Tesla",
          "range": 400
        }
      ]
    },
    "sampleInput1": {
      "value": "Tesla Motors"
    },
    "node": {
      "isResult": true,
      "agent": "propertyFilterAgent",
      "params": {
        "exclude": [
          "color",
          "model"
        ]
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "color": "red",
          "model": "Model 3",
          "type": "EV",
          "maker": "Tesla",
          "range": 300
        },
        {
          "color": "blue",
          "model": "Model Y",
          "type": "EV",
          "maker": "Tesla",
          "range": 400
        }
      ]
    },
    "sampleInput1": {
      "value": "Tesla Motors"
    },
    "node": {
      "isResult": true,
      "agent": "propertyFilterAgent",
      "params": {
        "alter": {
          "color": {
            "red": "blue",
            "blue": "red"
          }
        }
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "color": "red",
          "model": "Model 3",
          "type": "EV",
          "maker": "Tesla",
          "range": 300
        },
        {
          "color": "blue",
          "model": "Model Y",
          "type": "EV",
          "maker": "Tesla",
          "range": 400
        }
      ]
    },
    "sampleInput1": {
      "value": "Tesla Motors"
    },
    "node": {
      "isResult": true,
      "agent": "propertyFilterAgent",
      "params": {
        "inject": [
          {
            "propId": "maker",
            "from": 1
          }
        ]
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "color": "red",
          "model": "Model 3",
          "type": "EV",
          "maker": "Tesla",
          "range": 300
        },
        {
          "color": "blue",
          "model": "Model Y",
          "type": "EV",
          "maker": "Tesla",
          "range": 400
        }
      ]
    },
    "sampleInput1": {
      "value": "Tesla Motors"
    },
    "node": {
      "isResult": true,
      "agent": "propertyFilterAgent",
      "params": {
        "inject": [
          {
            "propId": "maker",
            "from": 1,
            "index": 0
          }
        ]
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "color": "red",
          "model": "Model 3",
          "type": "EV",
          "maker": "Tesla",
          "range": 300
        },
        {
          "color": "blue",
          "model": "Model Y",
          "type": "EV",
          "maker": "Tesla",
          "range": 400
        }
      ]
    },
    "sampleInput1": {
      "value": "Tesla Motors"
    },
    "node": {
      "isResult": true,
      "agent": "propertyFilterAgent",
      "params": {
        "swap": {
          "maker": "model"
        }
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": [
        {
          "color": "red",
          "model": "Model 3",
          "type": "EV",
          "maker": "Tesla",
          "range": 300
        },
        {
          "color": "blue",
          "model": "Model Y",
          "type": "EV",
          "maker": "Tesla",
          "range": 400
        }
      ]
    },
    "sampleInput1": {
      "value": "Tesla Motors"
    },
    "node": {
      "isResult": true,
      "agent": "propertyFilterAgent",
      "params": {
        "inspect": [
          {
            "propId": "isTesla",
            "equal": "Tesla Motors"
          },
          {
            "propId": "isGM",
            "notEqual": "Tesla Motors",
            "from": 1
          }
        ]
      },
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": {
        "color": "red",
        "model": "Model 3"
      }
    },
    "node": {
      "isResult": true,
      "agent": "copyAgent",
      "params": {},
      "inputs": [
        ":sampleInput0"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput0": {
      "value": "Hello World"
    },
    "sampleInput1": {
      "value": "Discarded"
    },
    "node": {
      "isResult": true,
      "agent": "copyAgent",
      "params": {},
      "inputs": [
        ":sampleInput0",
        ":sampleInput1"
      ]
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "url": "https://www.google.com",
        "queryParams": {
          "foo": "bar"
        },
        "headers": {
          "x-myHeader": "secret"
        }
      }
    },
    "node": {
      "isResult": true,
      "agent": "vanillaFetchAgent",
      "params": {
        "debug": true
      },
      "inputs": {
        "url": ":sampleInput.url",
        "queryParams": ":sampleInput.queryParams",
        "headers": ":sampleInput.headers"
      }
    }
  }
}
```

```json
{
  "version": 0.5,
  "nodes": {
    "sampleInput": {
      "value": {
        "url": "https://www.google.com",
        "body": {
          "foo": "bar"
        }
      }
    },
    "node": {
      "isResult": true,
      "agent": "vanillaFetchAgent",
      "params": {
        "debug": true
      },
      "inputs": {
        "url": ":sampleInput.url",
        "body": ":sampleInput.body"
      }
    }
  }
}
```

