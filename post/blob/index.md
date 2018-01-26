<!--more-->
*Attention: The code is based on Flink 1.5*

[](img/blob.png)

## 



# 1. BlobServer


| Value | type |
| --- | --- |
| **storageDir** | File |
| **blobStore** | BlobStore |
| **activeConnections**  | Set&lt;BlobServerConnection&gt; |
| **cleanupTimer** | Timer |
| **blobExpiryTimes** | ConcurrentHashMap&lt;Tuple2&lt;JobID, TransientBlobKey&gt;, Long&gt; |
| **shutdownHook** | Thread |
| **cleanTimer** | Timer |

## 1. BlobKey

| Value | type |
| --- | --- |
| **Key** | byte[20] |
| **type** | PERMANENT_BLOB or TRANSIENT_BLOB |
| **random** | AbstractID (```long``` + ```long```) |

## 2. BlobServerConnection

# 4. BlobCacheService (in TaskManager)

| Valye | type |
| --- | --- |
| **permanentBlobCache** | PermanentBlobCache |
| **transientBlobCache** | TransientBlobCache |

### 1. PermanentBlobCache
```
static class RefCount {
	public int references = 0;
	public long keepUntil = -1;
	}
	
private final Map<JobID, RefCount> jobRefCounters = new HashMap<>();
private final long cleanupInterval;
private final Timer cleanupTimer;
```
### 2. TransientBlobCache
```
private final ConcurrentHashMap<Tuple2<JobID, TransientBlobKey>, Long> blobExpiryTimes =
		new ConcurrentHashMap<>()
private final long cleanupInterval;
private final Timer cleanupTimer;
```
## 5. Protocol

- sender: 

| PUT(GET)_OPERATION | JOB_UNRELATED_CONTENT |
| --- | --- |
| 1 byte | 1 byte |

or 

| PUT(GET)_OPERATION | JOB_RELATED_CONTENT | JobID | BlobType  |
| --- | --- | --- | --- |
| 1 byte | 1 byte | long + long | PERMANENT_BLOB or TRANSIENT_BLOB |



- receiver:
| RETURN_OKAY | BlobKey |
| --- | --- |

or 
| --- | --- |
| RETURN_ERROR | BlobKey |

