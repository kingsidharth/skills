# Toolbar Reference

## Setup

```tsx
import { toolbarPlugin } from '@mdxeditor/editor'

plugins={[
  toolbarPlugin({
    toolbarClassName: 'my-toolbar-class',  // optional custom class
    toolbarContents: () => (
      <>
        <UndoRedo />
        <BoldItalicUnderlineToggles />
      </>
    )
  })
]}
```

## Built-in Toolbar Components

All imports from `@mdxeditor/editor`.

| Component | Required Plugin | Description |
|---|---|---|
| `UndoRedo` | — | Undo/redo buttons |
| `BoldItalicUnderlineToggles` | — | Toggle bold, italic, underline |
| `CodeToggle` | — | Toggle inline `code` formatting |
| `BlockTypeSelect` | `headingsPlugin`, `quotePlugin` | Dropdown: paragraph, H1–H6, blockquote |
| `CreateLink` | `linkDialogPlugin` | Opens link edit dialog |
| `InsertImage` | `imagePlugin` | Insert image from URL or upload |
| `InsertTable` | `tablePlugin` | Insert a new table |
| `InsertThematicBreak` | `thematicBreakPlugin` | Insert horizontal rule |
| `ListsToggle` | `listsPlugin` | Toggle bullet/numbered lists |
| `InsertCodeBlock` | `codeBlockPlugin` | Insert fenced code block |
| `InsertSandpack` | `sandpackPlugin` | Insert live code block (dropdown of presets) |
| `InsertFrontmatter` | `frontmatterPlugin` | Insert front-matter block |
| `InsertAdmonition` | `directivesPlugin` + `AdmonitionDirectiveDescriptor` | Insert admonition |
| `ChangeAdmonitionType` | `directivesPlugin` + `AdmonitionDirectiveDescriptor` | Change admonition type |
| `ChangeCodeMirrorLanguage` | `codeMirrorPlugin` | Change code block language |
| `ShowSandpackInfo` | `sandpackPlugin` | Show focused Sandpack block name |
| `DiffSourceToggleWrapper` | `diffSourcePlugin` | Wraps toolbar children; adds rich-text/source/diff toggle |
| `Separator` | — | Vertical line between toolbar groups |

## Toolbar Composition Patterns

### Basic Toolbar

```tsx
toolbarPlugin({
  toolbarContents: () => (
    <>
      <UndoRedo />
      <Separator />
      <BoldItalicUnderlineToggles />
      <CodeToggle />
      <Separator />
      <BlockTypeSelect />
    </>
  )
})
```

### With Diff/Source Toggle

Wrap rich-text controls inside `DiffSourceToggleWrapper`. When in source/diff mode, the wrapper hides the rich-text controls and shows mode toggle buttons:

```tsx
toolbarPlugin({
  toolbarContents: () => (
    <DiffSourceToggleWrapper>
      <UndoRedo />
      <Separator />
      <BoldItalicUnderlineToggles />
      <BlockTypeSelect />
      <CreateLink />
      <InsertImage />
      <InsertTable />
      <ListsToggle />
      <InsertCodeBlock />
    </DiffSourceToggleWrapper>
  )
})
```

### Context-Aware Toolbar (Code Blocks)

Use `ConditionalContents` to show different toolbar items based on what's focused:

```tsx
import { ConditionalContents } from '@mdxeditor/editor'

toolbarPlugin({
  toolbarContents: () => (
    <ConditionalContents
      options={[
        {
          when: (editor) => editor?.editorType === 'codeblock',
          contents: () => <ChangeCodeMirrorLanguage />
        },
        {
          when: (editor) => editor?.editorType === 'sandpack',
          contents: () => <ShowSandpackInfo />
        },
        {
          fallback: () => (
            <>
              <UndoRedo />
              <BoldItalicUnderlineToggles />
              <InsertCodeBlock />
              <InsertSandpack />
            </>
          )
        }
      ]}
    />
  )
})
```

## Toolbar Primitives (Custom Components)

Build custom toolbar buttons using these primitives from `@mdxeditor/editor`:

| Primitive | Purpose |
|---|---|
| `Button` | Basic toolbar button |
| `ButtonWithTooltip` | Button with hover tooltip |
| `ButtonOrDropdownButton` | Button that becomes dropdown with multiple items |
| `DialogButton` | Opens dialog with text input + autocomplete + submit |
| `Select` | Dropdown select |
| `SingleChoiceToggleGroup` | Exclusive toggle group (like radio buttons) |
| `MultipleChoiceToggleGroup` | Non-exclusive toggles (like checkboxes) |
| `Separator` | Vertical divider |

### Custom Button Example

```tsx
import { Button, usePublisher, insertJsx$ } from '@mdxeditor/editor'

const InsertCallout = () => {
  const insertJsx = usePublisher(insertJsx$)
  return (
    <Button
      onClick={() => insertJsx({
        name: 'Callout',
        kind: 'flow',
        props: { type: 'info' }
      })}
    >
      Insert Callout
    </Button>
  )
}
```

### Custom View Mode Toggle

Build a custom rich-text/source/diff toggle without `DiffSourceToggleWrapper`:

```tsx
import { useCellValue, usePublisher, viewMode$ } from '@mdxeditor/editor'

const CustomViewToggle = () => {
  const viewMode = useCellValue(viewMode$)
  const changeViewMode = usePublisher(viewMode$)

  return (
    <SingleChoiceToggleGroup
      value={viewMode}
      items={[
        { title: 'Rich text', contents: 'Rich', value: 'rich-text' },
        { title: 'Source', contents: 'Source', value: 'source' },
        { title: 'Diff', contents: 'Diff', value: 'diff' }
      ]}
      onChange={(value) => changeViewMode(value || 'rich-text')}
    />
  )
}
```
