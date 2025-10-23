# Code Explanation: coding.js

This file demonstrates **streaming responses** with token limits and real-time output, showing how to get immediate feedback from the LLM as it generates text.

## Step-by-Step Code Breakdown

### 1. Import and Setup (Lines 1-8)
```javascript
import {
    getLlama,
    LlamaChatSession,
} from "node-llama-cpp";
import {fileURLToPath} from "url";
import path from "path";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```
Standard setup for LLM interaction.

### 2. Load Model (Lines 10-18)
```javascript
const llama = await getLlama();
const model = await llama.loadModel({
    modelPath: path.join(
        __dirname,
        "../",
        "models",
        "hf_giladgd_gpt-oss-20b.MXFP4.gguf"
    )
});
```
- Uses **gpt-oss-20b**: A 20 billion parameter model
- **MXFP4**: Mixed precision 4-bit quantization for smaller size
- Larger model = better code explanations

### 3. Create Context and Session (Lines 19-22)
```javascript
const context = await model.createContext();
const session = new LlamaChatSession({
    contextSequence: context.getSequence(),
});
```
Basic session setup with no system prompt.

### 4. Define the Question (Line 24)
```javascript
const q1 = `What is hoisting in JavaScript? Explain with examples.`;
```
A technical programming question that requires detailed explanation.

### 5. Display Context Size (Line 26)
```javascript
console.log('context.contextSize', context.contextSize)
```
- Shows the maximum context window size
- Helps understand memory limitations
- Useful for debugging

### 6. Streaming Prompt Execution (Lines 28-36)
```javascript
const a1 = await session.prompt(q1, {
    // Tip: let the lib choose or cap reasonably; using the whole context size can be wasteful
    maxTokens: 2000,

    // Fires as soon as the first characters arrive
    onTextChunk: (text) => {
        process.stdout.write(text); // optional: live print
    },
});
```

**Key parameters:**

**maxTokens: 2000**
- Limits response length to 2000 tokens (~1500 words)
- Prevents runaway generation
- Saves time and compute
- Without limit: model uses entire context

**onTextChunk callback**
- Fires **as each token is generated**
- Receives text as it's produced
- `process.stdout.write()`: Prints without newlines
- Creates real-time "typing" effect

### How Streaming Works

```
Without streaming:
User → [Wait 10 seconds...] → Complete response appears

With streaming:
User → [Token 1] → [Token 2] → [Token 3] → ... → Complete
       "What"      "is"        "hoisting"
       (Immediate feedback!)
```

### 7. Display Final Answer (Line 38)
```javascript
console.log("\n\nFinal answer:\n", a1);
```
- Prints the complete response again
- Useful for logging or verification
- Shows full text after streaming

### 8. Cleanup (Lines 41-44)
```javascript
llama.dispose()
model.dispose()
context.dispose()
session.dispose()
```
Standard resource cleanup.

## Key Concepts Demonstrated

### 1. Streaming Responses

**Why streaming matters:**
- **Better UX**: Users see progress immediately
- **Early termination**: Can stop if response is off-track
- **Perceived speed**: Feels faster than waiting
- **Debugging**: See generation in real-time

**Comparison:**
```
Non-streaming:           Streaming:
═══════════════         ═══════════════
Request sent            Request sent
[10s wait...]           "What" (0.1s)
Complete response       "is" (0.2s)
                        "hoisting" (0.3s)
                        ... continues
                        (Same total time, better experience!)
```

### 2. Token Limits

**maxTokens controls generation length:**

```
No limit:               With limit (2000):
─────────             ─────────────────
May generate forever   Stops at 2000 tokens
Uses entire context    Saves computation
Unpredictable cost     Predictable cost
```

**Token approximation:**
- 1 token ≈ 0.75 words (English)
- 2000 tokens ≈ 1500 words
- 4-5 paragraphs of detailed explanation

### 3. Real-Time Feedback Pattern

The `onTextChunk` callback enables:
```javascript
onTextChunk: (text) => {
    // Do anything with each chunk:
    process.stdout.write(text);      // Console output
    // socket.emit('chunk', text);   // WebSocket to client
    // buffer += text;               // Accumulate for processing
    // analyzePartial(text);         // Real-time analysis
}
```

### 4. Context Size Awareness

```javascript
console.log('context.contextSize', context.contextSize)
```

Shows model's memory capacity:
- Small models: 2048-4096 tokens
- Medium models: 8192-16384 tokens  
- Large models: 32768+ tokens

**Why it matters:**
```
Context Size: 4096 tokens
Prompt: 100 tokens
Max response: 2000 tokens
History: Up to 1996 tokens
```

## Use Cases

### 1. Code Explanations (This Example)
```javascript
prompt: "Explain hoisting in JavaScript"
→ Streams detailed explanation with examples
```

### 2. Long-Form Content Generation
```javascript
prompt: "Write a blog post about AI agents"
maxTokens: 3000
→ Streams article as it's written
```

### 3. Interactive Tutoring
```javascript
// User sees explanation being built
prompt: "Teach me about closures"
onTextChunk: (text) => displayToUser(text)
```

### 4. Web Applications
```javascript
// Server-Sent Events or WebSocket
onTextChunk: (text) => {
    websocket.send(text);  // Send to browser
}
```

## Performance Considerations

### Token Generation Speed

Depends on:
- **Model size**: Larger = slower per token
- **Hardware**: GPU > CPU
- **Quantization**: Lower bits = faster
- **Context length**: Longer context = slower

**Typical speeds:**
```
Model Size    GPU (RTX 4090)    CPU (M2 Max)
──────────    ──────────────    ────────────
1.7B          50-80 tok/s       15-25 tok/s
8B            20-35 tok/s       5-10 tok/s
20B           10-15 tok/s       2-4 tok/s
```

### When to Use maxTokens

```
✓ Use maxTokens when:
  • Response length is predictable
  • You want to save computation
  • Testing/debugging
  • API rate limiting

✗ Don't limit when:
  • Need complete answer
  • Length varies greatly
  • Using stop sequences instead
```

## Advanced Streaming Patterns

### Pattern 1: Progressive Enhancement
```javascript
let buffer = '';
onTextChunk: (text) => {
    buffer += text;
    if (buffer.includes('\n\n')) {
        // Complete paragraph ready
        processParagraph(buffer);
        buffer = '';
    }
}
```

### Pattern 2: Early Stopping
```javascript
let isRelevant = true;
onTextChunk: (text) => {
    if (text.includes('irrelevant_keyword')) {
        isRelevant = false;
        // Stop generation (would need additional API)
    }
}
```

### Pattern 3: Multi-Consumer
```javascript
onTextChunk: (text) => {
    console.log(text);           // Console
    logFile.write(text);         // File
    websocket.send(text);        // Client
    analyzer.process(text);      // Analysis
}
```

## Expected Output

When run, you'll see:
1. Context size logged (e.g., "context.contextSize 32768")
2. Streaming response appearing token-by-token
3. Complete final answer printed again

Example output flow:
```
context.contextSize 32768
Hoisting is a JavaScript mechanism where variables and function 
declarations are moved to the top of their scope before code 
execution. For example:

console.log(x); // undefined (not an error!)
var x = 5;

This works because...
[continues streaming...]

Final answer:
[Complete response printed again]
```

## Why This Matters for AI Agents

### User Experience
- Real-time agents feel more responsive
- Users can interrupt if going wrong direction
- Better for conversational interfaces

### Resource Management
- Token limits prevent runaway generation
- Predictable costs and timing
- Can cancel expensive operations early

### Integration Patterns
- Web UIs show "typing" effect
- CLIs display progressive output
- APIs stream to clients efficiently

This pattern is essential for production agent systems where user experience and resource control matter.
