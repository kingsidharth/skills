# Image Input (VLMs)

Requires a Vision-Language Model (e.g. `qwen2-vl-2b-instruct`).

## From file path

```ts
const image = await client.files.prepareImage("/path/to/image.jpg");
```

## From base64

```ts
const image = await client.files.prepareImageBase64(base64String);
```

## Pass to `.respond()`

```ts
const prediction = model.respond([
  { role: "user", content: "Describe this image", images: [image] },
]);
```

Supported formats: JPEG, PNG, WebP.
