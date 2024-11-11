# GGUF API

GGUF is a binary format used by [llama.cpp](https://github.com/ggerganov/llama.cpp) for storing models and their associated metadata and weights, 
designed for efficient memory mapping of model weights, making it particularly suitable for large language models and other AI applications.

**See** [GGUF specification](https://github.com/ggerganov/ggml/blob/master/docs/gguf.md).

The GGUF API provides methods to read and write GGUF files, access and manage metadata, handle tensor information, and support various data types and array structures.

While the API manages the file structure and metadata, it does not directly write tensor data to the file.
Users are responsible for writing tensor data at the correct file offsets specified by the API, ensuring proper data alignment and following the tensor specifications (type, dimensions, and layout) defined in the metadata.

This design choice separates the concerns of file format management from the actual tensor data handling, allowing users to implement custom tensor writing strategies and optimize for their specific use cases.
The API provides the necessary offset information and tensor specifications, but the actual tensor data writing must be handled by the user's implementation.
For example, after creating a GGUF file with metadata and tensor definitions, users must:

 - Get tensor offsets from the API via `GGUF#tensorDataOffset()` and `TensorInfo#offset()`
 - Seek to the correct file position
 - Write tensor data in the correct format
 - Ensure proper memory alignment according to `GGUF#getAlignment()`

## Get Started

To use the GGUF library in your project, add the following dependency using your preferred build tool. The library requires Java 11 or higher and has no external dependencies.

=== "Maven"

    ```xml title="pom.xml"
    <dependency>
        <groupId>com.llama4j</groupId>
        <artifactId>gguf</artifactId>
        <version>0.1.1</version>
    </dependency>
    ```

=== "Gradle"

    ```groovy title="build.gradle"
    implementation 'com.llama4j:gguf:0.1.1'
    ```

=== "SBT"

    ```scala title="build.sbt"
    libraryDependencies += "com.llama4j" % "gguf" % "0.1.1"
    ```

=== "Mill"

    ```scala title="build.sc"
    ivy"com.llama4j::gguf:0.1.1"
    ```


## Reading GGUF files

```java
import com.llama4j.gguf.GGUF;

Path modelPath = Path.of("/path/to/Llama-3.2-1B-Instruct-Q8_0.gguf");
GGUF gguf = GGUF.read(modelPath);

System.out.println(gguf.getMetadataKeys());
```    

!!! note
    When reading GGUF files, the order of metadata keys and tensors within the file is preserved.

#### Basic Information

```java
// Get GGUF format version
int version = gguf.getVersion();

// Get alignment value (default or specified)
int alignment = gguf.getAlignment();

// Get tensor data offset
long tensorDataOffset = gguf.getTensorDataOffset();
```

#### Accessing Metadata

```java
// Get all metadata keys
Set<String> keys = gguf.getMetadataKeys();

// Check if key exists
boolean hasKey = gguf.containsKey("key");

// Get metadata type
MetadataValueType type = gguf.getType("key");

// Get component type for arrays
MetadataValueType arrayType = gguf.getComponentType("key");

// Get value with type casting
int intValue = gguf.getValue(int.class, "numberKey");
String text = gguf.getValue(String.class, "textKey");
float[] floats = gguf.getValue(float[].class, "floatArrayKey");

// Generic access if type is unknown
Object value = gguf.getValue(Object.class, "key");

// Get value with default fallback
int theValue = gguf.getValueOrDefault(int.class, "key", 0);

// String array
String[] strings = gguf.getValue(String[].class, "stringArray");

// Numeric arrays
int[] integers = gguf.getValue(int[].class, "intArray");
float[] floats = gguf.getValue(float[].class, "floatArray");
double[] doubles = gguf.getValue(double[].class, "doubleArray");

// Boolean array
boolean[] bools = gguf.getValue(boolean[].class, "boolArray");
```

#### Accessing Tensors

```java
// Get all tensor information, tensor order is preserved
Collection<TensorInfo> tensors = gguf.getTensors();

// Get specific tensor info
TensorInfo tensor = gguf.getTensor("token_embeddings");

// Check tensor existence
boolean hasTensor = gguf.containsTensor("token_embeddings");
```

### Best Practices

1. Always close channels after reading/writing:
```java
try (var channel = Files.newByteChannel(path, StandardOpenOption.READ)) {
    GGUF gguf = GGUF.read(channel);
    // Process GGUF data
}
```

2. Check for key/tensor existence before accessing values:
```java
if (gguf.containsKey("key")) {
    int value = gguf.getValue(int.class, "key");
}
// Or use default values
float ropeTetha = gguf.getValueOrDefault(float.class, "key", 1e-5f);
```

### Reading from URLs

No need to download large GGUF files or using a browser to peek at the GGUF metadata - GGUF metadata can be easily read from various sources, including URLs.

```java
static GGUF readFromHuggingFace(String user, String repo, String file) throws IOException {
    URL url = new URL("https://hf.co/%s/%s/resolve/main/%s".formatted(user, repo, file));
    try (var ch = Channels.newChannel(
            new BufferedInputStream(url.openConnection().getInputStream()))) {
        return GGUF.read(ch);
    }
}

// Example usage
String user = "mukel";
String repo = "Llama-3.2-1B-Instruct-GGUF";
String file = "Llama-3.2-1B-Instruct-Q8_0.gguf";
GGUF gguf = readFromHuggingFace(user, repo, file);

System.out.println(gguf.getMetadataKeys());
```

## Writing GGUF files

```java
// Write to file
GGUF gguf = ...;
GGUF.write(gguf, Path.of("model.gguf"));
```

The Builder interface provides a fluent API for creating and modifying GGUF metadata.

### Creating a New GGUF File

```java
// Create a new empty builder
Builder builder = Builder.newBuilder()
    .setVersion(3)     // (optionsl) set GGUF version
    .setAlignment(32); // (optional) set alignment (must be power of 2)

// Add metadata
builder
    .putString("name", "my-model")
    .putInteger("num_layers", 32)
    .putFloat("learning_rate", 0.001f)
    .putArrayOfString("vocab", new String[]{"token1", "token2"});

// Add tensor information
builder.putTensor(
    TensorInfo.create(
        "weights",    // tensor name
        new long[]{1024, 1024}, // shape
        GGMLType.F32, // GGML type
        offset        // tensor offset w.r.t. to gguf.tensorDataOffset()
));

// Build the GGUF instance
GGUF gguf = builder.build();

// Write to file
GGUF.write(gguf, Path.of("model.gguf"));
```

### Modifying Existing GGUF Files

```java
// Create builder from existing GGUF
GGUF existing = GGUF.read(Path.of("model.gguf"));
Builder builder = Builder.newBuilder(existing);

// Modify metadata
builder
    .putString("description", "Updated model")
    .removeKey("old_key")
    .putInteger("num_layers", 64);

// Modify tensors
builder
    .removeTensor("old_tensor")
    .putTensor(TensorInfo.create("new_tensor", ...));

// Build and save
GGUF updated = builder.build();
GGUF.write(updated, Path.of("updated-model.gguf"));
```

### Re-compute tensors offsets

By default, the tensor offsets are re-computed on `.build()`.

```java
// Tensors are packed in the same order, respecting the alignment
GGUF gguf1 = builder.build(); // recomputeTensorOffsets = true

// Leave tensors as-is, do not re-compute tensors offsets
GGUF gguf2 = builder.build(false); // recomputeTensorOffsets = false
```

!!! warning
    When `recomputeTensorOffsets` is false, you must ensure tensor offsets are correctly set manually.

## Metadata Types

The API supports various data types for metadata values, unsigned types are always stored using Java signed types.

| GGUF Type | Java Type | Description |
|-----------|-----------|-------------|
| INT8 | byte | Signed 8-bit integer |
| UINT8 | byte | Unsigned 8-bit integer (requires manual conversion) |
| INT16 | short | Signed 16-bit integer |
| UINT16 | short | Unsigned 16-bit integer (requires manual conversion) |
| INT32 | int | Signed 32-bit integer |
| UINT32 | int | Unsigned 32-bit integer (requires manual conversion) |
| INT64 | long | Signed 64-bit integer |
| UINT64 | long | Unsigned 64-bit integer (requires manual conversion) |
| FLOAT32 | float | 32-bit floating point |
| FLOAT64 | double | 64-bit floating point |
| BOOL | boolean | Boolean value |
| STRING | String | Text string |
| ARRAY | String[] or primitive arrays | Array of supported types |

!!! warning
    Reading/writing arrays of arrays is currently **NOT** supported.

```java
Builder.newBuilder()
    .putBoolean("flag", true)
    .putByte("byte_val", (byte) 1)
    .putShort("short_val", (short) 100)
    .putInteger("int_val", 1000)
    .putLong("long_val", 10000L)
    .putFloat("float_val", 0.5f)
    .putDouble("double_val", 0.75)
    .putBoolean("string_val", "hello")
    // Unsigned types
    .putUnsignedByte("ubyte", (byte) 255)
    .putUnsignedShort("ushort", (short) 65535)
    .putUnsignedInteger("uint", 0xFFFFFFFF)
    .putUnsignedLong("ulong", -1L)  // Represents max unsigned long
    // Arrays
    .putArrayOfString("strings", new String[]{"a", "b", "c"})
    .putArrayOfInteger("ints", new int[]{1, 2, 3})
    .putArrayOfFloat("floats", new float[]{0.1f, 0.2f, 0.3f})    
    .putArrayOfBoolean("flags", new boolean[]{true, false, true});
```
