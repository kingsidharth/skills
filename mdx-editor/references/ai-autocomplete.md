# AI Autocomplete Reference

MDXEditor does not ship with a built-in AI autocomplete feature. Implementing AI-powered ghost text (Copilot-style inline suggestions) requires building a custom plugin that interacts with the underlying Lexical editor. This reference covers the architecture and implementation patterns.

## Approach Overview

There are three main approaches to AI features in MDXEditor:

| Approach | Complexity | UX |
|---|---|---|
| **Ghost text (inline suggestion)** | High — requires custom Lexical DecoratorNode | Copilot-style dimmed text at cursor, Tab to accept |
| **Toolbar button → AI insert** | Medium — uses `insertMarkdown` ref method | User clicks button, AI generates, inserts at cursor |
| **Streaming AI replace** | Medium — uses `setMarkdown` ref method | Replace entire editor content with AI-generated stream |

## Approach 1: Ghost Text (Inline AI Autocomplete)

This is the most sophisticated approach — showing dimmed suggestion text at the cursor that the user can accept with Tab. Requires building a custom MDXEditor plugin with a custom Lexical node.

### Architecture

1. **AutocompleteNode** — a Lexical `DecoratorNode` that renders ghost text as a React component
2. **AutocompletePlugin** — a custom MDXEditor plugin that listens for typing pauses, calls the AI API, and inserts/removes the AutocompleteNode
3. **Debounced trigger** — waits for user to pause typing before fetching suggestions
4. **Tab handler** — accepts the suggestion by replacing the node with actual text

### Implementation Skeleton

```tsx
// AutocompleteNode.ts
import { DecoratorNode, type LexicalNode, type NodeKey } from 'lexical'

export class AutocompleteNode extends DecoratorNode<JSX.Element> {
  __suggestion: string

  static getType(): string { return 'autocomplete' }

  static clone(node: AutocompleteNode): AutocompleteNode {
    return new AutocompleteNode(node.__suggestion, node.__key)
  }

  constructor(suggestion: string, key?: NodeKey) {
    super(key)
    this.__suggestion = suggestion
  }

  createDOM(): HTMLElement {
    return document.createElement('span')
  }

  updateDOM(): boolean { return false }

  // Prevent the node from being included in markdown export
  exportJSON() {
    return { type: 'autocomplete', version: 1, suggestion: this.__suggestion }
  }

  static importJSON(): AutocompleteNode {
    return new AutocompleteNode('')
  }

  decorate(): JSX.Element {
    return (
      <span
        style={{ color: '#999', fontStyle: 'italic', userSelect: 'none' }}
        data-lexical-autocomplete="true"
      >
        {this.__suggestion}
      </span>
    )
  }
}
```

```tsx
// useAIAutocomplete.ts — hook for AI suggestion logic
import { useLexicalComposerContext } from '@lexical/react/LexicalComposerContext'
// Note: MDXEditor exposes Lexical context via its plugin system

import { useEffect, useRef, useCallback } from 'react'
import {
  $getSelection, $isRangeSelection, $createTextNode,
  COMMAND_PRIORITY_LOW, KEY_TAB_COMMAND, KEY_ESCAPE_COMMAND,
  $getRoot
} from 'lexical'
import { AutocompleteNode } from './AutocompleteNode'

type FetchSuggestion = (context: string) => Promise<string | null>

export function useAIAutocomplete(
  editor: LexicalEditor,
  fetchSuggestion: FetchSuggestion,
  debounceMs = 500
) {
  const timeoutRef = useRef<ReturnType<typeof setTimeout>>()
  const currentNodeRef = useRef<AutocompleteNode | null>(null)

  // Remove any existing autocomplete node
  const clearSuggestion = useCallback(() => {
    editor.update(() => {
      const existing = currentNodeRef.current
      if (existing && existing.isAttached()) {
        existing.remove()
      }
      currentNodeRef.current = null
    })
  }, [editor])

  // Insert ghost text at cursor
  const showSuggestion = useCallback((text: string) => {
    editor.update(() => {
      const selection = $getSelection()
      if (!$isRangeSelection(selection) || !selection.isCollapsed()) return

      // Remove previous suggestion
      if (currentNodeRef.current?.isAttached()) {
        currentNodeRef.current.remove()
      }

      const node = new AutocompleteNode(text)
      selection.insertNodes([node])
      currentNodeRef.current = node
    })
  }, [editor])

  // Listen for text changes → debounce → fetch suggestion
  useEffect(() => {
    const removeListener = editor.registerUpdateListener(({ editorState, tags }) => {
      // Skip updates caused by our own autocomplete insertions
      if (tags.has('autocomplete')) return

      clearSuggestion()
      clearTimeout(timeoutRef.current)

      timeoutRef.current = setTimeout(() => {
        editorState.read(() => {
          const selection = $getSelection()
          if (!$isRangeSelection(selection) || !selection.isCollapsed()) return

          const textContent = $getRoot().getTextContent()
          // Get text up to cursor for context
          fetchSuggestion(textContent).then((suggestion) => {
            if (suggestion) showSuggestion(suggestion)
          })
        })
      }, debounceMs)
    })

    return () => {
      removeListener()
      clearTimeout(timeoutRef.current)
    }
  }, [editor, fetchSuggestion, debounceMs, clearSuggestion, showSuggestion])

  // Tab to accept
  useEffect(() => {
    return editor.registerCommand(
      KEY_TAB_COMMAND,
      (event) => {
        const node = currentNodeRef.current
        if (!node || !node.isAttached()) return false

        event.preventDefault()
        editor.update(() => {
          const textNode = $createTextNode(node.__suggestion)
          node.replace(textNode)
          textNode.selectEnd()
          currentNodeRef.current = null
        }, { tag: 'autocomplete' })
        return true
      },
      COMMAND_PRIORITY_LOW
    )
  }, [editor])

  // Escape to dismiss
  useEffect(() => {
    return editor.registerCommand(
      KEY_ESCAPE_COMMAND,
      () => {
        if (currentNodeRef.current?.isAttached()) {
          clearSuggestion()
          return true
        }
        return false
      },
      COMMAND_PRIORITY_LOW
    )
  }, [editor, clearSuggestion])
}
```

### Integrating with MDXEditor via Custom Plugin

MDXEditor uses Gurx for state management. To access the Lexical editor instance:

```tsx
import { realmPlugin, Cell, useCellValue, rootEditor$ } from '@mdxeditor/editor'

// Define the plugin
export const aiAutocompletePlugin = realmPlugin({
  init(realm) {
    // The rootEditor$ cell holds the Lexical editor instance
    // Register the AutocompleteNode
    realm.pub(addLexicalNode$, AutocompleteNode)
  }
})

// React component that uses the plugin
function AIAutocompleteToolbarButton() {
  const rootEditor = useCellValue(rootEditor$)
  // Use rootEditor with useAIAutocomplete hook
  // ...
}
```

### Alternative: Simpler Plugin via Editor Ref

If you don't want to build a full Gurx plugin, access Lexical through MDXEditor's ref:

```tsx
function EditorWithAI() {
  const editorRef = useRef<MDXEditorMethods>(null)

  // Access the underlying Lexical editor via internal methods
  // Note: This is less clean but simpler
  return (
    <MDXEditor
      ref={editorRef}
      markdown=""
      plugins={[
        // Register AutocompleteNode via codeBlockPlugin pattern
        // or use a realmPlugin
      ]}
    />
  )
}
```

## Approach 2: Toolbar AI Button (Simpler)

A much simpler pattern — user clicks a toolbar button to trigger AI generation:

```tsx
import { Button, usePublisher, useCellValue, markdown$ } from '@mdxeditor/editor'

function AIAssistButton() {
  const currentMarkdown = useCellValue(markdown$)

  const handleClick = async () => {
    const editorRef = /* get ref */
    const response = await fetch('/api/ai/complete', {
      method: 'POST',
      body: JSON.stringify({ context: currentMarkdown })
    })
    const { text } = await response.json()
    editorRef.current?.insertMarkdown(text)
  }

  return <Button onClick={handleClick}>AI Assist</Button>
}
```

## Approach 3: Streaming AI Content

For replacing or appending content from an AI stream (e.g., Vercel AI SDK `useCompletion`):

```tsx
import { useCompletion } from 'ai/react'

function AIEditor() {
  const editorRef = useRef<MDXEditorMethods>(null)

  const { completion, isLoading, complete } = useCompletion({
    api: '/api/ai/generate',
  })

  // Update editor as stream arrives
  useEffect(() => {
    if (completion) {
      editorRef.current?.setMarkdown(completion)
    }
  }, [completion])

  return (
    <>
      <button onClick={() => complete('Write a blog post about...')}>
        Generate
      </button>
      <MDXEditor ref={editorRef} markdown="" plugins={[...]} />
    </>
  )
}
```

**Caveat**: Calling `setMarkdown` on every stream chunk clears and re-renders the entire editor. For better UX with streaming, consider accumulating the stream in state and only calling `setMarkdown` periodically (throttled), or use the Lexical `$generateNodesFromDOM` approach with an HTML intermediary.

## AI API Integration Patterns

### With Anthropic Claude

```tsx
const fetchSuggestion = async (context: string): Promise<string | null> => {
  try {
    const response = await fetch('/api/ai/autocomplete', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        context,
        maxTokens: 100,
      })
    })
    const data = await response.json()
    return data.suggestion || null
  } catch {
    return null
  }
}

// Server-side route handler (e.g., Express, Next.js API route)
// POST /api/ai/autocomplete
app.post('/api/ai/autocomplete', async (req, res) => {
  const { context, maxTokens } = req.body

  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': process.env.ANTHROPIC_API_KEY,
      'anthropic-version': '2023-06-01',
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-20250514',
      max_tokens: maxTokens || 100,
      messages: [{
        role: 'user',
        content: `Continue this markdown document naturally. Return ONLY the continuation text, no explanation:\n\n${context}`
      }]
    })
  })

  const data = await response.json()
  const suggestion = data.content?.[0]?.text || null
  res.json({ suggestion })
})
```

### Performance Tips

- **Debounce**: 300–500ms after last keystroke before triggering AI call
- **Cancel stale requests**: Use AbortController to cancel in-flight requests when user keeps typing
- **Token limit**: Keep AI responses short (50–100 tokens) for inline suggestions
- **Context window**: Send only the last ~2000 characters as context, not the full document
- **Caching**: Cache recent suggestions to avoid redundant API calls
- **Rate limiting**: Implement client-side rate limiting (max 1 request per second)
