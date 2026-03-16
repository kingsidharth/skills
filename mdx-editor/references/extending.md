# Extending the Editor Reference

MDXEditor is built for extensibility — even core features are implemented as plugins. This reference covers the internal architecture for building custom plugins.

## Gurx State Management

MDXEditor uses **Gurx**, a graph-based reactive state system (not React context). State primitives:

- **Cell** — stateful observable with an initial value (like a signal/atom)
- **Signal** — stateless observable (event stream, no stored value)
- **Action** — named operation that can be published to

### Accessing State from React

```tsx
import { useCellValue, useCellValues, usePublisher } from '@mdxeditor/editor'

// Read a single cell value (re-renders on change)
const markdown = useCellValue(markdown$)

// Read multiple cell values
const [markdown, rootEditor] = useCellValues([markdown$, rootEditor$])

// Get a publisher function for a cell/signal
const applyBlockType = usePublisher(applyBlockType$)
```

### Key Exported Cells & Signals

| Node | Type | Purpose |
|---|---|---|
| `markdown$` | Cell | Current markdown content |
| `rootEditor$` | Cell | The Lexical editor instance |
| `viewMode$` | Cell | Current view mode: `'rich-text'` \| `'source'` \| `'diff'` |
| `applyBlockType$` | Signal | Apply a block type to current selection |
| `insertJsx$` | Signal | Insert a JSX component |
| `addImportVisitor$` | Signal | Register a markdown→Lexical import visitor |
| `addExportVisitor$` | Signal | Register a Lexical→markdown export visitor |
| `addLexicalNode$` | Signal | Register a custom Lexical node class |

## Creating a Custom Plugin

```tsx
import { realmPlugin, Cell, Signal } from '@mdxeditor/editor'

// Declare state nodes at module level
const myValue$ = Cell<string>('initial', (r) => {
  // Optional: initialization logic using the realm instance `r`
  // r.sub(otherCell$, (value) => { ... })
})

const myAction$ = Signal<string>((r) => {
  r.sub(myAction$, (value) => {
    // React to the signal being published
    r.pub(myValue$, value)
  })
})

// Create the plugin
export const myPlugin = realmPlugin<{ initialValue?: string }>({
  init(realm, params) {
    // Register visitors, nodes, or initialize state
    if (params?.initialValue) {
      realm.pub(myValue$, params.initialValue)
    }
  },
  update(realm, params) {
    // Called when plugin params change (optional)
  }
})

// Use in editor
<MDXEditor plugins={[myPlugin({ initialValue: 'hello' })]} />
```

### Plugin with Custom React Components

For simpler cases, you may not need a full plugin. Just declare cells/signals and use them in React components — they'll be automatically included in the realm:

```tsx
import { Cell, useCellValue, usePublisher } from '@mdxeditor/editor'

const counter$ = Cell(0)

function CounterWidget() {
  const count = useCellValue(counter$)
  const setCount = usePublisher(counter$)
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>
}
```

## Markdown ↔ Lexical Pipeline

The editor converts between markdown and Lexical nodes in two directions:

### Import (Markdown → Lexical)

1. Markdown string is parsed to MDAST (Markdown Abstract Syntax Tree) via remark
2. **Import visitors** walk the MDAST tree and create Lexical nodes

Register a custom import visitor:

```tsx
// In your plugin's init function:
realm.pub(addImportVisitor$, {
  // Which MDAST node type to handle
  testNode: (mdastNode) => mdastNode.type === 'myCustomNode',
  // How to convert it to Lexical nodes
  visitNode: ({ mdastNode, actions, lexicalParent }) => {
    const node = $createMyCustomNode(mdastNode.data)
    actions.addAndStepInto(node)
  }
})
```

### Export (Lexical → Markdown)

Export visitors convert Lexical nodes back to MDAST:

```tsx
realm.pub(addExportVisitor$, {
  testLexicalNode: (lexicalNode) => lexicalNode instanceof MyCustomNode,
  visitLexicalNode: ({ lexicalNode, actions }) => {
    actions.addAndStepInto({
      type: 'myCustomNode',
      data: lexicalNode.getData()
    })
  }
})
```

## Working with Lexical

MDXEditor's rich-text surface is Lexical. Key concepts:

### Node Types

| Node Type | Role | Example |
|---|---|---|
| `TextNode` | Leaf node containing text | Bold text, plain text |
| `ElementNode` | Parent node (block or inline) | Paragraphs, headings, lists |
| `DecoratorNode` | Renders arbitrary React component | Images, code blocks, embeds |

### Registering Custom Nodes

```tsx
// In plugin init:
realm.pub(addLexicalNode$, MyCustomNode)
```

### Editor Updates

All Lexical mutations must happen inside `editor.update()`:

```tsx
const rootEditor = useCellValue(rootEditor$)

rootEditor.update(() => {
  const root = $getRoot()
  const paragraph = $createParagraphNode()
  const text = $createTextNode('Hello')
  paragraph.append(text)
  root.append(paragraph)
})
```

### Commands & Listeners

```tsx
import { COMMAND_PRIORITY_LOW, createCommand } from 'lexical'

const MY_COMMAND = createCommand<string>('MY_COMMAND')

// Register listener
editor.registerCommand(
  MY_COMMAND,
  (payload) => {
    console.log('Command received:', payload)
    return true // handled
  },
  COMMAND_PRIORITY_LOW
)

// Dispatch command
editor.dispatchCommand(MY_COMMAND, 'some payload')
```

### Update Listeners

```tsx
editor.registerUpdateListener(({ editorState, dirtyElements, dirtyLeaves, tags }) => {
  editorState.read(() => {
    const selection = $getSelection()
    const text = $getRoot().getTextContent()
    // React to editor state changes
  })
})
```

## Integrating Lexical Plugins

Pure Lexical plugins (React components using `useLexicalComposerContext`) can be integrated into MDXEditor. However, MDXEditor does NOT use Lexical's own plugin system — it uses Gurx instead. To use a Lexical plugin:

1. Access the root editor via `rootEditor$`
2. Use `useEffect` to register commands/listeners directly on the editor
3. Or wrap the Lexical plugin component and render it within the editor's realm

## Useful Patterns

### Accessing Selection State

```tsx
import { useCellValue, currentSelection$ } from '@mdxeditor/editor'

function SelectionInfo() {
  const selection = useCellValue(currentSelection$)
  // selection is the Lexical selection state
}
```

### Programmatic Formatting

```tsx
import { usePublisher, applyBlockType$ } from '@mdxeditor/editor'

function MakeH1Button() {
  const applyBlockType = usePublisher(applyBlockType$)
  return (
    <button onClick={() => applyBlockType('h1')}>Make H1</button>
  )
}
```

### Reading Markdown Programmatically

```tsx
import { useCellValue, markdown$ } from '@mdxeditor/editor'

function WordCount() {
  const markdown = useCellValue(markdown$)
  const words = markdown.split(/\s+/).filter(Boolean).length
  return <span>{words} words</span>
}
```

## Resources

- [MDXEditor API Reference](https://mdxeditor.dev/editor/api) — all exported cells, signals, types
- [Lexical Docs](https://lexical.dev/docs/intro) — node model, commands, listeners
- [Lexical Playground Source](https://github.com/facebook/lexical/tree/main/packages/lexical-playground) — examples of custom nodes and plugins
- [Gurx Docs](https://www.npmjs.com/package/@mdxeditor/gurx) — reactive state system
