# TipTap-Paste-Limit-FIX


I was facing an issue in TipTap where:

1). Pasting stopped working when character limit was near full
2). CharacterCount blocked paste completely
3). Only plain text paste worked, but formatted content lost styling
4). When pressing Enter & pasting, formatting broke
5). Remaining characters were not pasted properly

There was no complete solution in the TipTap community, so I created a fully working extension that:

1). Preserves formatting (bold, italic, lists, paragraphs)
2). Allows paste even when limit is not fully available
3). Inserts only the remaining allowed characters
4). Supports nested nodes
5). Works with CharacterCount & StarterKit
6). Works in Next.js without SSR issues
7). Fixes “formatting removed” & “paste blocked” problems


this is our extension code :

import { Extension } from "@tiptap/core";
import { Plugin, PluginKey } from "prosemirror-state";
import { Slice, Fragment } from "prosemirror-model";

export const LimitPasteHTML = Extension.create({
    name: "limitPasteHTML",

    addProseMirrorPlugins() {
        const limit = 1500;

        return [
            new Plugin({
                key: new PluginKey('limitPasteHTML'),
                props: {
                    // Handle plain text paste
                    clipboardTextParser: (text, context, plain, view) => {
                        const currentLength = context.doc.textContent.length;
                        const remaining = limit - currentLength;

                        // If no space left, return empty slice
                        if (remaining <= 0) {
                            return Slice.empty;
                        }

                        // Truncate text to fit remaining space
                        const truncatedText = text.length > remaining ? text.substring(0, remaining) : text;
                        
                        // Create a text node from the truncated text (preserves all characters including spaces)
                        const textNode = context.doc.type.schema.text(truncatedText);
                        const fragment = Fragment.from(textNode);
                        
                        // Return a Slice with the truncated content
                        return new Slice(fragment, 0, 0);
                    },
                    
                    // Handle formatted content (HTML) paste
                    transformPasted: (slice, view) => {
                        const currentLength = view.state.doc.textContent.length;
                        const pastedText = slice.content.textBetween(0, slice.content.size, '\n', '\n');
                        const remaining = limit - currentLength;

                        // If paste fits completely, return as is
                        if (pastedText.length <= remaining) {
                            return slice;
                        }

                        // If no space left, return empty
                        if (remaining <= 0) {
                            return Slice.empty;
                        }

                        // Truncate the pasted content recursively
                        let charCount = 0;
                        
                        const truncateNode = (node: any, isFirstNode: boolean = false): any => {
                            if (charCount >= remaining) {
                                return null;
                            }

                            if (node.isText) {
                                const nodeText = node.text || '';
                                const availableSpace = remaining - charCount;
                                
                                if (nodeText.length <= availableSpace) {
                                    charCount += nodeText.length;
                                    return node;
                                } else {
                                    // Truncate text to fit
                                    const truncatedText = nodeText.substring(0, availableSpace);
                                    charCount += truncatedText.length;
                                    return node.type.schema.text(truncatedText, node.marks);
                                }
                            }

                            // Handle nodes with content (like paragraphs, lists, etc.)
                            if (node.content && node.content.size > 0) {
                                const newContent: any[] = [];
                                
                                node.content.forEach((childNode: any) => {
                                    if (charCount >= remaining) {
                                        return false;
                                    }
                                    const truncated = truncateNode(childNode, false);
                                    if (truncated) {
                                        newContent.push(truncated);
                                    }
                                });

                                if (newContent.length > 0) {
                                    return node.copy(Fragment.from(newContent));
                                }
                            } else if (!node.isText && node.type.name !== 'hardBreak') {
                                // Empty block node (like empty paragraph) - count as newline
                                if (!isFirstNode && charCount < remaining) {
                                    charCount += 1; // Count the paragraph break as 1 character
                                    return node;
                                }
                            } else if (node.type.name === 'hardBreak') {
                                // Hard break - count as newline
                                if (charCount < remaining) {
                                    charCount += 1;
                                    return node;
                                }
                            }

                            return null;
                        };

                        const truncatedNodes: any[] = [];
                        let isFirst = true;
                        slice.content.forEach((node: any) => {
                            if (charCount >= remaining) {
                                return false;
                            }
                            const truncated = truncateNode(node, isFirst);
                            if (truncated) {
                                truncatedNodes.push(truncated);
                                isFirst = false;
                            }
                        });

                        const newFragment = Fragment.from(truncatedNodes);
                        return new Slice(newFragment, slice.openStart, slice.openEnd);
                    },
                },
            }),
        ];
    },
});


this is tip tap editor :

  const editor = useEditor({
        immediatelyRender: false,   // 🔥 REQUIRED FIX FOR SSR ERROR
      
        extensions: [
            LimitPasteHTML.configure(),
            StarterKit.configure({
                bold: {},  // ✅ Enable bold
                italic: {},  // ✅ Enable italic
                strike: false,
                heading: false,
                orderedList: {},  // ✅ Enable lists
                bulletList: {},  // ✅ Enable bullet lists
            }),
            CharacterCount.configure({
                limit: MAX_CHARACTERS,
                mode: "textSize",
            }),
            Placeholder.configure({
                placeholder: "Describe the role, responsibilities, and requirements...",
            }),
        ],

        content: "",
        editorProps: {
            attributes: {
                class:
                    "prose prose-sm max-w-none min-h-[200px] w-full outline-none text-base focus:outline-none",
                'data-placeholder': "Describe the role, responsibilities, and requirements...",
            },
        },

        onUpdate({ editor }) {
            // Store current valid HTML content
            const html = editor.getHTML();
            lastValidContentRef.current = html;

            // Process markdown
            const markdown = turndownService.turndown(html);
            setMarkdownText(markdown);

            const textContent = editor.getText();
            if (textContent.length === 0) {
                setError("");
            }
        },

        onCreate({ editor }) {
            // Store initial valid content
            lastValidContentRef.current = editor.getHTML();
        },

    });


   

    const { charactersCount, wordsCount } = useEditorState({
        editor,
        selector: context => ({
          charactersCount: context?.editor?.storage?.characterCount?.characters() ?? 0,
          wordsCount: context?.editor?.storage?.characterCount?.words() ?? 0,
        }),
      }) ?? { charactersCount: 0, wordsCount: 0 }

       {editor && <EditorContent editor={editor} />}

          {/* Character Count */}
                           <div className={cn(
                               "flex justify-end text-xs sm:text-sm font-medium",
                               charactersCount >= MAX_CHARACTERS ? "text-red-600" : "text-gray-500"
                           )}>
                            {charactersCount}/{MAX_CHARACTERS} characters
                           </div>

                           
