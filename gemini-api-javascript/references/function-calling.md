# Function Calling

Connect Gemini to external tools and APIs. The model decides when to call functions and provides parameters — your code executes the actual function.

## Define and Call

```ts
import { GoogleGenAI, Type } from "@google/genai";

const ai = new GoogleGenAI({});

// 1. Define function declaration
const scheduleMeeting = {
  name: "schedule_meeting",
  description: "Schedules a meeting with specified attendees at a given time.",
  parameters: {
    type: Type.OBJECT,
    properties: {
      attendees: {
        type: Type.ARRAY,
        items: { type: Type.STRING },
        description: "List of people attending the meeting.",
      },
      date: { type: Type.STRING, description: "Date (e.g., '2024-07-29')" },
      time: { type: Type.STRING, description: "Time (e.g., '15:00')" },
      topic: { type: Type.STRING, description: "Subject of the meeting." },
    },
    required: ["attendees", "date", "time", "topic"],
  },
};

// 2. Send request with tool declarations
const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "Schedule a meeting with Bob and Alice for 03/14 at 10AM about Q3 planning.",
  config: {
    tools: [{ functionDeclarations: [scheduleMeeting] }],
  },
});

// 3. Check for function call
if (response.functionCalls?.length) {
  const fc = response.functionCalls[0];
  console.log(`Call: ${fc.name}`, fc.args);
  // Execute your function, then send result back...
}
```

## Complete Round-Trip (Auto via Chat)

The chat API can handle the full loop automatically with `automaticFunctionCalling`:

```ts
// Define actual function implementation
function setLightValues({ brightness, color_temp }) {
  return { brightness, colorTemperature: color_temp };
}

const chat = ai.chats.create({
  model: "gemini-3-flash-preview",
  config: {
    tools: [{
      functionDeclarations: [{
        name: "set_light_values",
        description: "Sets the brightness and color temperature of a light.",
        parameters: {
          type: Type.OBJECT,
          properties: {
            brightness: { type: Type.NUMBER, description: "Light level 0-100" },
            color_temp: { type: Type.STRING, enum: ["daylight", "cool", "warm"] },
          },
          required: ["brightness", "color_temp"],
        },
      }],
    }],
    automaticFunctionCalling: {
      // Map function names to implementations
      set_light_values: setLightValues,
    },
  },
});

const response = await chat.sendMessage({
  message: "Turn the lights down to a romantic level",
});
console.log(response.text); // Final response after function execution
```

## Manual Round-Trip

When you need control over function execution:

```ts
// Step 1: Get function call from model
const response1 = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: [{ role: "user", parts: [{ text: "What's the weather in NYC?" }] }],
  config: { tools: [{ functionDeclarations: [getWeatherDecl] }] },
});

const fc = response1.candidates[0].content.parts[0].functionCall;

// Step 2: Execute function yourself
const weatherResult = await getWeather(fc.args);

// Step 3: Send result back
const response2 = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: [
    { role: "user", parts: [{ text: "What's the weather in NYC?" }] },
    response1.candidates[0].content,  // Model's function call (preserves thoughtSignature)
    { role: "user", parts: [{ functionResponse: { name: fc.name, response: weatherResult } }] },
  ],
  config: { tools: [{ functionDeclarations: [getWeatherDecl] }] },
});

console.log(response2.text); // "The weather in NYC is..."
```

## Parallel Function Calling

Model may return multiple function calls in one response:

```ts
// Response might contain:
// parts[0] = { functionCall: { name: "check_weather", args: { city: "Paris" } }, thoughtSignature: "<Sig>" }
// parts[1] = { functionCall: { name: "check_weather", args: { city: "London" } } }

// Send both results back:
{ role: "user", parts: [
  { functionResponse: { name: "check_weather", response: { temp: "15C" } } },
  { functionResponse: { name: "check_weather", response: { temp: "12C" } } },
]}
```

## Multimodal Function Responses (Gemini 3)

Function responses can include images and other media:

```ts
{
  role: "tool",
  parts: [{
    functionResponse: {
      name: "get_image",
      response: { image_ref: { "$ref": "photo.jpg" } },
      parts: [{
        inlineData: {
          mimeType: "image/jpeg",
          displayName: "photo.jpg",
          data: base64ImageData,
        },
      }],
    },
  }],
}
```

## Function Calling Modes

Control when the model calls functions via `toolConfig`:

```ts
config: {
  tools: [{ functionDeclarations: [...] }],
  toolConfig: {
    functionCallingConfig: {
      mode: "AUTO",  // AUTO (default) | ANY | NONE
      // allowedFunctionNames: ["specific_function"],  // with ANY mode
    },
  },
}
```

- `AUTO`: Model decides whether to call functions or respond with text
- `ANY`: Model must call a function (optionally restricted to `allowedFunctionNames`)
- `NONE`: Model cannot call functions (use for testing prompts)

## Thought Signatures (Critical for Gemini 3)

Function calling with Gemini 3 strictly requires thought signatures. Always preserve `thoughtSignature` on model parts when building history manually. The SDK chat API handles this automatically.
