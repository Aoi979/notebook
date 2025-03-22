# LevelDB笔记

> [!IMPORTANT]
>
> 阅读所需基础知识: 
>
> - [LSM-Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)
> - [SkipList](https://oi-wiki.org/ds/skiplist/)
>



[TOC]



## 核心接口分析

db_impl.cc

| 函数                                                         | 作用                   | 备注                                                         |
| ------------------------------------------------------------ | ---------------------- | ------------------------------------------------------------ |
| **Status DBImpl::Write(const WriteOptions& options, WriteBatch* updates)** | 将需要存储的kv对写入   | 初始化一个writer,持有mutex，为writer设置需要写入的KV对以及任务的状态信息，此后上锁，将自身加入writers队列，这是等待写入的任务队列，writer若是非完成状态且不是队列的最前端那么将进入等待，至此这个写任务正式开始处理，通过MakeRoomForWrite函数检查memTable的状态，处理时尝试将当前写任务和正在等待的写任务合并处理，处理完这步就释放锁，把数据写入Log,随后写入memTable,此刻其他写任务也能正常进入任务队列，随后又上锁，开始处理错误以及其他所需的元信息，此时数据已经写入memTable，只需要把刚才批量处理掉的任务以及自身移出任务队列即可，此时若队列非空，那就唤醒头部的任务（writer） |
| **Status DBImpl::MakeRoomForWrite(bool force) **             | 确保memTable能正常使用 | 开始<br/>├─ 1. 是否有后台错误？ → 返回错误<br/>│<br/>├─ 2. L0 文件数是否触发了写入减速？ → 延迟 1ms<br/>│<br/>├─ 3. 当前内存表是否有空间？ → 允许写入<br/>│<br/>├─ 4. 是否有未完成的不可变内存表（imm_）压缩？ → 等待压缩完成<br/>│<br/>├─ 5. L0 文件数是否触发了停止写入？ → 等待压缩完成<br/>│<br/>└─ 6. 其他情况 → 切换内存表 + 触发压缩 |
| **Status DBImpl::Get(const ReadOptions& options, const Slice& key,std::string* value) ** | 读取KV                 | 尝试在memTable，immuMemTable,SSTable读,若在SSTable读需要更新统计信息，同时读取的统计信息可能会触发压缩条件，所以需要尝试压缩 |

## MemTable 实现分析

MemTable的实现依赖于SkipList,这是一种媲美红黑树的数据结构

下面给出一个简单的实现

```cpp
class Skiplist {
public:
    struct Node {
        int value;
        std::vector<Node *> forward;    
    	Node(int value, int level) : value(value), forward(level, nullptr) {}
};

Skiplist() {
    srand(time(0));
    max_level = 4;
    header = new Node(-1, max_level);
    level = 0;
}

bool search(int target) {
    Node *current = header;
    for (int i = level; i >= 0; --i) {
        while (current->forward[i] != nullptr &&
               current->forward[i]->value < target) {
            current = current->forward[i];
        }
        if (current->forward[i] != nullptr &&
            current->forward[i]->value == target) {
            return true;
        }
    }
    return false;
}

void add(int num) {
    Node *current = header;
    std::vector<Node *> list(max_level, nullptr);
    for (int i = level; i >= 0; --i) {
        while (current->forward[i] != nullptr &&
               current->forward[i]->value < num) {
            current = current->forward[i];
        }
        list[i] = current->forward[i];
    }
    int node_lv = randomLevel();
    if (node_lv > level) {
        for (int i = level + 1; i <= node_lv; i++) {
            list[i] = header->forward[i];
        }
        level = node_lv;
    }
    Node *new_node = new Node(num, node_lv + 1);
    for (int i = 0; i <= node_lv; i++) {
        new_node->forward[i] = list[i];
        list[i] = new_node;
    }
}

bool erase(int num) {
    std::vector<Node *> update(max_level, nullptr);
    Node *current = header;
    for (int i = level; i >= 0; --i) {
        while (current->forward[i] != nullptr &&
               current->forward[i]->value < num) {
            current = current->forward[i];
        }
        update[i] = current;
    }

    current = current->forward[0];
    if (current != nullptr && current->value == num) {
        for (int i = 0; i <= level; ++i) {
            if (update[i]->forward[i] != current) {
                break;
            }
            update[i]->forward[i] = current->forward[i];
        }

        delete current; 
        while (level > 0 && header->forward[level] == nullptr) {
            --level;
        }

        return true; 
    }
    return false;  
}

int randomLevel() {
    int lv = 0;
    while (rand() % 2 && lv < max_level - 1) {
        lv++;
    }
    return lv;
}
private:
    int level;
    int max_level;
    Node *header;
};    
```

### 编码格式

| key_len | key  | tag  | value_len | value |
| ------- | ---- | ---- | --------- | ----- |

**tag = sequence(7B) + ValueType(1B)**

都使用varint编码

| 函数                                                         | 作用               | 具体描述                                                     |
| ------------------------------------------------------------ | ------------------ | ------------------------------------------------------------ |
| **void MemTable::Add(SequenceNumber s, ValueType type, const Slice& key, const Slice& value) ** | 写入数据到memTable | 编码并插入跳表                                               |
| **bool MemTable::Get(const LookupKey& key, std::string* value, Status* s)** | 查询               | 通过内部跳表初始化一个迭代器，在跳表中查找，找到后再根据写入的格式进行解码，最后根据数据的tag判断，如果数据是删除状态的就返回不存在 |

## WAL日志分析

日志文件数据按32KB固定大小的Block写入，写入的数据称为Record

### 编码格式

| Header                         | Raw                          |
| ------------------------------ | ---------------------------- |
| checksum  4B   \|   length  2B | type  1B \| data    length B |

当遇到的内容非常大的数据，或者当前的Block剩余空间不足以存储当前要存储的数据时，会先将该数据进行分段，然后组织成多条Record存入多个Block中，为了标识Record的类型引入了type,他是一个枚举类型

| Name        | value | note                              |
| ----------- | ----- | --------------------------------- |
| kFullType   | 1     | 表示该Record完整存储在当前block中 |
| kFirstType  | 2     | 表示为第一个分段                  |
| kMiddleType | 3     | 表示为中间分段                    |
| kLastType   | 4     | 表示为最后一个分段                |

Writer用来追加写入数据，Reader在启动后调用，通过从WAL日志中读取数据来尝试恢复之前内存中可能丢失的数据

### Writer的实现

WAL日志通过这个结构对外提供AddRecord(data)的接口，负责编码和写入文件

```cpp
Status Writer::AddRecord(const Slice& slice) {
  const char* ptr = slice.data();
  size_t left = slice.size();

  // Fragment the record if necessary and emit it.  Note that if slice
  // is empty, we still want to iterate once to emit a single
  // zero-length record
  Status s;
  bool begin = true;
  do {
    const int leftover = kBlockSize - block_offset_;
    assert(leftover >= 0);
    if (leftover < kHeaderSize) {
      // Switch to a new block
      if (leftover > 0) {
        // Fill the trailer (literal below relies on kHeaderSize being 7)
        static_assert(kHeaderSize == 7, "");
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
      }
      block_offset_ = 0;
    }

    // Invariant: we never leave < kHeaderSize bytes in a block.
    assert(kBlockSize - block_offset_ - kHeaderSize >= 0);

    const size_t avail = kBlockSize - block_offset_ - kHeaderSize;
    const size_t fragment_length = (left < avail) ? left : avail;

    RecordType type;
    //如果当前段的长度和待写入数据的剩余长度相等，说明是最后一个record
    const bool end = (left == fragment_length);
      //如果begin和end都为true,说明一次就能写完
    if (begin && end) {
      type = kFullType;
    } else if (begin) {
        //当前是第一段
      type = kFirstType;
    } else if (end) {
        //当前是最后段
      type = kLastType;
    } else {
        //是中间段
      type = kMiddleType;
    }
      //将ptr~ptr+fragment_length写入log
    s = EmitPhysicalRecord(type, ptr, fragment_length);
      //更新指针
    ptr += fragment_length;
    left -= fragment_length;
    begin = false;
  } while (s.ok() && left > 0);
  return s;
}

```

没啥好介绍的，下面是写入逻辑，通过dest_的append和flush完成

```cpp
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr,
                                  size_t length) {
  assert(length <= 0xffff);  // Must fit in two bytes
  assert(block_offset_ + kHeaderSize + length <= kBlockSize);

  // Format the header
  char buf[kHeaderSize];
  buf[4] = static_cast<char>(length & 0xff);
  buf[5] = static_cast<char>(length >> 8);
  buf[6] = static_cast<char>(t);

  // Compute the crc of the record type and the payload.
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, length);
  crc = crc32c::Mask(crc);  // Adjust for storage
  EncodeFixed32(buf, crc);

  // Write the header and the payload
  Status s = dest_->Append(Slice(buf, kHeaderSize));
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, length));
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  block_offset_ += kHeaderSize + length;
  return s;
}

```



### WritableFile实现

是顺序写文件的抽象结构

注意当调用Append(data)追加完数据以后，上层应用程序需要手动调用Flush()方法来触发缓冲区中的数据写入磁盘。同时LevelDB提供了处理写操作的WriteOptions参数，可选地设置参数sync的值。如果sync设置为true，那么写入WAL日志后还会同步调用WritableFile结构的Sync()方法确保数据一定写到磁盘上。因为系统默认数据会先写入操作系统的缓冲区，然后在将来的某个时刻，操作系统会将缓冲区中的数据刷到磁盘上。如果数据未刷到磁盘之前操作系统宕机了，那么数据仍然有丢失的风险。如果只是进程崩溃了，操作系统正常运行，则不会有风险。为了确保数据一定能写入磁盘，可以在调用LevelDB写入数据时将WriteOptions参数中的sync设置为true。

### Reader实现

主要是ReadRecord(record)方法，分为二步，从文件以block为单位加载到buffer,解码数据，重建写入前的数据，如果是分段的那就合并



## SSTable实现

SSTable一般由Minor压缩和Major压缩生成，不管怎么样他都是只读文件，生成后不会改变，且他的数据是有序的，查询的时候需要使用这个性质

### SSTable 结构

| Data Block1      | 数据                                    | 存储KV数据                                     |
| ---------------- | --------------------------------------- | ---------------------------------------------- |
| Data Block2      | 数据                                    | 存储kv数据                                     |
| ......           | 数据                                    | 存储kv数据                                     |
| DataBlock n      | 数据                                    | 存储kv数据                                     |
| Filter Block     | 数据                                    | 布隆过滤器                                     |
| Meta Index Block | 索引 -> Filter Block                    | 存储Filter Block的索引信息                     |
| Index Block      | 索引 -> Data Block n                    | 存储 Data Block的索引信息                      |
| Footer           | 索引 -> Index Block \| Meta Index Block | 存储 Meta Index Block 和 Index Block的索引信息 |

### Block结构

每个Block默认大小为4KB,由数据，压缩类型，校验码组成，默认采用Snappy压缩算法

| Data | CompressionType | CRC  |
| ---- | --------------- | ---- |

### DataBlock

| kv 1                | kv数据     |
| ------------------- | ---------- |
| kv 2                | kv数据     |
| ......              | kv数据     |
| kv n                | kv数据     |
| restart offset 1(4) | 重启点数据 |
| restart offset 2(4) | 重启点数据 |
| ......              | 重启点数据 |
| restart offset m(4) | 重启点数据 |
| restart count(4)    | 重启点数据 |
| Type(1)             | 压缩类型   |
| CRC(4)              | 校验码     |

在DataBlock中的KV数据如下

| K的共享长度    | K的未共享长度    | V的长度   | K的未共享内容 | V的内容 |
| -------------- | ---------------- | --------- | ------------- | ------- |
| shared_key_len | unshared_key_len | value_len | unshared key  | value   |

LevelDB中多条KV数据之间是按照K的顺序有序存储的，这意味着相邻的多条数据之间key的内容可能会存在相同部分，因此leveldb在存储每条kv数据时对k部分进行压缩，将key分为两部分：第一部分是当前的k和前一条kv的k重复的部分，第二部分是不重复部分。

这样操作，空间得到了优化，问题是获取一条完整的kv数据需要对k进行拼接，如果全部的kv数据都按照这个方式存储那么恢复k的过程很长，所以引入重启点这个概念。通过设定一个数据间隔，每隔几条数据就记录一个完整的k,然后再按照上述方式进行压缩，当达到间隔后再记录一次完整的k,不断重复

### 重启点数据

重启点数据由多个重启项和重启点个数两个部分组成，大小均为4B。每个重启项记录的是该条重启点的KV数据写入DataBlock中的位置。通过该重启点就可以直接读取该条KV数据的完整数据。位于两个重启点之间的数据在恢复的时候需要顺序遍历逐个恢复。重启点个数记录的就是当前DataBlock总共存储的重启点的个数。重启点间隔通过参数来配置，默认16,16条数据就要保存一个重启点，这是一个超参数

### Index Block

一个SSTable中有多个Data Block存储kv数据，满了之后就需要打开另一个文件继续写，采用Index Block来完成对已满的Data Block进行索引，他和**<u>Data Block的结构完全一样</u>**，不难设想对Data Block索引信息为key--offset--length，key表示当前DataBlock保存的所有KV数据中K的最大值，offset和length表示该DataBlock写入SSTable的位置以及长度。实际上LevelDB索引信息的K是当前DataBlock的K的最大值（最后一个）和下一个DataBlock的k的最小值（第一个）的最短分隔符，这样的设计可以减少占用空间。V是经过BlockHandle结构编码后的内容，就是封装了前面的offset和length

### Filter Block

就是布隆过滤器，存在误判，发生误判概率为
$$
p = [1-(1-\frac{1}{m})^{kn}]^k \approx (1-e^{\frac{kn}{m}})^k
$$
m为位数组大小，n为元素数量，k为哈希函数个数，最优k如下
$$
当k=0.7\frac{m}{n}时误判率最低，p=f(k)=(1-e^{\frac{kn}{m}})^k=2^{-ln2\frac{m}{n}} \approx (0.6158)^{\frac{m}{n}}
$$
可以得到
$$
m=-\frac{nlnp}{{(ln2})^2}
$$
结构如下

| filter 1           | 过滤器内容   |
| ------------------ | ------------ |
| filter 2           | 过滤器内容   |
| ......             | 过滤器内容   |
| filter n           | 过滤器内容   |
| filter offset 1(4) | 过滤器偏移量 |
| filter offset 2(4) | 过滤器偏移量 |
| ......             | 过滤器偏移量 |
| filter offset n(4) | 过滤器偏移量 |
| filter data size   | 过滤器元信息 |
| filter base(1)     | 过滤器元信息 |
| Type(1)            | 压缩类型     |
| CRC(4)             | 检验码       |

对于SSTable而言，每2kb的kv数据就会生成一个布隆过滤器，布隆过滤器的相关数据存入filter。过滤器偏移量记录每个过滤器内容写入的位置，根据前后两个过滤器的偏移量就可以获取对应的过滤器的内容，每个偏移量使用4B存储。过滤器元数据主要包含过滤器内容大小和过滤器基数，过滤器内容大小用4B存储，主要记录过滤器内容所占大小，过滤器基数在LevelDB中是一个常数11,代表2的11次方，即每2KB分配一个布隆过滤器

### Meta Index Block

当Filter Block写完后也需要记录其索引信息，该索引信息在SSTable中是采用Meta Index Block,以KV数据存储的，其**<u>和Data Block结构一致</u>**，Filter Block的索引信息中K为 “filter."加上布隆过滤器的名字。V也是一个Block Handle结构，存储Filter Block在SSTable中写入的位置和长度。通过这个可以得到Filter Block的完整内容

### Footer

结构如下

| Meta Index Block 的索引 |
| ----------------------- |
| Index Block 的索引      |
| Padding 填充            |
| Magic 魔数              |

## Block的写入

从上面可以得知，只有Filter Block的结构和其他Block不同，所以对于Filter Block他的读写需要单独的Builder，这里分为FilterBlockBuilder和BlockBuilder,读取则是FilterBlockReader和Block::Iter

### BlockBuilder结构

table/block_builder.cc

核心函数

| void BlockBuilder::Add(const Slice& key, const Slice& value) | 添加KV数据    |
| ------------------------------------------------------------ | ------------- |
| Slice BlockBuilder::Finish()                                 | 返回完整Block |

```c++
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  Slice last_key_piece(last_key_);
  assert(!finished_);// 确保未调用 Finish()
  assert(counter_ <= options_->block_restart_interval);
  assert(buffer_.empty()  // No values yet?
         || options_->comparator->Compare(key, last_key_piece) > 0);// 保证有序
  size_t shared = 0;
  if (counter_ < options_->block_restart_interval) {
    // See how much sharing to do with previous string
    // 计算当前 key 与前一个 key 的共享前缀长度
    const size_t min_length = std::min(last_key_piece.size(), key.size());
    while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
      shared++;
    }
  } else {
    // 达到重启间隔，记录重启点并重置计数器
    // Restart compression
    restarts_.push_back(buffer_.size());
    counter_ = 0;
  }
  const size_t non_shared = key.size() - shared;

  // Add "<shared><non_shared><value_size>" to buffer_
  PutVarint32(&buffer_, shared);
  PutVarint32(&buffer_, non_shared);
  PutVarint32(&buffer_, value.size());

  // Add string delta to buffer_ followed by value
  buffer_.append(key.data() + shared, non_shared);
  buffer_.append(value.data(), value.size());

  // Update state
  last_key_.resize(shared);
  last_key_.append(key.data() + shared, non_shared);
  // 更新 last_key_ 为当前 key
  assert(Slice(last_key_) == key);
  counter_++;
}
//添加重启点数据进Block
Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    PutFixed32(&buffer_, restarts_[i]);
  }
  PutFixed32(&buffer_, restarts_.size());
  finished_ = true;
  return Slice(buffer_);
}
```

### FilterBlockBuilder结构

table/filter_block.cc

成员变量

|       变量名        |         类型          |                             描述                             |
| :-----------------: | :-------------------: | :----------------------------------------------------------: |
|     **policy_**     |  const FilterPolicy*  | 指向**过滤器策略**的指针（如 Bloom Filter 的具体实现），负责创建和检查过滤器。 |
|      **keys_**      |      std::string      | **扁平化存储所有键的字符串**，例如将多个键连续存储为 `"key1key2key3..."`。 |
|     **start_**      |  std::vector<size_t>  | 记录每个键在 `keys_` 中的**起始位置索引**，用于快速定位键内容（例如 `start_ = [0, 4, 8]` 表示三个键分别从 0、4、8 字节开始）。 |
|     **result_**     |      std::string      | **最终生成的 Filter Block 数据**，包含所有过滤器二进制内容及元数据。 |
|    **tmp_keys_**    |  std::vector<Slice>   | 临时存储当前数据块的键集合，用于调用 `policy_->CreateFilter()` 生成单个过滤器。 |
| **filter_offsets_** | std::vector<uint32_t> | 记录每个过滤器在 `result_` 中的**偏移量**（起始位置），用于快速定位特定数据块的过滤器。 |

核心函数

| void StartBlock(uint64_t block_offset) | 初始化     |
| -------------------------------------- | ---------- |
| void AddKey(const Slice& key)          | 添加Key    |
| Slice Finish();                        | 完成       |
| void GenerateFilter()                  | 生成过滤器 |

```cpp
void FilterBlockBuilder::StartBlock(uint64_t block_offset) {
  uint64_t filter_index = (block_offset / kFilterBase);
  assert(filter_index >= filter_offsets_.size());
  while (filter_index > filter_offsets_.size()) {
    GenerateFilter();
  }
}

void FilterBlockBuilder::AddKey(const Slice& key) {
  Slice k = key;
  start_.push_back(keys_.size());
  keys_.append(k.data(), k.size());
}

Slice FilterBlockBuilder::Finish() {
  if (!start_.empty()) {
    GenerateFilter();
  }

  // Append array of per-filter offsets
  const uint32_t array_offset = result_.size();
  for (size_t i = 0; i < filter_offsets_.size(); i++) {
    PutFixed32(&result_, filter_offsets_[i]);
  }

  PutFixed32(&result_, array_offset);
  result_.push_back(kFilterBaseLg);  // Save encoding parameter in result
  return Slice(result_);
}

void FilterBlockBuilder::GenerateFilter() {
  const size_t num_keys = start_.size();
  if (num_keys == 0) {
    // Fast path if there are no keys for this filter
    filter_offsets_.push_back(result_.size());
    return;
  }

  // Make list of keys from flattened key structure
  start_.push_back(keys_.size());  // Simplify length computation
  tmp_keys_.resize(num_keys);
  for (size_t i = 0; i < num_keys; i++) {
    const char* base = keys_.data() + start_[i];
    size_t length = start_[i + 1] - start_[i];
    tmp_keys_[i] = Slice(base, length);
  }

  // Generate filter for current set of keys and append to result_.
  filter_offsets_.push_back(result_.size());
  policy_->CreateFilter(&tmp_keys_[0], static_cast<int>(num_keys), &result_);

  tmp_keys_.clear();
  keys_.clear();
  start_.clear();
}

```

前面提到每2k需要一个Filter,这反应在  `uint64_t filter_index = (block_offset / kFilterBase)`其中kFilterBase就是2048,计算得到当前数据的FilterIndex大于filter数量时那就创建新的Filter,在向SSTable中添加数据时会同步调用FilterBlockBuilder的Add方法来设置Filter,添加的Key会扁平化存储在keys中，同时在starts中存储索引信息。当一个SSTable写满后会调用Finish生成一个DataBlock,同时也会同步调用Filter的Finsh将布隆过滤器的数据写入FilterBlock。过滤器的创建在bloom.cc文件中，跳过不讲了。

### Block结构和FilterBlockReade结构

#### Block结构

SSTable中每个Block读取出来后通过Block结构来存储，而读取是通过Block::Iter迭代器实现的。

成员变量

|   **变量名**    |  **类型**   |                           **描述**                           |
| :-------------: | :---------: | :----------------------------------------------------------: |
|      data_      | const char* |          指向块数据的指针，存储实际内容（不可修改）          |
|      size_      |   size_t    |                   块数据的实际大小（字节）                   |
| restart_offset_ |  uint32_t   | 块内重启点（Restart Point）数组在 data_中的偏移量，用于快速定位 |
|     owned_      |    bool     | 标记 Block 是否拥有 data_ 的内存所有权，控制析构时是否释放资源 |

```c++
Block::Block(const BlockContents& contents)
    : data_(contents.data.data()),
      size_(contents.data.size()),
      owned_(contents.heap_allocated) {
  if (size_ < sizeof(uint32_t)) {
    size_ = 0;  // Error marker
  } else {
    size_t max_restarts_allowed = (size_ - sizeof(uint32_t)) / sizeof(uint32_t);
    if (NumRestarts() > max_restarts_allowed) {
      // The size is too small for NumRestarts()
      size_ = 0;
    } else {
      restart_offset_ = size_ - (1 + NumRestarts()) * sizeof(uint32_t);
    }
  }
}
```

先进行基本校验，DataBlock末尾必须存储一个重启点数据大小（uint32_t），再验证重启点数量合法性和计算重启点数组偏移量。

```cpp
// Helper routine: decode the next block entry starting at "p",
// storing the number of shared key bytes, non_shared key bytes,
// and the length of the value in "*shared", "*non_shared", and
// "*value_length", respectively.  Will not dereference past "limit".
//
// If any errors are detected, returns nullptr.  Otherwise, returns a
// pointer to the key delta (just past the three decoded values).
static inline const char* DecodeEntry(const char* p, const char* limit,
                                      uint32_t* shared, uint32_t* non_shared,
                                      uint32_t* value_length) {
  if (limit - p < 3) return nullptr;
  *shared = reinterpret_cast<const uint8_t*>(p)[0];
  *non_shared = reinterpret_cast<const uint8_t*>(p)[1];
  *value_length = reinterpret_cast<const uint8_t*>(p)[2];
  if ((*shared | *non_shared | *value_length) < 128) {
    // Fast path: all three values are encoded in one byte each
    p += 3;
  } else {
    if ((p = GetVarint32Ptr(p, limit, shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, non_shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, value_length)) == nullptr) return nullptr;
  }

  if (static_cast<uint32_t>(limit - p) < (*non_shared + *value_length)) {
    return nullptr;
  }
  return p;
}
```

从p位置开始解析entry的shared,non_shared,value_length等信息

Block通过迭代器来读取，在Block中进行查找时，主要通过Block::Iter的Seek方法完成

```cpp
  void Seek(const Slice& target) override {
    // Binary search in restart array to find the last restart point
    // with a key < target
    uint32_t left = 0;
    uint32_t right = num_restarts_ - 1;
    int current_key_compare = 0;

    if (Valid()) {
      // If we're already scanning, use the current position as a starting
      // point. This is beneficial if the key we're seeking to is ahead of the
      // current position.
      current_key_compare = Compare(key_, target);
      if (current_key_compare < 0) {
        // key_ is smaller than target
        left = restart_index_;
      } else if (current_key_compare > 0) {
        right = restart_index_;
      } else {
        // We're seeking to the key we're already at.
        return;
      }
    }

    while (left < right) {
      uint32_t mid = (left + right + 1) / 2;
      uint32_t region_offset = GetRestartPoint(mid);
      uint32_t shared, non_shared, value_length;
      const char* key_ptr =
          DecodeEntry(data_ + region_offset, data_ + restarts_, &shared,
                      &non_shared, &value_length);
      if (key_ptr == nullptr || (shared != 0)) {
        CorruptionError();
        return;
      }
      Slice mid_key(key_ptr, non_shared);
      if (Compare(mid_key, target) < 0) {
        // Key at "mid" is smaller than "target".  Therefore all
        // blocks before "mid" are uninteresting.
        left = mid;
      } else {
        // Key at "mid" is >= "target".  Therefore all blocks at or
        // after "mid" are uninteresting.
        right = mid - 1;
      }
    }

    // We might be able to use our current position within the restart block.
    // This is true if we determined the key we desire is in the current block
    // and is after than the current key.
    assert(current_key_compare == 0 || Valid());
    bool skip_seek = left == restart_index_ && current_key_compare < 0;
    if (!skip_seek) {
      SeekToRestartPoint(left);
    }
    // Linear search (within restart block) for first key >= target
    while (true) {
      if (!ParseNextKey()) {
        return;
      }
      if (Compare(key_, target) >= 0) {
        return;
      }
    }
  }
  bool ParseNextKey() {
    current_ = NextEntryOffset();
    const char* p = data_ + current_;
    const char* limit = data_ + restarts_;  // Restarts come right after data
    if (p >= limit) {
      // No more entries to return.  Mark as invalid.
      current_ = restarts_;
      restart_index_ = num_restarts_;
      return false;
    }

    // Decode next entry
    uint32_t shared, non_shared, value_length;
    p = DecodeEntry(p, limit, &shared, &non_shared, &value_length);
    if (p == nullptr || key_.size() < shared) {
      CorruptionError();
      return false;
    } else {
      key_.resize(shared);
      key_.append(p, non_shared);
      value_ = Slice(p + non_shared, value_length);
      while (restart_index_ + 1 < num_restarts_ &&
             GetRestartPoint(restart_index_ + 1) < current_) {
        ++restart_index_;
      }
      return true;
    }
  }
```

首先在重启点列表中进行二分查找，定位到比target小的最近的一个重启点，然后在该重启点开始顺序解析Entry（KV数据）进行查找比较，直到找到为止

#### FilterBlockReader结构

很简单，没什么好写的

## SSTable的写入和读取

### TableBuilder

```cpp
void TableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->num_entries > 0) {
    assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
  }

  if (r->pending_index_entry) {
    assert(r->data_block.empty());
    r->options.comparator->FindShortestSeparator(&r->last_key, key);
    std::string handle_encoding;
    r->pending_handle.EncodeTo(&handle_encoding);
    r->index_block.Add(r->last_key, Slice(handle_encoding));
    r->pending_index_entry = false;
  }

  if (r->filter_block != nullptr) {
    r->filter_block->AddKey(key);
  }

  r->last_key.assign(key.data(), key.size());
  r->num_entries++;
  r->data_block.Add(key, value);

  const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
  if (estimated_block_size >= r->options.block_size) {
    Flush();
  }
}
```

当调用SSTable的Add(k，v)方法添加一条KV数据时，首先会将该数据依次加入Data Block和Filter Block中，添加完成后再判定当前的Data Block大小是否已经大于设定的阈值了。如果大于阈值则会调用Flush()方法将当前的Data Block写入SSTable文件并清空(block->reset())Data Block，同时将pending_index_entry的值设为true。当下一条KV数据再进来时会命中该值为true的逻辑，然后往Index Block中追加一条索引信息，追加完成后再将其重新置回false

```cpp
Status TableBuilder::Finish() {
  Rep* r = rep_;
  Flush();
  assert(!r->closed);
  r->closed = true;

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Write filter block
  if (ok() && r->filter_block != nullptr) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != nullptr) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // Write index block
  if (ok()) {
    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write footer
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}

```

就是按SSTable的结构将每部分Block追加写入文件而已

### Table

读取靠Table的Iter完成

```cpp
Status Table::Open(const Options& options, RandomAccessFile* file,
                   uint64_t size, Table** table) {
  *table = nullptr;
  if (size < Footer::kEncodedLength) {
    return Status::Corruption("file is too short to be an sstable");
  }

  char footer_space[Footer::kEncodedLength];
  Slice footer_input;
  Status s = file->Read(size - Footer::kEncodedLength, Footer::kEncodedLength,
                        &footer_input, footer_space);
  if (!s.ok()) return s;

  Footer footer;
  s = footer.DecodeFrom(&footer_input);
  if (!s.ok()) return s;

  // Read the index block
  BlockContents index_block_contents;
  ReadOptions opt;
  if (options.paranoid_checks) {
    opt.verify_checksums = true;
  }
  s = ReadBlock(file, opt, footer.index_handle(), &index_block_contents);

  if (s.ok()) {
    // We've successfully read the footer and the index block: we're
    // ready to serve requests.
    Block* index_block = new Block(index_block_contents);
    Rep* rep = new Table::Rep;
    rep->options = options;
    rep->file = file;
    rep->metaindex_handle = footer.metaindex_handle();
    rep->index_block = index_block;
    rep->cache_id = (options.block_cache ? options.block_cache->NewId() : 0);
    rep->filter_data = nullptr;
    rep->filter = nullptr;
    *table = new Table(rep);
    (*table)->ReadMeta(footer);
  }

  return s;
}
```

首先读取Footer,根据Footer存储的Meta Index Block和Index Block的索引信息，依次调用ReadBlock读取数据，读取索引信息后进一步调用ReadFilter读取FilterBlock中布隆过滤器的数据，然后就可以处理查询请求了。在查询时SSTable对外通过TwoLevelIterator迭代器来查找。该迭代器创建时需要传递两个迭代器：一个是Index Block的迭代器，另一个是Data Block的迭代器。这也是TwoLevelIterator名称的由来。

## SSTable读取全过程

入口是Version::Get()

```cpp
Status Version::Get(const ReadOptions& options, const LookupKey& k,
                    std::string* value, GetStats* stats) {
  stats->seek_file = nullptr;
  stats->seek_file_level = -1;

  struct State {
    Saver saver;
    GetStats* stats;
    const ReadOptions* options;
    Slice ikey;
    FileMetaData* last_file_read;
    int last_file_read_level;

    VersionSet* vset;
    Status s;
    bool found;
    //从第level层的第f个文件开始判断是否匹配  
    static bool Match(void* arg, int level, FileMetaData* f) {
      State* state = reinterpret_cast<State*>(arg);

      if (state->stats->seek_file == nullptr &&
          state->last_file_read != nullptr) {
        // We have had more than one seek for this read.  Charge the 1st file.
        state->stats->seek_file = state->last_file_read;
        state->stats->seek_file_level = state->last_file_read_level;
      }

      state->last_file_read = f;
      state->last_file_read_level = level;
      //从SSTable的缓存中查找，其中Savlue是个函数  
      state->s = state->vset->table_cache_->Get(*state->options, f->number,
                                                f->file_size, state->ikey,
                                                &state->saver, SaveValue);
      if (!state->s.ok()) {
        state->found = true;
        return false;
      }
      switch (state->saver.state) {
        case kNotFound:
          return true;  // Keep searching in other files
        case kFound:
          state->found = true;
          return false;
        case kDeleted:
          return false;
        case kCorrupt:
          state->s =
              Status::Corruption("corrupted key for ", state->saver.user_key);
          state->found = true;
          return false;
      }

      // Not reached. Added to avoid false compilation warnings of
      // "control reaches end of non-void function".
      return false;
    }
  };

  State state;
  state.found = false;
  state.stats = stats;
  state.last_file_read = nullptr;
  state.last_file_read_level = -1;

  state.options = &options;
  state.ikey = k.internal_key();
  state.vset = vset_;

  state.saver.state = kNotFound;
  state.saver.ucmp = vset_->icmp_.user_comparator();
  state.saver.user_key = k.user_key();
  state.saver.value = value;
  //遍历所有层的SSTable
  ForEachOverlapping(state.saver.user_key, state.ikey, &state, &State::Match);

  return state.found ? state.s : Status::NotFound(Slice());
}
```

```cpp
void Version::ForEachOverlapping(Slice user_key, Slice internal_key, void* arg,
                                 bool (*func)(void*, int, FileMetaData*)) {
  const Comparator* ucmp = vset_->icmp_.user_comparator();

  // Search level-0 in order from newest to oldest.
  std::vector<FileMetaData*> tmp;
  tmp.reserve(files_[0].size());
  for (uint32_t i = 0; i < files_[0].size(); i++) {
    FileMetaData* f = files_[0][i];
    if (ucmp->Compare(user_key, f->smallest.user_key()) >= 0 &&
        ucmp->Compare(user_key, f->largest.user_key()) <= 0) {
      tmp.push_back(f);
    }
  }
  if (!tmp.empty()) {
    //从新到旧排序
    std::sort(tmp.begin(), tmp.end(), NewestFirst);
    for (uint32_t i = 0; i < tmp.size(); i++) {
      if (!(*func)(arg, 0, tmp[i])) {
        return;
      }
    }
  }

  // Search other levels.
  for (int level = 1; level < config::kNumLevels; level++) {
    size_t num_files = files_[level].size();
    if (num_files == 0) continue;

    // Binary search to find earliest index whose largest key >= internal_key.
    uint32_t index = FindFile(vset_->icmp_, files_[level], internal_key);
    if (index < num_files) {
      FileMetaData* f = files_[level][index];
      if (ucmp->Compare(user_key, f->smallest.user_key()) < 0) {
        // All of "f" is past any data for user_key
      } else {
        if (!(*func)(arg, level, f)) {
          return;
        }
      }
    }
  }
}
//在一个层的files中通过二分查找定位到某个SSTable文件
int FindFile(const InternalKeyComparator& icmp,
             const std::vector<FileMetaData*>& files, const Slice& key) {
  uint32_t left = 0;
  uint32_t right = files.size();
  while (left < right) {
    uint32_t mid = (left + right) / 2;
    const FileMetaData* f = files[mid];
    if (icmp.InternalKeyComparator::Compare(f->largest.Encode(), key) < 0) {
      // Key at "mid.largest" is < "target".  Therefore all
      // files at or before "mid" are uninteresting.
      left = mid + 1;
    } else {
      // Key at "mid.largest" is >= "target".  Therefore all files
      // after "mid" are uninteresting.
      right = mid;
    }
  }
  return right;
}
```

在上述查找过程中，首先调用ForEachOverlapping在所有层开始查找。具体过程是，首先在level 0层按照文件新旧的顺序逐个查找（因为level0层的SSTable之间的数据有可能相互重叠），只要找到就结束查找。当level 0层没有找到时，在剩下的层开始逐层查找。level 0层之外的其他层的多个SSTable中的数据是不重叠的，因此待查找的key只会命中其中一个SSTable文件。这也是FindFile中通过二分查找、利用每个SSTable文件保存的最大值来定位SSTable文件的逻辑。当找到该文件后，再在该文件中查找。单个SSTable的具体查找过程实际上是在Match方法中完成的

## Compact的实现分析

> [!NOTE]
>
> 累了，不搞了

