---
title: "Building an LSM Tree Storage Engine from Scratch"
description: "Building an LSM Tree Storage Engine from Scratch"
publishDate: "2 Jan 2025"
tags: ["tutorial", "go", "programming", "database"]
---

## Preface

This article will guide you through understanding the Log-Structured Merge-Tree (LSM-Tree), including its core concepts and structure. By the end, you'll be able to build your own storage engine based on LSM-Tree from scratch.

## What is LSM-Tree?

### Basic Concepts

The Log-Structured Merge-Tree (LSM-Tree) is a **data structure optimized for high-throughput write operations**. It is widely used in databases and storage systems, such as Cassandra, RocksDB, and LevelDB.

The key idea of LSM-Tree is to first write operations into an in-memory data structure (typically an ordered structure like a skip list or AVL tree). Later, these writes are batched and sequentially written to disk as SSTables, thereby minimizing random I/O.

### Basic Structure

![lsm-struct](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1nb1zccql8z5lu7tvfez.png)

The LSM-Tree is divided into two main components:

- **In-Memory Storage**
    - The core structure in memory is the **Memtable**.
    - All write operations (e.g., set, delete) first go to the Memtable, which inserts these operations into an ordered data structure (e.g., an ordered tree in the diagram).
    - Once the Memtable reaches a certain size threshold, it is flushed to disk as an SSTable (written sequentially).
    - New write operations continue on a fresh Memtable.
- **Disk Storage**
    - Disk storage involves **WAL** and **SSTable** files.
    - The **WAL (Write-Ahead Log)** ensures that recent writes (stored in the Memtable but not yet persisted to disk) are not lost in case of a database crash. Every write to the Memtable is appended to the WAL. Upon restarting the database, entries from the WAL can be replayed in order to restore the Memtable to its pre-crash state.
    - The **SSTable (Sorted String Table)** is a data storage format that holds a series of ordered key-value pairs.
    - When the Memtable reaches its size threshold, it generates a new SSTable and persists it to disk. Since the Memtable relies on an in-memory **ordered data structure**, no additional sorting is required when constructing the SSTable.
    - SSTables on disk are organized into multiple levels. Newly flushed SSTables are stored in **Level 0**. During subsequent compaction phases, SSTables in L0 are merged into **Level 1** and higher levels.
    - When the size of a level exceeds a threshold, an SSTable compaction process is triggered. During compaction, SSTables in the current level are merged into higher levels, producing larger, more ordered files. This reduces fragmentation and improves query efficiency.

> Typically, the structure of an SSTable includes more than just a series of ordered key-value pairs (data blocks). It also contains an **index block**, **metadata block**, and other components. These details will be discussed in-depth during the implementation section.

### Writing Data

Writing data involves adding a new key-value pair or updating an existing one. Updates overwrite old key-value pairs, which are later removed during the compaction process.

When data is written, it first goes to the **Memtable**, where the key-value pair is added to the in-memory ordered data structure. Simultaneously, the write operation is logged in the **WAL** and persisted to disk to prevent data loss in the event of a database crash.

The Memtable has a defined threshold (usually based on size). When the Memtable exceeds this threshold, it is switched to **read-only mode** and converted into a new **SSTable**, which is then persisted to **Level 0** on disk.

Once the Memtable is flushed as an SSTable, the corresponding WAL file can be safely deleted. Subsequent write operations will proceed on a new Memtable (and a new WAL).

### Deleting Data

In LSM-Tree, data is not immediately removed. Instead, deletions are handled using a mechanism called **tombstones** (similar to soft deletes). When a key-value pair is deleted, a new entry marked with a "tombstone" is written, indicating the deletion of the corresponding key-value pair. The actual removal occurs during the compaction process.

This tombstone-based deletion ensures the **append-only** property of LSM-Tree, avoiding random I/O and maintaining sequential writes to disk.

### Querying Data

The process of querying data starts with a search in the **Memtable**. If the key-value pair is found, it is returned to the client. If a tombstone-marked key-value pair is found, it indicates that the requested data has been deleted, and this information is also returned. If the key is not found in the Memtable, the query proceeds to search the SSTables from **Level 0** to **Level N**.

Since querying data may involve searching multiple SSTable files and can lead to random disk I/O, LSM-Tree is generally better suited for **write-heavy** workloads rather than read-intensive ones.

One common optimization for query performance is the use of a **Bloom filter**. A Bloom filter can quickly determine whether a key-value pair exists in a specific SSTable, reducing unnecessary disk I/O. Additionally, the sorted nature of SSTables enables efficient search algorithms, such as binary search, to be employed for faster lookups.

### Data Compaction

Here, we introduce the **Leveled Compaction Strategy**, which is used by LevelDB and RocksDB.

> Another common strategy is the **Size-Tiered Compaction Strategy**, where newer and smaller SSTables are successively merged into older and larger SSTables.

As previously mentioned, an SSTable stores a series of entries sorted by key. In the Leveled Compaction Strategy, SSTables are organized into multiple levels (**Level 0 to Level N**).

**In Level 0, SSTables can have overlapping key ranges, as they are directly flushed from the Memtable. However, in Levels 1 through N, SSTables within the same level do not have overlapping key ranges, although key range overlaps are allowed between SSTables in different levels.**

An illustrative (though not entirely accurate) example is shown below. In **Level 0**, the key ranges of the first and second SSTables overlap, while in **Level 1** and **Level 2**, the SSTables within each level have disjoint key ranges. However, SSTables between different levels (e.g., Level 0 and Level 1, or Level 1 and Level 2) may have overlapping key ranges.

![levels](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/78y0tfox9ymwgeu01v7z.png)

Let’s now explore how the Leveled Compaction Strategy maintains this organizational structure.

Since Level 0 is a special case, the compaction strategy discussion is divided into two parts:

- **Level 0 to Level 1**
  Since Level 0 allows overlapping keys among SSTables, compaction begins by selecting an SSTable from Level 0, along with all other SSTables in Level 0 that have overlapping key ranges with it. Next, all SSTables in Level 1 with overlapping key ranges are selected. These selected SSTables are merged and compacted into a single new SSTable, which is then inserted into Level 1. All old SSTables involved in the compaction process are deleted.
- **Level N to Level N+1 (N > 0)**
  From Level 1 onwards, SSTables within the same level do not have overlapping key ranges. During compaction, an SSTable is selected from Level N, and all SSTables in Level N+1 with overlapping key ranges are also selected. These SSTables are merged and compacted into one or more new SSTables, which are inserted into Level N+1, while the old SSTables are deleted.

The primary difference between **Level 0 to Level 1** compaction and **Level N to Level N+1** (N > 0) compaction lies in the selection of SSTables at lower levels (Level 0 or Level N).

The multi-SSTable compaction and merging process is illustrated below. During the merge, only the latest value for each key is retained. If the latest value has a "tombstone" marker, the key is deleted. In the implementation, we use the **k-way merge algorithm** to perform this process.

![sst-merge](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wvqdb6zcmzumbdff9j09.png)

It is important to note that the above description of the compaction process provides only a high-level overview. Many details need to be addressed during actual implementation.

> For example, in LevelDB, when constructing new SSTables for Level N+1 during compaction, if the new SSTable overlaps with more than 10 SSTables in Level N+2, the process switches to constructing another SSTable. This limits the data size involved in a single compaction.

## Implementation

Based on the overview of LSM-Tree above, I believe you now have a basic understanding of LSM-Tree and some ideas about its implementation. Next, we will build a storage engine based on LSM-Tree from scratch. Below, we will introduce only the core code; for the complete code, please refer to https://github.com/B1NARY-GR0UP/originium.

We will break down the implementation of LSM-Tree into the following core components and implement them one by one:

- Skip List
- WAL
- Memtable
- SSTable
- K-Way Merge
- Bloom Filter
- Leveled Compaction

### Skip List

In the process of introducing data writing, we mentioned that the LSM-Tree first writes data into an in-memory ordered data structure. Some common ordered data structures and the time complexity of their operations are as follows:

| Data Structure     | Insert  | Delete  | Search  | Traverse |
| ------------------ | ------- | ------- | ------- | -------- |
| **Skip List**      | O(log⁡n) | O(log⁡n) | O(log⁡n) | O(n)     |
| **AVL Tree**       | O(log⁡n) | O(log⁡n) | O(log⁡n) | O(n)     |
| **Red-Black Tree** | O(log⁡n) | O(log⁡n) | O(log⁡n) | O(n)     |

We chose the Skip List for two main reasons: it is simpler to implement and maintain (KISS principle), and the underlying linked list structure facilitates sequential traversal, making it easier to persist in-memory data to disk.

#### Core Struct

The complete implementation of the Skip List is available at [https://github.com/B1NARY-GR0UP/originium/blob/main/pkg/skiplist/skiplist.go](https://github.com/B1NARY-GR0UP/originium/blob/main/pkg/skiplist/skiplist.go) .

A Skip List consists of a base linked list and multiple levels of indices built on top of it. For large datasets, the index layers significantly shorten the search path.

In our implementation, the core structure of the Skip List is defined as follows:

```go
type SkipList struct {
	maxLevel int
	p        float64
	level    int
	rand     *rand.Rand
	size     int
	head     *Element
}
```

- **maxLevel**: The maximum number of levels in the Skip List (the base linked list has one level).
- **level**: The current number of levels in the Skip List.
- **p**: The probability of a node being promoted to a higher level. For example, if `p = 0.5`, a linked list with 10 nodes at the base level will have approximately 5 nodes in the next level of indices.
- **rand**: A random number generator used to compare against `p`.
- **size**: The number of key-value pairs stored in the Skip List, used to determine if the Memtable exceeds its size threshold.
- **head**: The head node, which holds references to the first node in each level.

The structure of the elements stored in the Skip List is defined as follows:

```go
type Element struct {
	types.Entry
	next []*Element
}

// https://github.com/B1NARY-GR0UP/originium/blob/main/pkg/types/entry.go
type Entry struct {
	Key       string
	Value     []byte
	Tombstone bool  
}
```

-  `types.Entry` represents a key-value pair in the storage engine, including the key, value, and a tombstone flag for deletion.

- **next**: Contains pointers to the next element at each level.

This structure may seem abstract, so let's illustrate it with an example:

```
Level 3:       3 ----------- 9 ----------- 21 --------- 26
Level 2:       3 ----- 6 ---- 9 ------ 19 -- 21 ---- 25 -- 26
Level 1:       3 -- 6 -- 7 -- 9 -- 12 -- 19 -- 21 -- 25 -- 26

next of head [ ->3, ->3, ->3 ]
next of Element 3 [ ->6, ->6, ->9 ]
next of Element 6 [ ->7, ->9 ]
```

In this three-level Skip List, the `next` pointers of the head node reference the first node at each level. Elements 3 and 6 store the next element for each of their levels.

For example, if we want to find the next node of element 19 at Level 2, we use `e19.next[2-1]`.

#### Set

```go
func (s *SkipList) Set(entry types.Entry)
```

The LSM-Tree uses tombstones to perform deletions, so we don’t need a `Delete` method in the skip list implementation. To delete an element, simply set the `Tombstone` of the `entry` to `true`. Thus, the `Set` method handles inserting new key-value pairs, updating existing ones, and deleting elements.

Let’s explore the implementation of the `Set` method. By traversing the nodes in each level from the highest, the last element smaller than the key to be set is saved in the `update` slice.

```go
curr := s.head
update := make([]*Element, s.maxLevel)

for i := s.maxLevel - 1; i >= 0; i-- {
    for curr.next[i] != nil && curr.next[i].Key < entry.Key {
        curr = curr.next[i]
    }
    update[i] = curr
}
```

At the end of this traversal, `curr` points to the last element smaller than the key to be set in the bottom-level linked list. So, we only need to check if the next element of `curr` equals the key we want to set. If it matches, the element has already been inserted; we update the existing element and return.

```go
// update entry
if curr.next[0] != nil && curr.next[0].Key == entry.Key {
    s.size += len(entry.Value) - len(curr.next[0].Value)

    // update value and tombstone
    curr.next[0].Value = entry.Value
    curr.next[0].Tombstone = entry.Tombstone
    return
}
```

If the element is not found, it is inserted as a new element. Using `randomLevel`, we calculate the index level of this element. If it exceeds the current number of levels in the skip list, we add the head node to the `update` slice and update `s.level` to the new level count.

```go
// add entry
level := s.randomLevel()

if level > s.level {
    for i := s.level; i < level; i++ {
        update[i] = s.head
    }
    s.level = level
}
```

Next, construct the element to be inserted, and the `next` pointers of each level are updated to complete the insertion.

```go
e := &Element{
    Entry: types.Entry{
        Key:       entry.Key,
        Value:     entry.Value,
        Tombstone: entry.Tombstone,
    },
    next: make([]*Element, level),
}

for i := range level {
    e.next[i] = update[i].next[i]
    update[i].next[i] = e
}
s.size += len(entry.Key) + len(entry.Value) + int(unsafe.Sizeof(entry.Tombstone)) + len(e.next)*int(unsafe.Sizeof((*Element)(nil)))
```

#### Get

The skip list can perform fast search operations by relying on multiple layers of indexes. The nested `for` loops in the implementation represent the index-based search operation. If the corresponding element is eventually found in the bottom-level linked list, it will be returned.

```go
func (s *SkipList) Get(key types.Key) (types.Entry, bool) {
	curr := s.head

	for i := s.maxLevel - 1; i >= 0; i-- {
		for curr.next[i] != nil && curr.next[i].Key < key {
			curr = curr.next[i]
		}
	}

	curr = curr.next[0]

	if curr != nil && curr.Key == key {
		return types.Entry{
			Key:       curr.Key,
			Value:     curr.Value,
			Tombstone: curr.Tombstone,
		}, true
	}
	return types.Entry{}, false
}
```

#### All

One reason we chose the skip list is its convenient sequential traversal, which is made possible by simply traversing the bottom-level linked list.

```go
func (s *SkipList) All() []types.Entry {
	var all []types.Entry

	for curr := s.head.next[0]; curr != nil; curr = curr.next[0] {
		all = append(all, types.Entry{
			Key:       curr.Key,
			Value:     curr.Value,
			Tombstone: curr.Tombstone,
		})
	}

	return all
}
```

### WAL

The complete implementation of WAL can be found at [https://github.com/B1NARY-GR0UP/originium/blob/main/wal/wal.go](https://github.com/B1NARY-GR0UP/originium/blob/main/wal/wal.go).

As mentioned earlier, the purpose of WAL (Write-Ahead Logging) is to prevent data loss in the Memtable caused by database crashes. Therefore, WAL needs to record operations on the Memtable and recover the Memtable from the WAL file when the database restarts.

#### Core Struct

The core structure of WAL is as follows, where `fd` stores the file descriptor of the WAL file:

```go
type WAL struct {
	mu      sync.Mutex
	logger  logger.Logger
	fd      *os.File
	dir     string
	path    string
	version string
}
```

#### Write

Since we need to record operations on the Memtable, this essentially involves writing each operation (`Set`, `Delete`) as an `Entry` into the WAL. The definition of the `Write` method is as follows:

```go
func (w *WAL) Write(entries ...types.Entry) error
```

When writing these `entries` to the file, we need to standardize the WAL file format. The format we use here is **length + data**. First, we serialize the `Entry`, then calculate the length of the serialized data, and finally write the length and serialized data sequentially into the WAL file.

The core code is as follows:

```go
data, err := utils.TMarshal(&entry)
if err != nil {
    return err
}

// data length
n := int64(len(data))
err = binary.Write(buf, binary.LittleEndian, n)
if err != nil {
    return err
}
// data body
err = binary.Write(buf, binary.LittleEndian, data)
if err != nil {
    return err
}
```

#### Read

Since we use the WAL file format **length + data**, during reading, we first read 8 bytes (`int64`) to obtain the length of the data, and then read the data based on this length and deserialize it to retrieve the `Entry`.

The core code is as follows:

```go
var entries []types.Entry
reader := bytes.NewReader(buf.Bytes())
for reader.Len() > 0 {
    // data length
    var n int64
    if err = binary.Read(reader, binary.LittleEndian, &n); err != nil {
        return nil, err
    }

    // data body
    data := make([]byte, n)
    if err = binary.Read(reader, binary.LittleEndian, &data); err != nil {
        return nil, err
    }

    var entry types.Entry
    if err = utils.TUnmarshal(data, &entry); err != nil {
        return nil, err
    }
    entries = append(entries, entry)
}
```

### Memtable

The complete implementation of Memtable can be found at [https://github.com/B1NARY-GR0UP/originium/blob/main/memtable.go](https://github.com/B1NARY-GR0UP/originium/blob/main/memtable.go).

The Memtable is responsible for writing client operations into the skip list and recording them in the WAL. It can also recover data from the WAL when the database starts.

#### Core Struct

The core structure of the Memtable is as follows, which includes two main components `skiplist` and `wal`:

```go
type memtable struct {
	mu       sync.RWMutex
	logger   logger.Logger
	skiplist *skiplist.SkipList
	wal      *wal.WAL
	dir      string
	readOnly bool
}
```

#### Set

When performing a `Set` operation, both the skip list and the WAL need to be updated simultaneously.

```go
func (mt *memtable) set(entry types.Entry) {
	mt.mu.Lock()
	defer mt.mu.Unlock()

	mt.skiplist.Set(entry)
	if err := mt.wal.Write(entry); err != nil {
		mt.logger.Panicf("write wal failed: %v", err)
	}
	mt.logger.Infof("memtable set [key: %v] [value: %v] [tombstone: %v]", entry.Key, string(entry.Value), entry.Tombstone)
}
```

#### Get

To retrieve a value, simply return the result of the skip list's `Get` operation.

```go
func (mt *memtable) get(key types.Key) (types.Entry, bool) {
	mt.mu.RLock()
	defer mt.mu.RUnlock()

	return mt.skiplist.Get(key)
}
```

#### Recover

Recovering the Memtable from the WAL file involves first reading the WAL file, then sequentially applying the `Entry` records from the WAL file to the Memtable, and finally deleting the recovered WAL file.

Retrieve the list of WAL files:

```go
files, err := os.ReadDir(mt.dir)
if err != nil {
    mt.logger.Panicf("read dir %v failed: %v", mt.dir, err)
}

var walFiles []string
for _, file := range files {
    if !file.IsDir() && path.Ext(file.Name()) == ".log" && wal.CompareVersion(wal.ParseVersion(file.Name()), mt.wal.Version()) < 0 {
        walFiles = append(walFiles, path.Join(mt.dir, file.Name()))
    }
}
```

Read the WAL and recover the Memtable:

```go
for _, file := range walFiles {
    l, err := wal.Open(file)
    if err != nil {
        mt.logger.Panicf("open wal %v failed: %v", file, err)
    }
    entries, err := l.Read()
    if err != nil {
        mt.logger.Panicf("read wal %v failed: %v", file, err)
    }
    for _, entry := range entries {
        mt.skiplist.Set(entry)
        if err = mt.wal.Write(entry); err != nil {
            mt.logger.Panicf("write wal failed: %v", err)
        }
    }
    if err = l.Delete(); err != nil {
        mt.logger.Panicf("delete wal %v failed: %v", file, err)
    }
}
```

### SSTable

#### LevelDB SSTable

In the previous introduction, we only mentioned that "SSTable (Sorted String Table) is a data storage format that maintains a series of ordered key-value pairs." Here, we will provide a more detailed explanation of the structure of SSTable.

In LevelDB, an SSTable consists of multiple blocks with different purposes. A schematic diagram is shown below:

![leveldb-sst](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/esa14p7yd4ped831z09j.png)

- **Data Block**: Stores a sequence of ordered key-value pairs.
- **Meta Block**: Includes two types: `filter` and `stats`. The `filter` type stores data for Bloom filters, while the `stats` type stores statistical information about the Data Blocks.
- **MetaIndex Block**: Stores index information for the Meta Blocks.
- **Index Block**: Stores index information for the Data Blocks.
- **Footer**: Fixed in length, it stores the index information of the MetaIndex Block and Index Block, as well as a magic number.

The index information is essentially a pointer structure called a **BlockHandle**, which includes two attributes: `offset` and `size`, used to locate the corresponding Block.

#### Our SSTable

In our implementation of SSTable, we have simplified the LevelDB SSTable structure. A schematic diagram is shown below:

![our-sst](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ht1is81p0xpwogn2kui4.png)

- **Data Block**: Stores a sequence of ordered key-value pairs.
- **Meta Block**: Stores some metadata for the SSTable.
- **Index Block**: Stores index information for the Data Blocks.
- **Footer**: Fixed in length, it stores the index information of the Meta Block and Index Block.

The complete implementation of SSTable can be found at [https://github.com/B1NARY-GR0UP/originium/tree/main/sstable](https://github.com/B1NARY-GR0UP/originium/tree/main/sstable).

#### Data Block

The structure of the Data Block is defined as follows, storing an ordered sequence of entries.

```go
// Data Block
type Data struct {
	Entries []types.Entry
}
```

We implemented three primary methods for the Data Block:

- **Encode**: Encodes the Data Block into binary data.

```go
func (d *Data) Encode() ([]byte, error)
```

We use **prefix compression** to encode the key-value sequence. In the buffer, we sequentially write the length of the common prefix, the length of the suffix, the suffix itself, the length of the value, the value, and the "tombstone" marker.

```go
var prevKey string
for _, entry := range d.Entries {
    lcp := utils.LCP(entry.Key, prevKey)
    suffix := entry.Key[lcp:]

    // lcp
    if err := binary.Write(buf, binary.LittleEndian, uint16(lcp)); err != nil {
        return nil, err
    }

    // suffix length
    if err := binary.Write(buf, binary.LittleEndian, uint16(len(suffix))); err != nil {
        return nil, err
    }
    // suffix
    if err := binary.Write(buf, binary.LittleEndian, []byte(suffix)); err != nil {
        return nil, err
    }

    // value length
    if err := binary.Write(buf, binary.LittleEndian, uint16(len(entry.Value))); err != nil {
        return nil, err
    }
    // value
    if err := binary.Write(buf, binary.LittleEndian, entry.Value); err != nil {
        return nil, err
    }

    // tombstone
    tombstone := uint8(0)
    if entry.Tombstone {
        tombstone = 1
    }
    if err := binary.Write(buf, binary.LittleEndian, tombstone); err != nil {
        return nil, err
    }

    prevKey = entry.Key
}
```

Finally, we compress the data using **s2**.

> **S2** is a high-performance extension of the Snappy compression algorithm.

```go
// s2 compress
if err := utils.Compress(buf, compressed); err != nil {
    return nil, err
}
return compressed.Bytes(), nil
```

- **Decode**: Decodes binary data into a Data Block.

```go
func (d *Data) Decode(data []byte) error
```

During decoding, the process is simply reversed. The full key-value pair is reconstructed using the prefix and suffix.

```go
reader := bytes.NewReader(buf.Bytes())
var prevKey string
for reader.Len() > 0 {
    // lcp
    var lcp uint16
    if err := binary.Read(reader, binary.LittleEndian, &lcp); err != nil {
        return err
    }

    // suffix length
    var suffixLen uint16
    if err := binary.Read(reader, binary.LittleEndian, &suffixLen); err != nil {
        return err
    }
    // suffix
    suffix := make([]byte, suffixLen)
    if err := binary.Read(reader, binary.LittleEndian, &suffix); err != nil {
        return err
    }

    // value length
    var valueLen uint16
    if err := binary.Read(reader, binary.LittleEndian, &valueLen); err != nil {
        return err
    }
    // value
    value := make([]byte, valueLen)
    if err := binary.Read(reader, binary.LittleEndian, &value); err != nil {
        return err
    }

    var tombstone uint8
    if err := binary.Read(reader, binary.LittleEndian, &tombstone); err != nil {
        return err
    }

    key := prevKey[:lcp] + string(suffix)
    d.Entries = append(d.Entries, types.Entry{
        Key:       key,
        Value:     value,
        Tombstone: tombstone == 1,
    })

    prevKey = key
}
```

- **Search**: Locates key-value pairs using binary search.

```go
func (d *Data) Search(key types.Key) (types.Entry, bool) {
	low, high := 0, len(d.Entries)-1
	for low <= high {
		mid := low + ((high - low) >> 1)
		if d.Entries[mid].Key < key {
			low = mid + 1
		} else if d.Entries[mid].Key > key {
			high = mid - 1
		} else {
			return d.Entries[mid], true
		}
	}
	return types.Entry{}, false
}
```

#### Index Block

The structure of the Index Block is defined as follows. It stores the first and last key of each Data Block, along with the `BlockHandle` of the corresponding Data Block.

```go
// Index Block
type Index struct {
	// BlockHandle of all data blocks of this sstable
	DataBlock BlockHandle
	Entries   []IndexEntry
}

// IndexEntry include index of a sstable data block
type IndexEntry struct {
	// StartKey of each Data block
	StartKey string
	// EndKey of each Data block
	EndKey string
	// offset and length of each data block
	DataHandle BlockHandle
}

type BlockHandle struct {
	Offset uint64
	Length uint64
}
```

Similarly, the **Index Block** implements three primary methods: **Encode**, **Decode**, and **Search**. The implementation ideas for the Encode and Decode methods are essentially the same, so we will focus on the **Search** method.

The Search method of the Data Block is designed to locate a specific key-value pair within the ordered key-value sequence stored in a single Data Block. In contrast, the Search method of the Index Block is used to locate the Data Block containing the given key within the entire SSTable.

```go
// Search data block included the key
func (i *Index) Search(key types.Key) (BlockHandle, bool) {
	n := len(i.Entries)
	if n == 0 {
		return BlockHandle{}, false
	}

	// check if the key is beyond this sstable
	if key > i.Entries[n-1].EndKey {
		return BlockHandle{}, false
	}

	low, high := 0, n-1
	for low <= high {
		mid := low + ((high - low) >> 1)
		if i.Entries[mid].StartKey > key {
			high = mid - 1
		} else {
			if mid == n-1 || i.Entries[mid+1].StartKey > key {
				return i.Entries[mid].DataHandle, true
			}
			low = mid + 1
		}
	}
	return BlockHandle{}, false
}
```

#### Meta Block And Footer

```go
type Meta struct {
	CreatedUnix int64
	Level       uint64
}

type Footer struct {
	MetaBlock  BlockHandle
	IndexBlock BlockHandle
	Magic      uint64
}
```

The implementations of these two Blocks are quite straightforward, with both requiring only the Encode and Decode methods.

#### Build

After introducing all the Blocks in our SSTable, constructing an SSTable simply involves building each Block step by step based on the key-value pairs. Finally, the in-memory index and the encoded SSTable are returned.

```go
func Build(entries []types.Entry, dataBlockSize, level int) (Index, []byte)
```

### K-Way Merge

The complete implementation of K-Way Merge is available at [https://github.com/B1NARY-GR0UP/originium/tree/main/pkg/kway](https://github.com/B1NARY-GR0UP/originium/tree/main/pkg/kway).

In the conceptual section, we illustrate the process of compressing and merging multiple SSTables through diagrams. This process is accomplished using the **k-way merge** algorithm.

The k-way merge algorithm is a method to merge **k** sorted sequences into a single sorted sequence, with a time complexity of **O(knlogk)**.

One implementation of this algorithm uses a **min-heap** as an auxiliary structure:

- Insert the first element of each sequence into the heap.
- Pop the smallest value from the heap and add it to the result set. If the popped element's sequence still has more elements, insert the next element from that sequence into the heap.
- Repeat this process until all elements from all sequences are merged.

#### Heap

The standard library provides a heap implementation in `container/heap`. By implementing the `heap.Interface`, we can build a min-heap.

- First, define the basic structure of the min-heap. A slice is used to store the elements. Each element includes not only an `Entry` but also an `LI` to indicate which sorted sequence this element originates from.

```go
type Element struct {
	types.Entry
	// list index
	LI int
}

// Heap min heap
type Heap []Element
```

- Implement the `sort.Interface` methods to sort elements in the heap. Special attention is needed for the `Less` method: by comparing the `LI` of the elements, we ensure that when elements have the same key, those from earlier sequences are ordered first. This facilitates deduplication when merging elements into the result set. This requirement also means that the sorted sequences should be arranged in order from oldest to newest when using the k-way merge algorithm.

```go
func (h *Heap) Len() int {
	return len(*h)
}

func (h *Heap) Less(i, j int) bool {
	return (*h)[i].Key < (*h)[j].Key && (*h)[i].LI == (*h)[j].LI || (*h)[i].Key == (*h)[j].Key && (*h)[i].LI < (*h)[j].LI
}

func (h *Heap) Swap(i, j int) {
	(*h)[i], (*h)[j] = (*h)[j], (*h)[i]
}
```

- Finally, implement the `Push` and `Pop` methods. `Push` appends an element to the end of the slice, while `Pop` removes the last element from the slice.

```go
func (h *Heap) Push(x any) {
	*h = append(*h, x.(Element))
}

// Pop the minimum element in heap
// 1. move the minimum element to the end of slice
// 2. pop it (what this method does)
// 3. heapify
func (h *Heap) Pop() any {
	curr := *h
	n := len(curr)
	e := curr[n-1]
	*h = curr[0 : n-1]
	return e
}
```

#### Merge

The function definition of the `Merge` method:

```go
func Merge(lists ...[]types.Entry) []types.Entry
```

Follows the k-way merge algorithm process.

- First, insert the first element of each sorted sequence into the min-heap.

```go
h := &Heap{}
heap.Init(h)

// push first element of each list
for i, list := range lists {
    if len(list) > 0 {
        heap.Push(h, Element{
            Entry: list[0],
            LI:    i,
        })
        lists[i] = list[1:]
    }
}
```

- Iteratively pop an element from the min-heap and add it to the result queue. If the popped element’s sequence still has more elements, insert the next element from that sequence into the heap. Here, a `map` is used instead of a result sequence. The `map` automatically handles deduplication, with newer keys always overwriting older ones.

```go
latest := make(map[string]types.Entry)

for h.Len() > 0 {
    // pop minimum element
    e := heap.Pop(h).(Element)
    latest[e.Key] = e.Entry
    // push next element
    if len(lists[e.LI]) > 0 {
        heap.Push(h, Element{
            Entry: lists[e.LI][0],
            LI:    e.LI,
        })
        lists[e.LI] = lists[e.LI][1:]
    }
}
```

Finally, traverse the `map` to add elements to the result queue, removing any key-value pairs marked as "tombstones." Since the `map` is unordered, the result queue needs to be sorted before being returned.

```go
var merged []types.Entry

for _, entry := range latest {
    if entry.Tombstone {
        continue
    }
    merged = append(merged, entry)
}

slices.SortFunc(merged, func(a, b types.Entry) int {
    return cmp.Compare(a.Key, b.Key)
})

return merged
```

### Bloom Filter

The complete implementation of the Bloom Filter can be found at [https://github.com/B1NARY-GR0UP/originium/blob/main/pkg/filter/filter.go](https://github.com/B1NARY-GR0UP/originium/blob/main/pkg/filter/filter.go).

A Bloom Filter is a data structure that efficiently checks whether an element is a member of a set.

- It uses a bit array and multiple hash functions.
- When adding an element, the element is hashed using multiple hash functions, which map it to different positions in the bit array, setting those positions to 1.
- During a query, if all positions mapped by the hash functions are 1, the element might exist. If any position is 0, the element definitely does not exist.

The time complexity for both insertion and query operations is **O(k)**, where **k** is the number of hash functions. **False positives may occur** (the Bloom Filter predicts that an element is in the set, but it is not), but **false negatives cannot occur** (the Bloom Filter predicts that an element is not in the set, but it actually is).

> **True Positive (TP):** The system predicts an event as "positive," and it is indeed positive.
> **False Positive (FP):** The system predicts an event as "positive," but it is actually negative.
> **True Negative (TN):** The system predicts an event as "negative," and it is indeed negative.
> **False Negative (FN):** The system predicts an event as "negative," but it is actually positive.

#### Core Struct

The core structure of a Bloom Filter includes the bit array (which can be optimized to use `uint8`) and multiple hash functions.

```go
type Filter struct {
	bitset  []bool
	hashFns []hash.Hash32
}
```

#### New

The method to create a Bloom Filter instance accepts two parameters: `n` (the expected number of elements) and `p` (the desired false positive rate).

First, the parameters are validated. Then, the size of the bit array (`m`) and the number of hash functions (`k`) are calculated using specific formulas. Finally, the bit array and hash functions are initialized based on `m` and `k`.

```go
// New creates a new BloomFilter with the given size and number of hash functions.
// n: expected nums of elements
// p: expected rate of false errors
func New(n int, p float64) *Filter {
	if n <= 0 || (p <= 0 || p >= 1) {
		panic("invalid parameters")
	}

	// size of bitset
	// m = -(n * ln(p)) / (ln(2)^2)
	m := int(math.Ceil(-float64(n) * math.Log(p) / math.Pow(math.Log(2), 2)))
	// nums of hash functions used
	// k = (m/n) * ln(2)
	k := int(math.Round((float64(m) / float64(n)) * math.Log(2)))

	hashFns := make([]hash.Hash32, k)
	for i := range k {
		hashFns[i] = murmur3.New32WithSeed(uint32(i))
	}

	return &Filter{
		bitset:  make([]bool, m),
		hashFns: hashFns,
	}
}
```

#### Add

When adding an element, all hash functions are used to compute hash values for the `key`. These values are then mapped to indices in the bit array, and the corresponding positions are set to `true`.

```go
// Add adds an element to the BloomFilter.
func (f *Filter) Add(key string) {
	for _, fn := range f.hashFns {
		_, _ = fn.Write([]byte(key))
		index := int(fn.Sum32()) % len(f.bitset)
		f.bitset[index] = true
		fn.Reset()
	}
}
```

#### Contains

To check if a `key` is in the set, the hash functions compute hash values and map them to indices in the bit array. If any of these positions is not `true`, the element is not in the set, and `false` is returned.

```go
// Contains checks if an element is in the BloomFilter.
func (f *Filter) Contains(key string) bool {
	for _, fn := range f.hashFns {
		_, _ = fn.Write([]byte(key))
		index := int(fn.Sum32()) % len(f.bitset)
		fn.Reset()
		if !f.bitset[index] {
			return false
		}
	}
	return true
}
```

### Leveled Compaction

The complete implementation of Leveled Compaction can be found at [https://github.com/B1NARY-GR0UP/originium/blob/main/level.go](https://github.com/B1NARY-GR0UP/originium/blob/main/level.go).

After implementing components like K-Way Merge and Bloom Filter, we can complete the final part of our implementation, the most crucial SSTable compaction process in the LSM-Tree storage engine. This implementation follows the **Leveled Compaction Strategy** described in the "Data Compaction" section.

In the Leveled Compaction Strategy, SSTables are organized into multiple levels (Level 0 - Level N). We need a structure to store this information and manage SSTables across different levels.

Thus, we implemented a structure called `levelManager`. We use a `[]*list.List` to store SSTable information for each level, where the index of the slice corresponds to the level. Each element in the slice is a `list.List`, a doubly-linked list that holds information about all SSTables in a specific level.

```go
type levelManager struct {
	mu            sync.Mutex
	dir           string
	l0TargetNum   int
	ratio         int
	dataBlockSize int
	// list.Element: tableHandle
	levels []*list.List
	logger logger.Logger
}

type tableHandle struct {
	// list index of table within a level
	levelIdx int
	// bloom filter
	filter filter.Filter
	// index of data blocks in this sstable
	dataBlockIndex sstable.Index
}
```

#### compactLN

The `compactLN` method is responsible for Level N to Level N+1 (N > 0) compaction. It selects the oldest SSTable from LN and all SSTables from LN+1 that have overlapping key ranges with this SSTable.

```go
lnTable := lm.levels[n].Front()
start, end := boundary(lnTable)

// overlap sstables in LN+1
ln1Tables := lm.overlapLN(n+1, start, end)
```

The selected SSTables are processed in order from oldest to newest. Key-value pairs from their data blocks are added to a two-dimensional slice and then merged using the K-Way Merge algorithm.

```go
// old -> new (append LN+1 first)
var dataBlockList [][]types.Entry
// LN+1 data block entries
for _, table := range ln1Tables {
    th := table.Value.(tableHandle)
    dataBlockLN1 := lm.fetch(n+1, th.levelIdx, th.dataBlockIndex.DataBlock)
    dataBlockList = append(dataBlockList, dataBlockLN1.Entries)
}
// LN data block entries
dataBlockLN := lm.fetch(n, lnTable.Value.(tableHandle).levelIdx, lnTable.Value.(tableHandle).dataBlockIndex.DataBlock)
dataBlockList = append(dataBlockList, dataBlockLN.Entries)

// merge sstables
mergedEntries := kway.Merge(dataBlockList...)
```

With the merged key-value pairs, we construct a new Bloom Filter and SSTable. The relevant information about the new SSTable is appended to the end of Level N+1.

```go
// build new bloom filter
bf := filter.Build(mergedEntries)
// build new sstable
dataBlockIndex, tableBytes := sstable.Build(mergedEntries, lm.dataBlockSize, n+1)

// table handle
th := tableHandle{
    levelIdx:       lm.maxLevelIdx(n+1) + 1,
    filter:         *bf,
    dataBlockIndex: dataBlockIndex,
}

// update index
// add new index to LN+1
lm.levels[n+1].PushBack(th)
```

Finally, the old SSTables are deleted, and the newly constructed SSTable is written to disk.

```go
// remove old sstable index from LN
lm.levels[n].Remove(lnTable)
// remove old sstable index from LN+1
for _, e := range ln1Tables {
    lm.levels[n+1].Remove(e)
}

// delete old sstables from LN
if err := os.Remove(lm.fileName(n, lnTable.Value.(tableHandle).levelIdx)); err != nil {
    lm.logger.Panicf("failed to delete old sstable: %v", err)
}
// delete old sstables from LN+1
for _, e := range ln1Tables {
    if err := os.Remove(lm.fileName(n+1, e.Value.(tableHandle).levelIdx)); err != nil {
        lm.logger.Panicf("failed to delete old sstable: %v", err)
    }
}

// write new sstable
fd, err := os.OpenFile(lm.fileName(n+1, th.levelIdx), os.O_CREATE|os.O_RDWR|os.O_TRUNC, 0600)
if err != nil {
    lm.logger.Panicf("failed to open sstable: %v", err)
}
defer func() {
    if err = fd.Close(); err != nil {
        lm.logger.Errorf("failed to close file: %v", err)
    }
}()

_, err = fd.Write(tableBytes)
```

The `compactL0` method handles Level 0 to Level 1 compaction. Unlike `compactLN`, it selects not only one SSTable from L0 but also all overlapping SSTables in L0. The rest of the process is identical to `compactLN`.

#### search

The `search` method locates the corresponding key-value pairs across all SSTables. It starts from L0 and iterates through each level up to LN. By leveraging the Bloom Filter and the ordered structure of SSTables, SSTables that do not contain the desired key-value pairs can be skipped efficiently.

```go
func (lm *levelManager) search(key types.Key) (types.Entry, bool) {
	lm.mu.Lock()
	defer lm.mu.Unlock()

	if len(lm.levels) == 0 {
		return types.Entry{}, false
	}

	for level, tables := range lm.levels {
		for e := tables.Front(); e != nil; e = e.Next() {
			th := e.Value.(tableHandle)

			// search bloom filter
			if !th.filter.Contains(key) {
				// not in this sstable, search next one
				continue
			}

			// determine which data block the key is in
			dataBlockHandle, ok := th.dataBlockIndex.Search(key)
			if !ok {
				// not in this sstable, search next one
				continue
			}

			// in this sstable, search according to data block
			entry, ok := lm.fetchAndSearch(key, level, th.levelIdx, dataBlockHandle)
			if ok {
				return entry, true
			}
		}
	}

	return types.Entry{}, false
}
```

### DB

With this, we have implemented all core components of the LSM-Tree-based storage engine. By assembling these components as described in the LSM-Tree introduction, we can finalize the database interface.

- Complete code: [https://github.com/B1NARY-GR0UP/originium/blob/main/db.go](https://github.com/B1NARY-GR0UP/originium/blob/main/db.go)

- Documentation: [https://github.com/B1NARY-GR0UP/originium?tab=readme-ov-file#usage](https://github.com/B1NARY-GR0UP/originium?tab=readme-ov-file#usage)

## Summary

We began by understanding LSM-Tree, familiarizing ourselves with its core components and the process of handling client requests. Ultimately, we built our own LSM-Tree storage engine from scratch.

Of course, this implementation is just a prototype. A production-grade storage engine requires consideration of many more details. ORIGINIUM will continue to receive optimizations and improvements in the future. Hope this article and ORIGINIUM help deepen your understanding of LSM-Tree.

That concludes everything covered in this article. If there are any errors or questions, feel free to reach out via private message or leave a comment. Thank you!

## Reference

- https://github.com/B1NARY-GR0UP/originium
- https://www.cnblogs.com/whuanle/p/16297025.html
- https://vonng.gitbook.io/vonng/part-i/ch3#sstables-he-lsm-shu
- https://github.com/google/leveldb/blob/main/doc/table_format.md

