# Chonkie-TS Copilot Instructions

## Project Overview

Chonkie-TS is a TypeScript port of the Python chonkie library, providing fast, lightweight text chunking for RAG applications. The library features a dual architecture with **local chunkers** for on-device processing and **cloud chunkers** for API-based processing.

## Key Architecture Patterns

### Chunker Hierarchy
All chunkers extend `BaseChunker` with standardized async patterns:
- `create()` static factory method (always async)
- `chunk(text: string): Promise<Chunk[]>` for single text
- `call()` method supports both single text and batch processing with overloads
- `chunkBatch()` with concurrent/sequential processing control

### Selective Imports for Tree-Shaking
The library uses selective imports to avoid loading heavy dependencies:
```typescript
// ❌ Avoid - loads all chunkers including CodeChunker+web-tree-sitter
import { TokenChunker } from 'chonkie';

// ✅ Prefer - selective import for better tree-shaking
import { TokenChunker } from 'chonkie/chunker/token';
import { CodeChunker } from 'chonkie/chunker/code';  // Only when needed
```

### Export Strategy
The main `index.ts` deliberately excludes heavy dependencies:
- `CodeChunker` requires `import { CodeChunker } from "chonkie/chunker/code"`
- `ChromaHandshake` requires `import { ChromaHandshake } from "chonkie/friends"`
- Cloud chunkers accessible via `import { TokenChunker } from "chonkie/cloud"`

## Tokenizer Architecture

The `Tokenizer` class wraps multiple backends:
- **"chonkie"**: Custom `CharacterTokenizer`/`WordTokenizer`
- **"transformers"**: HuggingFace transformers.js (default: `"google-bert/bert-base-uncased"`)
- **"callable"**: Custom tokenizer with `countTokens` method

Always use `await Tokenizer.create()` - never direct instantiation.

## Development Workflows

### Testing
```bash
npx jest tests/                        # All tests
npx jest tests/chunker/                # Chunker tests only
npx jest tests/chunker/tokenChunker.test.ts  # Specific test
```

### Build Process
```bash
npm run build      # Clean + TypeScript compilation to dist/
npm run clean      # Remove dist/ folder
npm test          # Run Jest test suite
```

### Project Structure
```
src/chonkie/
├── chunker/        # Local chunkers (TokenChunker, SentenceChunker, etc.)
├── cloud/          # Cloud API clients (same interface, different backend)
├── types/          # TypeScript type definitions
├── friends/        # External integrations (ChromaDB, etc.)
└── utils/          # Utilities (visualization, hub operations)
```

## Code Patterns

### Chunk Creation
All chunks use the `Chunk` class with validation:
```typescript
const chunk = new Chunk({
  text: "chunk text",
  startIndex: 0,
  endIndex: 10,
  tokenCount: 5,
  embedding?: number[]  // Optional
});
```

### Error Handling
- Validate parameters in `create()` methods (chunk size > 0, overlap < chunk size)
- Use descriptive error messages with context
- Handle tokenizer backend-specific errors gracefully

### Async Patterns
- All chunker operations are async (even local ones for consistency)
- Use `Promise.all()` for concurrent batch processing when `_useConcurrency = true`
- Support progress reporting for batch operations

### Optional Dependencies
The library uses `optionalDependencies` for features like:
- `chromadb` for vector database integration
- `tree-sitter-wasms` for code parsing
- `openai`/`cohere-ai` for cloud chunkers

Check availability before use:
```typescript
try {
  const treesSitter = await import('web-tree-sitter');
  // Use tree-sitter functionality
} catch (error) {
  throw new Error('web-tree-sitter not available');
}
```

## Type Safety

- Use strict TypeScript configuration
- Provide proper type definitions for all chunk types
- Use generics for chunker base classes
- Export types through `chonkie/types` for external use

## Testing Guidelines

- Test with multiple tokenizer backends
- Verify chunk index correctness using `verifyChunkIndices()` helper
- Test edge cases: empty text, single token, special characters, emojis
- Use normalized text comparison for chunk validation
- Test both single and batch processing modes

## Performance Considerations

- Chunkers support `_useConcurrency` flag for batch processing strategy
- TokenChunker uses sliding window approach for overlapping chunks
- Cloud chunkers implement request batching and caching
- Prefer streaming for large documents when possible

## Documentation Style

Follow Google-style JSDoc comments:
```typescript
/**
 * Splits text into chunks of specified size.
 *
 * @param text - Input text to chunk
 * @param chunkSize - Maximum size of each chunk
 * @returns Promise resolving to list of chunks
 * @throws Error if chunkSize <= 0
 */
```

## Examples Location

Local examples: `examples/local/` - demonstrate local chunker usage
Cloud examples: `examples/cloud/` - demonstrate cloud API usage
All examples follow the pattern: import → create → call → verify reconstruction
