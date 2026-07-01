# MongoDB Plugin

The MongoDB plugin provides indexer and retriever implementations that use MongoDB's vector database capabilities, including Atlas Search for text search, vector search, and hybrid search.

## Installation

```bash
npm install genkitx-mongodb
```

## Configuration

To use this plugin, specify it when you initialize Genkit:

```typescript
import { genkit } from "genkit";
import { mongodb } from "genkitx-mongodb";

const ai = genkit({
  plugins: [
    mongodb([
      {
        url: "mongodb://localhost:27017",
        indexer: {
          id: "my-indexer",
        },
        retriever: {
          id: "my-retriever",
        },
        crudTools: {
          id: "my-crud",
        },
        searchIndexTools: {
          id: "my-search-index",
        },
      },
    ]),
  ],
});
```

You must specify a MongoDB connection URL and configure the components you want to use (indexer, retriever, CRUD tools, search index tools).

## Usage

Import retriever and indexer references like so:

```typescript
import { mongoRetrieverRef, mongoIndexerRef } from "genkitx-mongodb";
```

Then, use these references with `ai.retrieve()` and `ai.index()`:

**Note:** The examples below use mock names for collections, databases, and indexes. Replace these with your actual collection names, database names, and index names as needed.

**Embedder Flexibility:** You can use any embedder that's compatible with Genkit. The examples show Google AI embedders, but you can use OpenAI, Vertex AI, or any other supported embedder.

### Vector Search

```typescript
// To use the retriever you configured when you loaded the plugin:
let docs = await ai.retrieve({
  retriever: mongoRetrieverRef("my-retriever"),
  query: "search query",
  options: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    embedder: googleAI.embedder("text-embedding-004"), // Required for vector search
    vectorSearch: {
      index: "vector_index", // Required: Your vector search index name
      path: "embedding", // Required: Field path containing embeddings
      exact: false, // Optional: Use exact search instead of approximate
      numCandidates: 100, // Optional: Number of candidates to consider
      limit: 10, // Optional: Maximum number of results to return
      filter: { category: "sample" }, // Optional: Filter results
    },
  },
});
```

### Text Search

```typescript
const results = await ai.retrieve({
  retriever: mongoRetrieverRef("my-retriever"),
  query: "search query",
  options: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    search: {
      index: "text_index", // Required: Your text search index name
      text: {
        path: "content", // Required: Field path to search
        matchCriteria: "any", // Optional: "any", "all", or "phrase"
        fuzzy: {
          // Optional: Fuzzy search configuration
          maxEdits: 2,
          prefixLength: 0,
          maxExpansions: 50,
        },
      },
    },
    pipelines: [{ $limit: 10 }, { $sort: { score: -1 } }], // Optional: Additional aggregation stages
  },
});
```

### Hybrid Search

```typescript
const results = await ai.retrieve({
  retriever: mongoRetrieverRef("my-retriever"),
  query: "search query",
  options: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    embedder: googleAI.embedder("text-embedding-004"), // Required for hybrid search
    hybridSearch: {
      search: {
        index: "text_index", // Required: Your text search index name
        text: {
          path: "content", // Required: Field path to search
          fuzzy: {
            // Optional: Fuzzy search configuration
            maxEdits: 2,
            prefixLength: 0,
            maxExpansions: 50,
          },
        },
      },
      vectorSearch: {
        index: "embedding_index", // Required: Your vector search index name
        path: "embedding", // Required: Field path containing embeddings
        exact: false, // Optional: Use exact search instead of approximate
        numCandidates: 100, // Optional: Number of candidates to consider
        limit: 10, // Optional: Maximum number of results to return
      },
      combination: {
        weights: {
          // Optional: Weight configuration for combining results
          vectorPipeline: 0.7,
          fullTextPipeline: 0.3,
        },
      },
      scoreDetails: true, // Optional: Include detailed scoring information
    },
  },
});
```

### Indexing Documents

```typescript
// To use the indexer you configured when you loaded the plugin:
await ai.index({
  indexer: mongoIndexerRef("my-indexer"),
  documents: [
    Document.fromText("Sample document content", {
      id: "1",
      category: "sample",
    }),
    Document.fromText("Another document", { id: "2", category: "example" }),
  ],
  options: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    embedder: googleAI.embedder("text-embedding-004"), // Required: Your chosen embedder
    embeddingField: "embedding", // Optional: Field name for embeddings (default: "embedding")
    batchSize: 100, // Optional: Batch size for processing (default: 100)
    skipData: false, // Optional: Skip storing document data (default: false)
    dataField: "data", // Optional: Field name for document data (default: "data")
    metadataField: "metadata", // Optional: Field name for metadata (default: "metadata")
    dataTypeField: "dataType", // Optional: Field name for data type (default: "dataType")
  },
});
```

### CRUD Operations

The plugin provides tools for basic CRUD operations by document ID. You can get all available tools using the helper functions:

```typescript
import {
  mongoCrudToolsRefArray,
  mongoSearchIndexToolsRefArray,
} from "genkitx-mongodb";

// Get all CRUD tool references for a connection
const crudTools = mongoCrudToolsRefArray("my-crud");
// Returns: ['mongodb/my-crud/create', 'mongodb/my-crud/read', 'mongodb/my-crud/update', 'mongodb/my-crud/delete']

// Get all search index tool references for a connection
const searchIndexTools = mongoSearchIndexToolsRefArray("my-search-index");
// Returns: ['mongodb/my-search-index/create', 'mongodb/my-search-index/list', 'mongodb/my-search-index/drop']
```

### CRUD Operations

```typescript
// Create a document
await ai.runTool({
  name: "mongodb/my-crud/create",
  input: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    document: { name: "John", age: 30 }, // Required: Document to insert
  },
});

// Read a document by ID
const result = await ai.runTool({
  name: "mongodb/my-crud/read",
  input: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    id: "507f1f77bcf86cd799439011", // Required: Document ID to retrieve
  },
});

// Update a document by ID
await ai.runTool({
  name: "mongodb/my-crud/update",
  input: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    id: "507f1f77bcf86cd799439011", // Required: Document ID to update
    document: { age: 31 }, // Required: Update data
  },
});

// Delete a document by ID
await ai.runTool({
  name: "mongodb/my-crud/delete",
  input: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    id: "507f1f77bcf86cd799439011", // Required: Document ID to delete
  },
});
```

### Search Index Management

```typescript
// Create a search index
await ai.runTool({
  name: "mongodb/my-search-index/create",
  input: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    indexName: "text_index", // Required: Name for the new index
    definition: {
      // Required: Index definition
      mappings: {
        dynamic: true,
        fields: {
          content: {
            type: "string",
            analyzer: "lucene.english",
          },
        },
      },
    },
  },
});

// List search indexes
const indexes = await ai.runTool({
  name: "mongodb/my-search-index/list",
  input: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
  },
});

// Drop a search index
await ai.runTool({
  name: "mongodb/my-search-index/drop",
  input: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "myCollection", // Required: Your collection name
    indexName: "text_index", // Required: Name of index to drop
  },
});
```

### Multiple Connections

You can configure multiple MongoDB connections with different settings:

```typescript
mongodb([
  {
    url: "mongodb://primary:27017",
    indexer: {
      id: "primary-indexer",
      retry: {
        // Optional: Retry configuration
        retryAttempts: 3,
        baseDelay: 1000,
      },
    },
    retriever: {
      id: "primary-retriever",
      retry: {
        // Optional: Retry configuration
        retryAttempts: 2,
        baseDelay: 500,
      },
    },
    crudTools: { id: "primary-crud" },
    searchIndexTools: { id: "primary-search" },
  },
  {
    url: "mongodb://secondary:27017",
    indexer: {
      id: "secondary-indexer",
      retry: {
        // Optional: Retry configuration
        retryAttempts: 5,
        baseDelay: 2000,
        jitterFactor: 0.2,
      },
    },
    retriever: { id: "secondary-retriever" },
    crudTools: { id: "secondary-crud" },
    searchIndexTools: { id: "secondary-search" },
  },
]);
```

### Multimodal Support

The plugin supports multimodal embeddings for processing images and documents:

```typescript
import { multimodalEmbedding001 } from "@genkit-ai/vertexai";

// Index images with multimodal embeddings
await ai.index({
  indexer: mongoIndexerRef("my-indexer"),
  documents: imageDocuments,
  options: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "imageCollection", // Required: Your collection name
    embedder: multimodalEmbedding001, // Required: Multimodal embedder
    embeddingField: "imageEmbedding", // Optional: Field name for embeddings
    dataField: "imageData", // Optional: Field name for image data
    metadataField: "imageMetadata", // Optional: Field name for metadata
    dataTypeField: "imageType", // Optional: Field name for data type
  },
});

// Retrieve similar images
const results = await ai.retrieve({
  retriever: mongoRetrieverRef("my-retriever"),
  query: "find images similar to a cat",
  options: {
    dbName: "myDatabase", // Required: Your database name
    collectionName: "imageCollection", // Required: Your collection name
    embedder: multimodalEmbedding001, // Required: Multimodal embedder
    vectorSearch: {
      index: "image_embedding_index", // Required: Your vector search index name
      path: "imageEmbedding", // Required: Field path containing embeddings
      numCandidates: 50, // Optional: Number of candidates to consider
      limit: 5, // Optional: Maximum number of results to return
    },
  },
});
```

## Additional Resources

- **Repository**: [genkitx-mongodb](https://github.com/orimdominic/genkitx-mongodb) - Full source code and detailed documentation
- **Plugin**: [plugin/](https://github.com/orimdominic/genkitx-mongodb/tree/main/plugin) - Plugin source code and documentation
- **Test Application**: [testapp/](https://github.com/orimdominic/genkitx-mongodb/tree/main/testapp) - Working example of a Genkit application that uses this plugin with various use cases including document processing, image analysis, and menu search
