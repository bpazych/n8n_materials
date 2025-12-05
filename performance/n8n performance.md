#n8n #automation #performance 

## Self-hosted vs service n8n vs vanilla nodejs

Both n8n and Node.js are single-threaded, which is not surprising since n8n under the hood uses Node.js as its engine.

Despite n8n being based on Node.js, it doesn't have the same performance as raw Node.js. n8n has overhead from:

- Workflow engine processing
- Additional DB writes (execution history)
- UI/monitoring layers
- Node-to-node data passing

These factors create additional overhead, which can be partially mitigated by disabling execution history saving—this can potentially provide some performance boost in tests.

**Conclusion:** If you prioritize performance, use Node.js or more performance-oriented languages/systems. If you need versatility and want to write as little code as possible, n8n is your choice.

### Simple Benchmarking

To test and get real numbers, I created a simple endpoint that simulates a regular workflow: fetch data from DB, transform it, and send it back.

> This test not about get absolute values,  but rather relative numbers where nodejs implementation serves as baseline.

Results:
 - vanilla nodejs showed next results
		```
		Total time: 20.05s
		Total requests: 5600
		Requests/sec: 279.27
		Avg processing time: 70.24ms
		Success rate: 100.0%
		```

- self-hosted n8n results
		```
		Total time: 20.46s
		Total requests: 1044
		Requests/sec: 51.01
		Avg processing time: 335.20ms
		Success rate: 100.0%
		```
- service n8n results
		```
		Total time: 22.18s
		Total requests: 139
		Requests/sec: 6.27
		Avg processing time: 2910.82ms
		Success rate: 100.0%
		```
    
Analysis: 
The n8n cloud service results don't fit neatly between vanilla Node.js and self-hosted n8n because the last two ran on the same PC (localhost), while the cloud service includes significant network latency.

Comparing self-hosted solutions: there's a ~265ms difference (335ms - 70ms) between self-hosted n8n and Node.js. This difference includes some measurement variance (it's hard to replicate the exact same logic in n8n workflows vs native JS code), but the majority of this overhead comes from n8n's workflow engine processing.

**Key takeaways:**

- **Node.js baseline**: 279 req/s (localhost, minimal overhead)
- **Self-hosted n8n**: 51 req/s (~5.5x slower due to workflow engine overhead)
- **n8n Cloud**: 6.27 req/s (~44x slower due to network latency + engine overhead)

The dramatic drop in cloud performance is primarily due to network round-trip time (~2800ms network latency), not n8n itself.

##### `server.js` code

```js
const express = require('express');
const app = express();
const PORT = 3000;

app.use(express.json());

// Simulated database delay
const simulateDbQuery = (ms = 50) => {
  return new Promise(resolve => setTimeout(resolve, ms));
};

// Main endpoint for performance testing
app.post('/api/process', async (req, res) => {
  const startTime = Date.now();
  
  try {
    const { items, userId } = req.body;
    
    // Validate input
    if (!items || !Array.isArray(items)) {
      return res.status(400).json({ error: 'items array required' });
    }
    
    // Simulate DB query 1: Get user details
    await simulateDbQuery(30);
    const user = { id: userId || 1, name: 'Test User', role: 'admin' };
    
    // Data transformation: Filter and enrich items
    const processedItems = items
      .filter(item => item.active !== false)
      .map(item => ({
        ...item,
        price: item.price * 1.2, // Add 20% markup
        processedBy: user.name,
        timestamp: new Date().toISOString()
      }));
    
    // Simulate DB query 2: Save results
    await simulateDbQuery(40);
    
    // Calculate summary
    const summary = {
      totalItems: processedItems.length,
      totalValue: processedItems.reduce((sum, item) => sum + item.price, 0),
      processedBy: user.name
    };
    
    const processingTime = Date.now() - startTime;
    
    res.json({
      success: true,
      processingTime,
      summary,
      items: processedItems
    });
    
  } catch (error) {
    res.status(500).json({ 
      error: error.message,
      processingTime: Date.now() - startTime 
    });
  }
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

app.listen(PORT, () => {
  console.log(`Performance test server running on http://localhost:${PORT}`);
  console.log(`Test endpoint: POST http://localhost:${PORT}/api/process`);
});

```

##### `benchmark.js` code

```js
const http = require('http');
const https = require('https');

const testPayload = {
    userId: 123,
    items: [
        {id: 1, name: 'Item A', price: 100, active: true},
        {id: 2, name: 'Item B', price: 200, active: true},
        {id: 3, name: 'Item C', price: 150, active: false},
        {id: 4, name: 'Item D', price: 300, active: true}
    ]
};

const makeRequest = (url) => {
    return new Promise((resolve, reject) => {
        const data = JSON.stringify(testPayload);
        const urlObj = new URL(url);
        const client = urlObj.protocol === 'https:' ? https : http;
        const options = {
            hostname: urlObj.hostname,
            port: urlObj.port || (urlObj.protocol === 'https:' ? 443 : 80),
            path: urlObj.pathname,
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Content-Length': data.length
            }
        };

        const req = client.request(options, (res) => {
            let body = '';
            res.on('data', (chunk) => body += chunk);
            res.on('end', () => {
                resolve({
                    status: res.statusCode,
                    time: Date.now(),
                    body: JSON.parse(body)
                });
            });
        });

        req.on('error', (err) => reject(err));
        req.write(data);
        req.end();
    });
};

const runBenchmark = async (url, durationSec = 30, concurrent = 10) => {
    console.log(`\nBenchmarking: ${url}`);
    console.log(`Duration: ${durationSec}s, Concurrency: ${concurrent}\n`);

    const results = [];
    const startTime = Date.now();
    const endTime = startTime + (durationSec * 1000);
    let requestsInFlight = 0;

    const executeRequest = async () => {
        try {
            requestsInFlight++;
            const result = await makeRequest(url);
            results.push(result);
            process.stdout.write(`Progress: ${results.length} requests, ${Math.floor((Date.now() - startTime) / 1000)}s elapsed\r`);
        } catch (error) {
            console.error('\nRequest failed:', error.message);
        } finally {
            requestsInFlight--;
        }
    };

    // Keep spawning requests until time is up
    const promises = [];
    while (Date.now() < endTime || requestsInFlight > 0) {
        // Spawn new requests if we have capacity and time remaining
        while (requestsInFlight < concurrent && Date.now() < endTime) {
            promises.push(executeRequest());
        }

        // Small delay to prevent CPU spinning
        await new Promise(resolve => setTimeout(resolve, 1));
    }

    // Wait for all in-flight requests to complete
    await Promise.all(promises);

    const totalTime = Date.now() - startTime;
    const processingTimes = results.map(r => r.body.processingTime);
    const avgProcessingTime = processingTimes.reduce((a, b) => a + b, 0) / processingTimes.length;
    const minTime = Math.min(...processingTimes);
    const maxTime = Math.max(...processingTimes);

    console.log(`\n\n=== Results ===`);
    console.log(`Total time: ${(totalTime / 1000).toFixed(2)}s`);
    console.log(`Total requests: ${results.length}`);
    console.log(`Requests/sec: ${((results.length / totalTime) * 1000).toFixed(2)}`);
    console.log(`Avg processing time: ${avgProcessingTime.toFixed(2)}ms`);
    console.log(`Min/Max: ${minTime}ms / ${maxTime}ms`);
    console.log(`Success rate: ${(results.length / (results.length + (promises.length - results.length)) * 100).toFixed(1)}%`);
};

// Get URL from command line or use default
// Usage: node benchmark.js <url> <duration_sec> <concurrency>
const url = process.argv[2] || 'http://localhost:3000/api/process';
const duration = parseInt(process.argv[3]) || 30;
const concurrent = parseInt(process.argv[4]) || 10;

runBenchmark(url, duration, concurrent).catch(console.error);

```

##### n8n workflow

```json
{
  "name": "Performance Test Workflow",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "api/process",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "d257f44f-4182-4185-acea-6a064773c913",
      "name": "Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [
        144,
        144
      ],
      "webhookId": "perf-test"
    },
    {
      "parameters": {
        "jsCode": "const startTime = Date.now();\nconst { items, userId } = $input.item.json.body;\n\n// Validate input\nif (!items || !Array.isArray(items)) {\n  return {\n    json: {\n      error: 'items array required',\n      status: 400\n    }\n  };\n}\n\nreturn {\n  json: {\n    items,\n    userId: userId || 1,\n    startTime\n  }\n};"
      },
      "id": "aa654885-0c92-4aae-a435-d883eceb6204",
      "name": "Validate Input",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        352,
        144
      ]
    },
    {
      "parameters": {
        "amount": 0.03,
        "unit": "seconds"
      },
      "id": "2d609b22-803f-4de6-a561-b0a3b3594e23",
      "name": "Wait - DB Query 1",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1,
      "position": [
        544,
        144
      ],
      "webhookId": "ed85272b-7e8b-4504-8ef3-426939fded82"
    },
    {
      "parameters": {
        "jsCode": "const { items, userId, startTime } = $input.item.json;\n\n// Simulate getting user from DB\nconst user = {\n  id: userId,\n  name: 'Test User',\n  role: 'admin'\n};\n\nreturn {\n  json: {\n    items,\n    user,\n    startTime\n  }\n};"
      },
      "id": "2d480fd8-9ac3-4ad9-a95b-351d9ef6217e",
      "name": "Get User Details",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        752,
        144
      ]
    },
    {
      "parameters": {
        "jsCode": "const { items, user, startTime } = $input.item.json;\n\n// Filter and transform items\nconst processedItems = items\n  .filter(item => item.active !== false)\n  .map(item => ({\n    ...item,\n    price: item.price * 1.2,\n    processedBy: user.name,\n    timestamp: new Date().toISOString()\n  }));\n\nreturn {\n  json: {\n    processedItems,\n    user,\n    startTime\n  }\n};"
      },
      "id": "e8f5cd61-a6ad-4d7e-a296-5ef1013ef6ee",
      "name": "Transform Items",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        944,
        144
      ]
    },
    {
      "parameters": {
        "amount": 0.04,
        "unit": "seconds"
      },
      "id": "79ef24c0-b52e-4c70-91ee-61f424cd61fe",
      "name": "Wait - DB Query 2",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1,
      "position": [
        1152,
        144
      ],
      "webhookId": "6c70f18b-a3aa-40f6-bea3-7579a5af29ec"
    },
    {
      "parameters": {
        "jsCode": "const { processedItems, user, startTime } = $input.item.json;\n\n// Calculate summary\nconst summary = {\n  totalItems: processedItems.length,\n  totalValue: processedItems.reduce((sum, item) => sum + item.price, 0),\n  processedBy: user.name\n};\n\nconst processingTime = Date.now() - startTime;\n\nreturn {\n  json: {\n    success: true,\n    processingTime,\n    summary,\n    items: processedItems\n  }\n};"
      },
      "id": "c416176d-94c9-4fe8-9015-51c3eaf106aa",
      "name": "Calculate Summary",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1344,
        144
      ]
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ $json }}",
        "options": {}
      },
      "id": "3ac49dbe-9697-4b19-8a40-207cb2327d38",
      "name": "Respond to Webhook",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1,
      "position": [
        1552,
        144
      ]
    }
  ],
  "pinData": {},
  "connections": {
    "Webhook": {
      "main": [
        [
          {
            "node": "Validate Input",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Validate Input": {
      "main": [
        [
          {
            "node": "Wait - DB Query 1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Wait - DB Query 1": {
      "main": [
        [
          {
            "node": "Get User Details",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get User Details": {
      "main": [
        [
          {
            "node": "Transform Items",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Transform Items": {
      "main": [
        [
          {
            "node": "Wait - DB Query 2",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Wait - DB Query 2": {
      "main": [
        [
          {
            "node": "Calculate Summary",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Calculate Summary": {
      "main": [
        [
          {
            "node": "Respond to Webhook",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": true,
  "settings": {
    "executionOrder": "v1"
  },
  "versionId": "85fedb9b-1024-471e-91c7-8cff4075b125",
  "meta": {
    "instanceId": "8f13e67da1b35671af007bffbe85480fd82a1c413b4779f113c83d773aee58f3"
  },
  "id": "pzT0s7gu0fEInAy3",
  "tags": []
}
```