# Chapter 6: Binary Data Management

Welcome back! In [Chapter 5: Node Execution Context](05_node_execution_context_.md), we learned about the "personal assistant" each node gets, providing it with all the tools and information needed to do its job. One common type of information nodes work with isn't just simple text or numbers, but files like images, PDFs, or spreadsheets. How does n8n handle these larger, more complex pieces of data? That's what this chapter is all about!

## The Problem: Handling Digital Files in Workflows

Imagine your workflow needs to:
1.  Download a customer's profile picture from a website.
2.  Add a company logo as a watermark to that picture.
3.  Upload the watermarked picture to a cloud storage service like Google Drive.

Each of these steps involves a "binary file." Binary data is just a way of saying "any kind of file that isn't plain text" – think images (JPEG, PNG), documents (PDF, Word), audio, video, etc.

If n8n just passed the entire image file from one node to the next in memory, it could use up a lot of your computer's resources, especially for large files or many workflow runs. Also, what if you want n8n to store these files on your server's hard drive, or maybe in a dedicated cloud storage service like Amazon S3?

This is where n8n's **Binary Data Management** system comes in. It's like a **universal digital locker service** for your files.

## The Universal Digital Locker Service: Key Concepts

Think of this system as a very organized attendant at a high-tech locker facility.

1.  **The File (Binary Data):** This is the item you want to store – your image, PDF, etc.
2.  **Metadata (The Label):** Information *about* your file, like its name (`profile.jpg`), its type (`image/jpeg`), and its size.
3.  **The Binary Data Service (The Head Attendant):** This is the main n8n service (`BinaryDataService`) that manages all file operations. You talk to this attendant.
4.  **Storage Managers (The Locker Sections):** The attendant can use different sections of lockers:
    *   **Local Filesystem Manager (`FileSystemManager`):** Like a set of physical cabinets in your local office (n8n server's disk).
    *   **Cloud Storage Manager (`ObjectStoreManager` for S3):** Like a secure, remote warehouse (Amazon S3, etc.).
5.  **The Unique ID (Your Locker Key):** When you give a file to the attendant, they store it and give you back a unique key (an ID string). This key usually tells the attendant *which locker section* it's in and *which specific locker*. For example, `filesystem-v2:abcdef-12345`.
6.  **Abstraction (The Magic):** You, as the workflow builder (or even the node developer), don't usually need to worry *where* the attendant stored your file. You just use the locker key (ID) to get it back when you need it.

This system solves the problem by:
*   **Efficiently storing files:** It can save them to disk or cloud storage instead of keeping them all in memory.
*   **Providing flexibility:** You can configure n8n to use local disk or cloud storage based on your needs.
*   **Simplifying file access for nodes:** Nodes get a consistent way to store and retrieve files.

## How Nodes Use the Locker Service

Let's see how a node interacts with this "digital locker service," usually through helper functions provided by its [Node Execution Context](05_node_execution_context_.md).

### Storing a File (Putting an Item in a Locker)

Imagine an "HTTP Request" node just downloaded an image.
1.  **The Node has the File:** The node has the image data (maybe as a chunk of raw data called a `Buffer` or a continuous flow called a `Stream`) and some metadata (filename, content type).
2.  **Handing it to the Attendant:** The node, often through a helper function like `this.helpers.prepareBinaryData()`, effectively tells the `BinaryDataService` to store this file. It passes the file data and metadata.
3.  **Attendant at Work (`BinaryDataService.store()`):**
    *   The `BinaryDataService` looks at n8n's configuration to see which "locker section" (storage mode: local filesystem or S3) is active.
    *   It calls the appropriate `Manager` (e.g., `FileSystemManager.store()`).
    *   The `Manager` saves the file (e.g., to a specific folder on the disk) and reports back the file's actual ID within that storage and its size.
4.  **Getting the Locker Key:** The `BinaryDataService` then gives the node an updated piece of information about the binary data. This information, often an object called `IBinaryData`, now includes:
    *   A unique `id` (your "locker key"), like `filesystem-v2:some-unique-identifier`.
    *   The `fileName`, `mimeType`, and `fileSize`.
    *   Importantly, if the file was stored externally (like on disk), the raw file `data` is often *removed* from this `IBinaryData` object in memory to save space. The `data` property might instead hold the name of the storage mode (e.g., `"filesystem-v2"`).

```typescript
// Simplified view of what an IBinaryData object might look like
// Before being processed by BinaryDataService for external storage:
let binaryInfo = {
  fileName: 'cat.jpg',
  mimeType: 'image/jpeg',
  data: Buffer.from('...very long image data...'), // Actual file content
  // id: undefined
};

// After BinaryDataService.store() (if stored on filesystem):
binaryInfo = {
  fileName: 'cat.jpg',
  mimeType: 'image/jpeg',
  fileSize: '123 KB', // Added by the service
  id: 'filesystem-v2:generated-uuid-123', // The "locker key"
  data: 'filesystem-v2', // Raw data removed, mode indicated
};
```
This `binaryInfo` object (with the ID) is what gets passed to the next node in the workflow.

### Retrieving a File (Getting an Item from a Locker)

Now, a "Watermark Image" node receives the `binaryInfo` from the previous step. It needs the actual image content.
1.  **Node has the Locker Key:** The node has the `IBinaryData` object, which includes the `id` (e.g., `filesystem-v2:generated-uuid-123`).
2.  **Asking the Attendant:** The node uses a helper like `this.helpers.getBinaryDataBuffer()` or directly asks the `BinaryDataService` (e.g., `BinaryDataService.getAsBuffer(binaryInfo.id)`).
3.  **Attendant Finds the File:**
    *   The `BinaryDataService` looks at the `id`. The part before the colon (e.g., `filesystem-v2`) tells it which `Manager` (locker section) to use. The part after is the specific file's key in that section.
    *   It calls the correct `Manager` (e.g., `FileSystemManager.getAsBuffer(specificFileKey)`).
    *   The `Manager` fetches the file content from its storage (e.g., reads it from disk).
4.  **File Returned:** The `Manager` gives the file content (as a `Buffer` or `Stream`) back to the `BinaryDataService`, which then gives it to the node.

The node can now work with the image data!

## Under the Hood: A Look Inside the Locker Room

Let's see the main components:

```mermaid
sequenceDiagram
    participant Node as Workflow Node
    participant ExecCtx as Node Execution Context Helper
    participant BDS as BinaryDataService (Head Attendant)
    participant FSM as FileSystemManager (Local Cabinet)
    participant OSM as ObjectStoreManager (Remote Warehouse - S3)
    participant Disk as Local Disk Storage
    participant S3 as S3 Cloud Storage

    Node->>ExecCtx: Store this file data & metadata
    ExecCtx->>BDS: store(workflowId, execId, fileData, metadata)
    Note over BDS: Checks n8n config for active mode (e.g., 'filesystem-v2')
    BDS->>FSM: store(workflowId, execId, fileData, metadata)
    FSM->>Disk: Save file, generate unique name
    Disk-->>FSM: File saved, path/ID
    FSM-->>BDS: { fileId: "local-uuid", fileSize: 12345 }
    Note over BDS: Creates full ID: "filesystem-v2:local-uuid"
    BDS-->>ExecCtx: Updated IBinaryData (with ID, fileSize; raw data removed)
    ExecCtx-->>Node: Here's the IBinaryData with the "locker key"

    %% Later, for retrieval
    Node->>ExecCtx: Get file content for ID: "filesystem-v2:local-uuid"
    ExecCtx->>BDS: getAsBuffer("filesystem-v2:local-uuid")
    Note over BDS: Parses ID: mode="filesystem-v2", key="local-uuid"
    BDS->>FSM: getAsBuffer("local-uuid")
    FSM->>Disk: Read file "local-uuid"
    Disk-->>FSM: File content (Buffer)
    FSM-->>BDS: File content (Buffer)
    BDS-->>ExecCtx: File content (Buffer)
    ExecCtx-->>Node: Here's the file content
```

### 1. `BinaryDataService` (The Head Attendant)
File: `src/binary-data/binary-data.service.ts`

This service is the central point.
*   **`init(config)`:** When n8n starts, this method is called. It looks at your n8n environment configuration (e.g., `N8N_BINARY_DATA_MODE` which can be `default`, `filesystem`, or `s3`). Based on this, it prepares the necessary "Storage Managers." For instance, if you set `filesystem`, it creates and initializes a `FileSystemManager`.
    ```typescript
    // Simplified from BinaryDataService.init()
    async init(config: BinaryData.Config) {
        // 'config.mode' comes from n8n settings (e.g., "filesystem" or "s3")
        this.mode = config.mode === 'filesystem' ? 'filesystem-v2' : config.mode;

        if (config.availableModes.includes('filesystem')) {
            // If "filesystem" is an option, prepare the FileSystemManager
            const { FileSystemManager } = await import('./file-system.manager');
            this.managers.filesystem = new FileSystemManager(config.localStoragePath);
            // ... and init it ...
        }
        if (config.availableModes.includes('s3')) {
            // If "s3" is an option, prepare the ObjectStoreManager
            // ... similar setup ...
        }
    }
    ```
*   **`store(workflowId, executionId, bufferOrStream, binaryData)`:** This is called to save a new file.
    ```typescript
    // Simplified from BinaryDataService.store()
    async store(
        workflowId: string,
        executionId: string,
        bufferOrStream: Buffer | Readable, // The actual file content
        binaryData: IBinaryData, // Holds metadata like fileName, mimeType
    ) {
        const manager = this.managers[this.mode]; // Get the active manager (e.g., FileSystemManager)

        if (!manager) {
            // If no manager (e.g., mode='default'), handle in memory (simplified)
            // binaryData.data = (await binaryToBuffer(bufferOrStream)).toString('base64');
            // binaryData.fileSize = ...
            return binaryData;
        }

        // Ask the active manager to store the file
        const { fileId, fileSize } = await manager.store(
            workflowId, executionId, bufferOrStream,
            { fileName: binaryData.fileName, mimeType: binaryData.mimeType },
        );

        // Update binaryData with the "locker key" and details
        binaryData.id = this.createBinaryDataId(fileId); // e.g., "filesystem-v2:actual-file-id"
        binaryData.fileSize = prettyBytes(fileSize);
        binaryData.data = this.mode; // IMPORTANT: Raw data cleared from memory!
        return binaryData;
    }
    ```
    The `createBinaryDataId(fileId)` method simply prepends the current mode to the ID received from the manager, like `filesystem-v2:${fileId}`.

*   **`getAsBuffer(binaryData)` or `getAsStream(binaryDataId)`:** These are called to retrieve a file.
    ```typescript
    // Simplified from BinaryDataService.getAsBuffer()
    async getAsBuffer(binaryData: IBinaryData) {
        if (binaryData.id) { // If there's a "locker key"
            const [mode, fileId] = binaryData.id.split(':'); // Split "filesystem-v2:actual-id"
            // Get the correct manager (e.g., FileSystemManager for "filesystem-v2" mode)
            return await this.getManager(mode).getAsBuffer(fileId);
        }
        // If no ID, data might be directly in binaryData.data (e.g., for 'default' mode)
        return Buffer.from(binaryData.data, 'base64'); // 'base64' is BINARY_ENCODING
    }
    ```
    The `getManager(mode)` helper just looks up the manager (e.g., `this.managers.filesystem`) based on the `mode` string.

### 2. `BinaryData.Manager` Interface (The Job Description)
File: `src/binary-data/types.ts`

This is not a class, but a TypeScript `interface`. It's like a contract or a job description that all storage managers must follow. It defines what methods they must have, like `store`, `getAsBuffer`, `getAsStream`, `getMetadata`, etc.

```typescript
// Simplified from BinaryData.Manager interface in types.ts
export interface Manager {
    init(): Promise<void>;
    store(/*...*/): Promise<{ fileId: string; fileSize: number }>;
    getAsBuffer(fileId: string): Promise<Buffer>;
    getAsStream(fileId: string, chunkSize?: number): Promise<Readable>;
    // ... other methods like getMetadata, copyByFileId, deleteMany ...
}
```
This ensures that `BinaryDataService` can talk to any manager (filesystem, S3, or potentially others in the future) in the same way.

### 3. `FileSystemManager` (The Local Cabinet Attendant)
File: `src/binary-data/file-system.manager.ts`

This manager handles storing files on the n8n server's local disk.
*   **`store(...)`:**
    ```typescript
    // Simplified from FileSystemManager.store()
    async store(
        workflowId: string, executionId: string,
        bufferOrStream: Buffer | Readable, // File content
        { mimeType, fileName }: BinaryData.PreWriteMetadata,
    ) {
        const fileId = this.toFileId(workflowId, executionId); // Generates a unique path/ID
                                                              // e.g., workflows/wfId/executions/execId/binary_data/uuid
        const filePath = this.resolvePath(fileId); // Gets full disk path

        await fs.writeFile(filePath, bufferOrStream); // Save the file to disk
        const fileSize = await this.getSize(fileId);  // Get its size

        // Also save metadata (fileName, mimeType) in a separate .metadata file
        await this.storeMetadata(fileId, { mimeType, fileName, fileSize });

        return { fileId, fileSize }; // Return the ID within this manager & size
    }
    ```
    It creates a structured path (e.g., `workflows/<workflow_id>/executions/<execution_id>/binary_data/<uuid>`) and saves the file there. A separate JSON file with `.metadata` extension is stored alongside to keep the original filename and MIME type.

*   **`getAsBuffer(fileId)`:** Reads the file from the disk path corresponding to `fileId` and returns its content as a `Buffer`.

### 4. `ObjectStoreManager` (The Remote Warehouse Attendant)
File: `src/binary-data/object-store.manager.ts`

This manager handles storing files in an S3-compatible object store.
*   **`store(...)`:**
    ```typescript
    // Simplified from ObjectStoreManager.store()
    async store(
        workflowId: string, executionId: string,
        bufferOrStream: Buffer | Readable, // File content
        metadata: BinaryData.PreWriteMetadata,
    ) {
        const fileId = this.toFileId(workflowId, executionId); // Generates path for S3
                                                              // e.g., workflows/wfId/executions/execId/binary_data/uuid
        const buffer = await binaryToBuffer(bufferOrStream); // Ensure we have a Buffer

        // 'this.objectStoreService' is an instance of ObjectStoreService (from .ee.ts file)
        // which handles actual S3 communication.
        await this.objectStoreService.put(fileId, buffer, metadata);

        return { fileId, fileSize: buffer.length };
    }
    ```
    This manager itself doesn't contain complex S3 logic. It mostly prepares the `fileId` (which acts as the S3 object key/path) and then delegates the actual upload to another service, `ObjectStoreService` (found in `object-store/object-store.service.ee.ts`), which is specialized for S3 communication.

*   **`getAsBuffer(fileId)`:** Tells the `objectStoreService` to download the object with key `fileId` from S3.

This layered approach keeps the main `BinaryDataService` clean and allows different storage backends to be plugged in via their respective `Manager` implementations.

## Conclusion

You've now unlocked the secrets of how n8n manages files and other binary data! You've learned:
*   Why a dedicated system for binary data is crucial (efficiency, flexibility).
*   The "universal digital locker" analogy: how files are stored with metadata, and you get a unique ID (locker key) to retrieve them.
*   That this system abstracts the storage, meaning n8n can use local disk or cloud (like S3) without nodes needing to know the difference.
*   The main players:
    *   `BinaryDataService`: The central coordinator.
    *   `BinaryData.Manager` interface: The contract for different storage types.
    *   `FileSystemManager`: For local disk storage.
    *   `ObjectStoreManager`: For S3-compatible storage (which uses `ObjectStoreService.ee` for actual S3 calls).
*   How a file's `id` (e.g., `mode:file_key`) helps the service find it.
*   That raw file data is often cleared from memory once stored externally, saving resources.

This robust Binary Data Management system is essential for building workflows that can handle all sorts of files, making n8n a powerful tool for diverse automation tasks. This concludes our deep dive into some of the core concepts of n8n! With these foundational building blocks, n8n can orchestrate complex automations involving data transformation, service integrations, and file handling.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)