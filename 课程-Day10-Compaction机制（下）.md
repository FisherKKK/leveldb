# Day 10: Compaction机制（下）

## 学习目标
- 掌握PickCompaction算法
- 理解DoCompactionWork实现细节
- 学习Compaction优化技巧
- 分析写放大问题

## 1. PickCompaction算法

### 1.1 选择Compaction层级

```cpp
// db/version_set.cc
Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  int level;

  const bool size_compaction = (current_->compaction_score_ >= 1);
  const bool seek_compaction = (current_->file_to_compact_ != nullptr);
  
  if (size_compaction) {
    level = current_->compaction_level_;
    assert(level >= 0);
    assert(level + 1 < config::kNumLevels);
    c = new Compaction(options_, level);
    
    // 选择文件
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      if (compact_pointer_[level].empty() ||
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0) {
        c->inputs_[0].push_back(f);
        break;
      }
    }
    
    if (c->inputs_[0].empty()) {
      c->inputs_[0].push_back(current_->files_[level][0]);
    }
  } else if (seek_compaction) {
    level = current_->file_to_compact_level_;
    c = new Compaction(options_, level);
    c->inputs_[0].push_back(current_->file_to_compact_);
  } else {
    return nullptr;
  }
  
  c->input_version_ = current_;
  c->input_version_->Ref();
  
  if (level == 0) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[0], &smallest, &largest);
    current_->GetOverlappingInputs(0, &smallest, &largest, &c->inputs_[0]);
    assert(!c->inputs_[0].empty());
  }
  
  SetupOtherInputs(c);
  return c;
}
```

### 1.2 选择重叠文件

```cpp
void VersionSet::SetupOtherInputs(Compaction* c) {
  const int level = c->level();
  InternalKey smallest, largest;
  
  AddBoundaryInputs(icmp_, current_->files_[level], &c->inputs_[0]);
  GetRange(c->inputs_[0], &smallest, &largest);
  
  current_->GetOverlappingInputs(level + 1, &smallest, &largest,
                                  &c->inputs_[1]);
  AddBoundaryInputs(icmp_, current_->files_[level + 1], &c->inputs_[1]);
  
  InternalKey all_start, all_limit;
  GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
  
  if (!c->inputs_[1].empty()) {
    std::vector<FileMetaData*> expanded0;
    current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
    AddBoundaryInputs(icmp_, current_->files_[level], &expanded0);
    const int64_t inputs0_size = TotalFileSize(c->inputs_[0]);
    const int64_t inputs1_size = TotalFileSize(c->inputs_[1]);
    const int64_t expanded0_size = TotalFileSize(expanded0);
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
            ExpandedCompactionByteSizeLimit(options_)) {
      InternalKey new_start, new_limit;
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      current_->GetOverlappingInputs(level + 1, &new_start, &new_limit,
                                      &expanded1);
      AddBoundaryInputs(icmp_, current_->files_[level + 1], &expanded1);
      if (expanded1.size() == c->inputs_[1].size()) {
        Log(options_->info_log,
            "Expanding@%d %d+%d (%ld+%ld bytes) to %d+%d (%ld+%ld bytes)\n",
            level, int(c->inputs_[0].size()), int(c->inputs_[1].size()),
            long(inputs0_size), long(inputs1_size), int(expanded0.size()),
            int(expanded1.size()), long(expanded0_size), long(inputs1_size));
        smallest = new_start;
        largest = new_limit;
        c->inputs_[0] = expanded0;
        c->inputs_[1] = expanded1;
        GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
      }
    }
  }
  
  if (level + 2 < config::kNumLevels) {
    current_->GetOverlappingInputs(level + 2, &all_start, &all_limit,
                                    &c->grandparents_);
  }
  
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
}
```

## 2. DoCompactionWork深度分析

### 2.1 逐行代码详解

```cpp
// db/db_impl.cc, lines 897-1056
Status DBImpl::DoCompactionWork(CompactionState* compact) {
  // ========== 阶段1：初始化 ==========
  const uint64_t start_micros = env_->NowMicros();
  int64_t imm_micros = 0;  // 记录处理Immutable MemTable的时间

  // 记录日志
  Log(options_.info_log, "Compacting %d@%d + %d@%d files",
      compact->compaction->num_input_files(0),      // Level-N文件数
      compact->compaction->level(),
      compact->compaction->num_input_files(1),      // Level-N+1文件数
      compact->compaction->level() + 1);

  // 前置条件检查
  assert(versions_->NumLevelFiles(compact->compaction->level()) > 0);
  assert(compact->builder == nullptr);
  assert(compact->outfile == nullptr);

  // ========== 阶段2：确定快照版本 ==========
  // smallest_snapshot: 最老快照的序列号
  // 任何序列号 > smallest_snapshot的数据都可能有快照引用
  if (snapshots_.empty()) {
    // 没有快照，使用最新序列号
    compact->smallest_snapshot = versions_->LastSequence();
  } else {
    // 有快照，使用最老的快照序列号
    compact->smallest_snapshot = snapshots_.oldest()->sequence_number();
  }

  // ========== 阶段3：创建合并迭代器 ==========
  // MergingIterator可以同时遍历多个有序数据源
  Iterator* input = versions_->MakeInputIterator(compact->compaction);

  // ========== 阶段4：主合并循环 ==========
  mutex_.Unlock();  // ⚠️ 释放锁，允许并发
  input->SeekToFirst();

  Status status;
  ParsedInternalKey ikey;
  std::string current_user_key;           // 当前用户键（用于去重）
  bool has_current_user_key = false;
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;

  // 核心循环：遍历所有键值对
  while (input->Valid() && !shutting_down_.load(std::memory_order_acquire)) {
    // ----- 子任务1：处理Immutable MemTable -----
    if (has_imm_.load(std::memory_order_relaxed)) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      if (imm_ != nullptr) {
        CompactMemTable();  // 优先刷盘
        background_work_finished_signal_.SignalAll();
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }

    // ----- 子任务2：检查是否需要切割输出文件 -----
    Slice key = input->key();
    if (compact->compaction->ShouldStopBefore(key) &&
        compact->builder != nullptr) {
      // 当前文件与Level+2重叠太多，关闭当前文件
      status = FinishCompactionOutputFile(compact, input);
      if (!status.ok()) {
        break;
      }
    }

    // ----- 子任务3：解析InternalKey -----
    bool drop = false;
    if (!ParseInternalKey(key, &ikey)) {
      // 解析失败，保留（但不清空当前键）
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
    } else {
      // 检查是否是新的user_key
      if (!has_current_user_key ||
          user_comparator()->Compare(ikey.user_key, Slice(current_user_key)) != 0) {
        // 新的user_key
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
        last_sequence_for_key = kMaxSequenceNumber;
      }

      // ----- 子任务4：去重逻辑 -----
      if (last_sequence_for_key <= compact->smallest_snapshot) {
        // 规则A: 被更新的版本覆盖，丢弃
        drop = true;
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
        // 规则B: 删除标记且更低层没有该键，可以丢弃
        drop = true;
      }

      last_sequence_for_key = ikey.sequence;
    }

    // ----- 子任务5：写入输出文件 -----
    if (!drop) {
      // 如果没有输出文件，创建一个
      if (compact->builder == nullptr) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }

      // 记录键范围
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);

      // 添加到TableBuilder
      compact->builder->Add(key, input->value());

      // 检查文件大小，如果超过阈值，关闭当前文件
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }

    // 移动到下一个键值对
    input->Next();
  }

  // ========== 阶段5：完成处理 ==========
  if (status.ok() && shutting_down_.load(std::memory_order_acquire)) {
    status = Status::IOError("Deleting DB during compaction");
  }

  // 关闭最后一个输出文件
  if (status.ok() && compact->builder != nullptr) {
    status = FinishCompactionOutputFile(compact, input);
  }

  // 检查迭代器状态
  if (status.ok()) {
    status = input->status();
  }

  // 清理迭代器
  delete input;
  input = nullptr;

  // ========== 阶段6：统计信息 ==========
  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros - imm_micros;

  // 统计读取的字节数
  for (int which = 0; which < 2; which++) {
    for (int i = 0; i < compact->compaction->num_input_files(which); i++) {
      stats.bytes_read += compact->compaction->input(which, i)->file_size;
    }
  }

  // 统计写入的字节数
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    stats.bytes_written += compact->outputs[i].file_size;
  }

  // 重新获取锁
  mutex_.Lock();
  stats_[compact->compaction->level() + 1].Add(stats);

  // ========== 阶段7：应用Compaction结果 ==========
  if (status.ok()) {
    status = InstallCompactionResults(compact);
  }

  if (!status.ok()) {
    RecordBackgroundError(status);
  }

  VersionSet::LevelSummaryStorage tmp;
  Log(options_.info_log, "compacted to: %s", versions_->LevelSummary(&tmp));
  return status;
}
```

### 2.2 数据流程图

```
DoCompactionWork数据流：

输入：
Level-N:
  ┌─────────────────────────────────────┐
  │ file001.ldb: [a, m]                 │
  │ file002.ldb: [n, z]                 │
  └─────────────────────────────────────┘

Level-N+1:
  ┌─────────────────────────────────────┐
  │ file100.ldb: [f, p]  ← 重叠         │
  │ file101.ldb: [q, w]                 │
  └─────────────────────────────────────┘
           │
           ▼ MergingIterator
           └─> 合并两个层级的有序数据
               输出：[a, z] 全部键

处理循环：
  a@100 (value1)  ← 保留（第一次见到key 'a'）
  b@95  (value2)  ← 保留
  ...
  f@80  (value3)  ← 保留
  f@50  (old_f)   ← 丢弃（被f@80覆盖）
  g@75  (value4)  ← 保留
  ...
  m@70  (DEL)     ← 保留（删除标记）
  m@40  (old_m)   ← 丢弃（被m@70覆盖）
  ...

输出：
Level-N+1:
  ┌─────────────────────────────────────┐
  │ file200.ldb: [a, z]                 │
  │   ├─ a@100, b@95, ..., f@80        │
  │   ├─ g@75, ..., m@70 (DEL)         │
  │   └─ ... (去重后)                   │
  └─────────────────────────────────────┘
```

### 2.3 去重逻辑深度解析

#### 2.3.1 去重场景分析

```
场景1：简单去重（同一user_key的多个版本）

Input:
  key="user:001", seq=100, type=Put,    value="Alice"
  key="user:001", seq=95,  type=Put,    value="Alice_95"
  key="user:001", seq=90,  type=Put,    value="Alice_90"
  key="user:001", seq=85,  type=Delete

Processing:
  seq=100: current_user_key="", 第一次见到user:001
          → has_current_user_key=true
          → last_sequence=100
          → drop=false (保留，输出)

  seq=95:  current_user_key="user:001", 同一个key
          → last_sequence_for_key=100 > smallest_snapshot
          → drop=false (保留，输出)

  seq=90:  current_user_key="user:001", 同一个key
          → last_sequence_for_key=95 > smallest_snapshot
          → drop=false (保留，输出)

  seq=85:  current_user_key="user:001", 同一个key
          → last_sequence_for_key=90 > smallest_snapshot
          → drop=false (保留，输出)

Output:
  4个版本全部保留（因为都有快照可能引用）

实际上，通常场景：
  smallest_snapshot = 50

  seq=100: last_sequence=MAX, drop=false ✓
  seq=95:  last_sequence=100 > 50, drop=false ✓
  seq=90:  last_sequence=95 > 50, drop=false ✓
  seq=85:  last_sequence=90 > 50, drop=false ✓

如果有快照=97：
  smallest_snapshot = 97

  seq=100: last_sequence=MAX, drop=false ✓
  seq=95:  last_sequence=100 > 97, drop=false ✓
  seq=90:  last_sequence=95 <= 97, drop=true ✗
  seq=85:  last_sequence=90 <= 97, drop=true ✗

Output: 只有seq=100和seq=95被保留
```

#### 2.3.2 去伪代码逻辑

```cpp
// 去重的核心逻辑（简化版）
void ProcessKey(const InternalKey& ikey) {
  // 1. 检查是否是新的user_key
  if (ikey.user_key != current_user_key) {
    // 新的user_key
    current_user_key = ikey.user_key;
    last_sequence = MAX;  // 重置
  }

  // 2. 去重检查
  if (last_sequence <= smallest_snapshot) {
    // 上一个版本的序列号 <= 最老快照
    // 意味着所有快照都只看到更新的版本
    // 当前版本可以被丢弃
    drop = true;
  } else {
    // 可能有快照看到这个版本，保留
    drop = false;
  }

  // 3. 更新last_sequence
  last_sequence = ikey.sequence;

  // 4. 写入或丢弃
  if (!drop) {
    builder->Add(ikey, value);
  }
}
```

**为什么这样设计？**

```
快照的语义：快照只能看到sequence <= snapshot_seq的数据

假设：
- 当前时间：seq=100
- 快照1：seq=97
- 快照2：seq=80

数据：
  user:001 @ 100 (Put)
  user:001 @ 95  (Put)
  user:001 @ 90  (Put)
  user:001 @ 85  (Delete)

smallest_snapshot = min(97, 80) = 80

去重逻辑：
  100 > 80: 保留（所有快照都可能需要）
  95 > 80:  保留
  90 > 80:  保留
  85 > 80:  保留

实际上：
- 快照1(seq=97)看到：100 ✓, 95 ✓（>97的不存在）
- 快照2(seq=80)看到：100 ✓, 95 ✓, 90 ✓, 85 ✓

如果smallest_snapshot = 97:
  100 > 97: 保留
  95 > 97: 保留
  90 <= 97: 丢弃！（快照1不需要seq<=90的版本）
  85 <= 97: 丢弃！

- 快照1(seq=97)看到：100 ✓, 95 ✓（正确）
- 理论上seq<=90的版本对快照1都是不可见的
```

### 2.4 删除标记清理

#### 2.4.1 清理条件

```cpp
// 删除标记清理逻辑
if (ikey.type == kTypeDeletion &&
    ikey.sequence <= compact->smallest_snapshot &&
    compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
  drop = true;  // 可以删除删除标记
}
```

**三个条件：**

1. **ikey.type == kTypeDeletion**
   - 必须是删除标记

2. **ikey.sequence <= smallest_snapshot**
   - 没有快照引用这个删除标记

3. **IsBaseLevelForKey(user_key) == true**
   - 更高层（Level-N+2及以上）没有这个键
   - 如果有，删除标记必须保留，用于删除更高层的旧版本

#### 2.4.2 清理场景

```
场景1：可以清理

Level-N:
  file001.ldb: [a, z]
    └─ user:001 @ 50 (Delete)

Level-N+1:
  (空)

Level-N+2:
  (空)

IsBaseLevelForKey("user:001") = true ✓
sequence=50 <= smallest_snapshot=80 ✓

结果：删除标记可以清理

---

场景2：不能清理

Level-N:
  file001.ldb: [a, z]
    └─ user:001 @ 50 (Delete)

Level-N+1:
  (空)

Level-N+2:
  file200.ldb: [m, w]
    └─ user:001 @ 30 (Put, "old_value")

IsBaseLevelForKey("user:001") = false ✗
（Level-N+2有user:001）

结果：删除标记不能清理
原因：需要用删除标记删除Level-N+2的旧版本
```

#### 2.4.3 IsBaseLevelForKey实现

```cpp
// db/version_set.cc
bool Compaction::IsBaseLevelForKey(const Slice& user_key) {
  // 检查Level+2及以上是否有user_key
  for (int lvl = level_ + 2; lvl < config::kNumLevels; lvl++) {
    const std::vector<FileMetaData*>& files = input_version_->files_[lvl];

    // 在每一层中进行二分查找
    while (files.size() > 0) {
      // 找到第一个 >= user_key的文件
      // ...
      if (user_key在文件的范围内) {
        return false;  // 找到了，不能清理
      }
    }
  }

  return true;  // 更高层没有，可以清理
}
```

### 2.5 ShouldStopBefore机制

#### 2.5.1 为什么需要切割文件？

```
问题：输出文件过大导致后续Compaction代价高

Level-N: 8MB
Level-N+1: 50MB
  ↓ Compaction
Level-N+1: 50MB (原本) + 40MB (新输出) = 90MB

如果Level-N+2重叠很多：
  Level-N+2: 500MB
  下次Compaction: 90MB + 500MB = 590MB

代价太高！

解决方案：ShouldStopBefore
- 监控与Level-N+2的重叠字节数
- 超过阈值（20MB）就切割输出文件
- 限制单个输出文件与Level-N+2的重叠
```

#### 2.5.2 实现细节

```cpp
// db/db_impl.cc
bool Compaction::ShouldStopBefore(const Slice& internal_key) {
  const VersionSet* vset = input_version_->vset_;
  const InternalKeyComparator* icmp = &vset->icmp_;

  // 1. 跳过已经检查过的grandparent文件
  while (grandparent_index_ < grandparents_.size() &&
         icmp->Compare(internal_key,
                       grandparents_[grandparent_index_]->largest.Encode()) > 0) {
    if (seen_key_) {
      overlapped_bytes_ += grandparents_[grandparent_index_]->file_size;
    }
    grandparent_index_++;
  }

  seen_key_ = true;

  // 2. 检查重叠是否超过阈值
  if (overlapped_bytes_ > MaxGrandParentOverlapBytes(vset->options_)) {
    overlapped_bytes_ = 0;
    return true;  // 需要切割
  } else {
    return false;  // 继续追加到当前文件
  }
}
```

**示例：**

```
Level-N:
  file001.ldb: [a, z] (2MB)

Level-N+1:
  (空)

Level-N+2:
  file100.ldb: [a, f]  (2MB)
  file101.ldb: [g, m]  (2MB)
  file102.ldb: [n, s]  (2MB)
  file103.ldb: [t, z]  (2MB)

Compaction流程：
  keys a-f:
    overlapped_bytes += file100.size() = 2MB
    overlapped_bytes += file101.size() = 4MB
    继续当前输出文件...

  keys g-m:
    overlapped_bytes += file101.size() = 6MB
    overlapped_bytes += file102.size() = 8MB
    继续当前输出文件...

  keys n-s:
    overlapped_bytes += file102.size() = 10MB
    overlapped_bytes += file103.size() = 12MB
    继续当前输出文件...

  keys t-z:
    overlapped_bytes += file103.size() = 14MB
    继续当前输出文件...

假设MaxGrandParentOverlapBytes = 10MB:
  keys a-f:  4MB < 10MB, 继续
  keys g-m:  8MB < 10MB, 继续
  keys n-s:  12MB > 10MB, 切割！
  keys t-z:  重置overlapped_bytes=0, 开始新文件

结果：
  输出文件1: [a, m] (与Level-N+2重叠8MB)
  输出文件2: [n, z] (与Level-N+2重叠6MB)

下次Compaction代价降低：
  文件1: 2MB + 8MB = 10MB
  文件2: 2MB + 6MB = 8MB
```

### 2.6 InstallCompactionResults

```cpp
// db/db_impl.cc
Status DBImpl::InstallCompactionResults(CompactionState* compact) {
  mutex_.AssertHeld();

  // ========== 步骤1：添加输出文件 ==========
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    const CompactionState::Output& out = compact->outputs[i];
    compact->compaction->edit()->AddFile(
        compact->compaction->level() + 1,
        out.number, out.file_size,
        out.smallest, out.largest);
  }

  // ========== 步骤2：删除输入文件 ==========
  for (int which = 0; which < 2; which++) {
    for (int i = 0; i < compact->compaction->num_input_files(which); i++) {
      compact->compaction->edit()->DeleteFile(
          compact->compaction->level() + which,
          compact->compaction->input(which, i)->number);
    }
  }

  // ========== 步骤3：应用版本变更 ==========
  Status s = versions_->LogAndApply(compact->compaction->edit(), &mutex_);

  // ========== 步骤4：删除输入文件（磁盘） ==========
  if (s.ok()) {
    for (int which = 0; which < 2; which++) {
      for (int i = 0; i < compact->compaction->num_input_files(which); i++) {
        FileMetaData* f = compact->compaction->input(which, i);
        if (f->refs > 0) {
          // 文件还有引用，暂时不删除
        } else {
          // 删除磁盘文件
          env_->DeleteFile(TableFileName(dbname_, f->number));
        }
      }
    }
  }

  return s;
}
```

**原子性保证：**

```
错误情况：
  1. LogAndApply失败（MANIFEST写入失败）
     → 输出文件被删除
     → 输入文件保留
     → 数据库状态不变

  2. 删除文件失败（磁盘错误）
     → 版本已切换
     → 输入文件已标记为删除
     → 下次RemoveObsoleteFiles重试

正确情况：
  1. 输出文件写入完成
  2. MANIFEST写入完成
  3. Version切换完成
  4. 输入文件标记为删除
  5. 后台清理线程删除输入文件
```

### 2.7 去重和删除逻辑总结

核心逻辑：遍历排序后的键，根据规则决定保留或丢弃

保留规则：
1. 同一user_key的第一个版本：保留
2. sequence > smallest_snapshot：保留（可能被快照引用）
3. 其他版本：丢弃

删除标记清理规则：
1. sequence <= smallest_snapshot：可以考虑清理
2. IsBaseLevelForKey = true：更高层没有该键，可以清理
3. 否则：保留删除标记

## 3. Compaction优化

### 3.1 TrivialMove优化

```cpp
bool Compaction::IsTrivialMove() const {
  const VersionSet* vset = input_version_->vset_;
  return (num_input_files(0) == 1 && num_input_files(1) == 0 &&
          TotalFileSize(grandparents_) <=
              MaxGrandParentOverlapBytes(vset->options_));
}
```

条件：
- Level-N只有1个文件
- Level-N+1没有重叠文件
- Level-N+2重叠 < 20MB

操作：直接移动文件到Level-N+1，无需合并
收益：避免读写I/O，极大提升性能

### 3.2 GrandParent限制

避免Level-N+1文件过大导致后续Compaction代价高

```cpp
bool Compaction::ShouldStopBefore(const Slice& internal_key) {
  const VersionSet* vset = input_version_->vset_;
  const InternalKeyComparator* icmp = &vset->icmp_;
  while (grandparent_index_ < grandparents_.size() &&
         icmp->Compare(internal_key,
                       grandparents_[grandparent_index_]->largest.Encode()) >
             0) {
    if (seen_key_) {
      overlapped_bytes_ += grandparents_[grandparent_index_]->file_size;
    }
    grandparent_index_++;
  }
  seen_key_ = true;
  
  if (overlapped_bytes_ > MaxGrandParentOverlapBytes(vset->options_)) {
    overlapped_bytes_ = 0;
    return true;
  } else {
    return false;
  }
}
```

## 4. 写放大分析

### 4.1 写放大计算

```
写放大 = 总写入字节数 / 用户写入字节数

示例：写入100MB数据
- 用户写入：100MB
- WAL写入：100MB
- MemTable刷盘：100MB (Level-0)
- Level-0 → Level-1：200MB (100MB输入 + 100MB Level-1)
- Level-1 → Level-2：1000MB
- Level-2 → Level-3：10GB

总写入：~11.3GB
写放大：113x

实际LevelDB：
- 小数据集：5-10x
- 大数据集：10-30x
- 取决于：层数、Compaction策略
```

### 4.2 降低写放大

优化策略：
1. 增大write_buffer_size：减少L0文件数
2. 增大max_file_size：减少文件数量
3. 调整扩大系数：权衡读写放大
4. 使用Tiered Compaction：减少重写

## 5. 性能分析

```
Compaction性能数据：

Minor Compaction (MemTable → L0):
- 输入：4MB
- 输出：2MB
- 时间：100ms
- 写放大：1x

Major Compaction (L0 → L1):
- 输入：8MB (L0) + 10MB (L1) = 18MB
- 输出：15MB
- 时间：500ms
- 写放大：1.875x

Major Compaction (L1 → L2):
- 输入：10MB (L1) + 100MB (L2) = 110MB
- 输出：100MB
- 时间：3秒
- 写放大：10x
```

## 总结

今天我们学习了：
1. ✅ **PickCompaction**：选择文件、扩展边界
2. ✅ **DoCompactionWork**：合并、去重、删除
3. ✅ **优化技巧**：TrivialMove、GrandParent限制
4. ✅ **写放大分析**：计算和优化

**关键要点：**
- Compaction是读写性能的权衡
- 去重和删除清理节省空间
- TrivialMove大幅减少I/O
- 写放大是LSM-Tree固有问题

**思考题：**
1. 为什么需要GrandParent限制？
2. 如何在不影响正确性的前提下清理删除标记？
3. 写放大能降到多低？
4. Compaction能否并行执行？

**明天预告：Day 11 - Version与VersionSet版本管理**
我们将学习LevelDB如何管理数据库状态的多个版本。
