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

## 2. DoCompactionWork实现

### 2.1 核心流程

```cpp
// db/db_impl.cc
Status DBImpl::DoCompactionWork(CompactionState* compact) {
  const uint64_t start_micros = env_->NowMicros();
  int64_t imm_micros = 0;
  
  Log(options_.info_log, "Compacting %d@%d + %d@%d files",
      compact->compaction->num_input_files(0), compact->compaction->level(),
      compact->compaction->num_input_files(1),
      compact->compaction->level() + 1);
  
  assert(versions_->NumLevelFiles(compact->compaction->level()) > 0);
  assert(compact->builder == nullptr);
  assert(compact->outfile == nullptr);
  
  if (snapshots_.empty()) {
    compact->smallest_snapshot = versions_->LastSequence();
  } else {
    compact->smallest_snapshot = snapshots_.oldest()->sequence_number();
  }
  
  Iterator* input = versions_->MakeInputIterator(compact->compaction);
  
  mutex_.Unlock();
  input->SeekToFirst();
  Status status;
  ParsedInternalKey ikey;
  std::string current_user_key;
  bool has_current_user_key = false;
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;
  
  while (input->Valid() && !shutting_down_.load(std::memory_order_acquire)) {
    if (has_imm_.load(std::memory_order_relaxed)) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      if (imm_ != nullptr) {
        CompactMemTable();
        background_work_finished_signal_.SignalAll();
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }
    
    Slice key = input->key();
    if (compact->compaction->ShouldStopBefore(key) &&
        compact->builder != nullptr) {
      status = FinishCompactionOutputFile(compact, input);
      if (!status.ok()) {
        break;
      }
    }
    
    bool drop = false;
    if (!ParseInternalKey(key, &ikey)) {
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
    } else {
      if (!has_current_user_key ||
          user_comparator()->Compare(ikey.user_key, Slice(current_user_key)) !=
              0) {
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
        last_sequence_for_key = kMaxSequenceNumber;
      }
      
      if (last_sequence_for_key <= compact->smallest_snapshot) {
        drop = true;
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
        drop = true;
      }
      
      last_sequence_for_key = ikey.sequence;
    }
    
    if (!drop) {
      if (compact->builder == nullptr) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      compact->builder->Add(key, input->value());
      
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }
    
    input->Next();
  }
  
  if (status.ok() && shutting_down_.load(std::memory_order_acquire)) {
    status = Status::IOError("Deleting DB during compaction");
  }
  if (status.ok() && compact->builder != nullptr) {
    status = FinishCompactionOutputFile(compact, input);
  }
  if (status.ok()) {
    status = input->status();
  }
  delete input;
  input = nullptr;
  
  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros - imm_micros;
  for (int which = 0; which < 2; which++) {
    for (int i = 0; i < compact->compaction->num_input_files(which); i++) {
      stats.bytes_read += compact->compaction->input(which, i)->file_size;
    }
  }
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    stats.bytes_written += compact->outputs[i].file_size;
  }
  
  mutex_.Lock();
  stats_[compact->compaction->level() + 1].Add(stats);
  
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

### 2.2 去重和删除逻辑

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
