@[toc]
## 1、ElasticSearch和Lucene
### 1.1、elasticsearch
Elasticsearch 是一个基于 Apache Lucene的开源搜索引擎，它在Lucene的基础上还提供了：
- 分布式的准实时文件存储
- 分布式的搜索引擎
- 可以扩展到上百台服务器，处理 PB 级结构化或非结构化数据
类似ES的基于Lucene实现的搜索引擎还有Solr
### 1.2、Lucene
Lucene 是一个基于[信息检索(Information Retrieval, IR)](https://en.wikipedia.org/wiki/Information_retrieval)领域的一个最初由[Doug Cuting](https://en.wikipedia.org/wiki/Doug_Cutting)(发起开发Lucene，Nutch，Hadoop等开源项目)开发的搜索引擎库。Lucene仅是一个基于本地的，搜索的类库，本身并不具有分布式的特性，并不算是完备的搜索软件产品。在基于Lucene库之上衍生了一系列著名的搜索引擎产品如Elasticsearch，Solr, Nutch等
## 2、ElasticSearch和Lucene数据模型关系
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200607182349433.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
### ElasticSearch Index和Lucene Index关系
- ES集群有多个 Node （节点）组成，每个节点即 ES 的实例；
- 对于一个ES索引来说，每个节点上会有多个 shard （分片）， 如上图shard_P1、shard_P2是主分片 shard_R1、shard_R2是与主分片一一对应的副本分片 
- 每个分片上对应着就是一个 Lucene Index（底层索引文件） 
- Lucene Index 由多个 Segment （段文件，也就是倒排索引）组成。每个段文件中存储着document。
- 每个document中根据schema含有多个field
- 每个field根据数据的实际情况可能有多个取值，被称之为term
## 3、Lucene Index的数据模型
### 3.1、Segment
Lucene index中可能包含多个segment，每一个segment之间都是相互独立的。在搜索索引的时候，lucene将会依次搜索该索引的所有segment。
当一个lucene writer（比如IndexWriter）被【打开（open）】，lucene将会创建一个新的segment，当writer【提交更改（commit）】或【关闭（close）】的时候，这个segment就固定下来了。这样做的好处是对于任何已经创建的segment来说，他们都是不可更改的，在对索引进行添加和更新的时候，这些新的数据将会写入到新的segment中，已经存在的segment是不会变化的。
删除一个segment中的文档，只是加入一个删除标记，实际上，该文档的数据仍然存在与文件中。更新一个文档，只是将上一个版本的文档标记为删除，并写入一个新的版本的文档。换句话说，lucene中的文档更新并非是真正意义上的更新。通过尽可能不去修改已经存在的数据，lucene可以尽可能地避免出现因为频繁更新文档中的数据而导致索引被破坏的情况。但是这种设计也存在缺点，那就是随着时间的推移，一个index会包含越来越多的segment，从而大大影响搜索时的效率。
为了解决这个问题，lucene会在一定情况下（MergePolicy），合并一部分segment来优化index的存储结构。这种合并操作被成为Merge。 
### 3.2、Document
document是lucene index中数据的最小单位，即“一条数据”，类比于关系型数据库，一个document可以包含多个字段，在lucene中，他们被称作field。和关系型数据库不同的是，lucene中，document的fields并非完全固定，每个document也不一定要完全一致。在lucene中，对于一个document，它事实上实现了`Iterable<? extends IndexableField>`。其中IndexableField，顾名思义即可以被索引的field。
### 3.3、Field
field代表document中的一个字段。事实上Field在lucene中是实现了IndexableField的类，它还有多种子类，用于支持各种类型的数据格式。
- TextField: Reader or String indexed for full-text search
- StringField: String indexed verbatim as a single token
- IntPoint: int indexed for exact/range queries.
- LongPoint: long indexed for exact/range queries.
- FloatPoint: float indexed for exact/range queries.
- DoublePoint: double indexed for exact/range queries.
- SortedDocValuesField: byte[] indexed column-wise for sorting/faceting
- SortedSetDocValuesField: SortedSet<byte[]> indexed column-wise for sorting/faceting
- NumericDocValuesField: long indexed column-wise for sorting/faceting
- SortedNumericDocValuesField: SortedSet<long> indexed column-wise for sorting/faceting
- StoredField: Stored-only value for retrieving in summary results
这些子类的名称完全揭示了他们的用途
### 3.4、Term
一个[Term](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/Term.html)代表一段文本中的一个单词。Term是搜索的最小单位。它包含两个元素，它所代表的[单词的字符串](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/Term.html#text--)以及它[所属field名称](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/Term.html#field--)。值得注意的是Term不一定只代表一段文本中的一个单词，也可以是日期、email地址、urls等等
### index的Commit Point与时间的相关性
仅在Writer提交（commit）了之后，对索引的新写入或修改才会可见，当完成了segment文件的写入时的时间点即为一个index commit。
[Index commit point](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexCommit.html)都有一个与之关联的唯一segment文件。随着时间的推移，越后期的与index commit point关联的segment文件将具有较大的N。
## 4、Lucene写入流程分析
### 4.1、整体写入流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200607200131405.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
### 4.2、IndexWriter类
建议对照阅读：[https://github.com/apache/lucene-solr/blob/master/lucene/core/src/java/org/apache/lucene/index/IndexWriter.java](https://github.com/apache/lucene-solr/blob/master/lucene/core/src/java/org/apache/lucene/index/IndexWriter.java)
[IndexWriter](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html)是lucene中最为重要的类之一，它能新建和修改已经存在的索引。
在调用[IndexWriterConfig.setOpenMode(OpenMode) ](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriterConfig.html#setOpenMode-org.apache.lucene.index.IndexWriterConfig.OpenMode-)的时候可以指定[IndexWriterConfig.OpenMode](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriterConfig.OpenMode.html)中的选项，这样就可以知道是要新建立索引还是要写入已经存在的索引。可选的选项如下：
- [IndexWriterConfig.OpenMode.CREATE](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriterConfig.OpenMode.html#CREATE)
- [IndexWriterConfig.OpenMode.APPEND](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriterConfig.OpenMode.html#APPEND)
- [IndexWriterConfig.OpenMode.CREATE_OR_APPEND](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriterConfig.OpenMode.html#CREATE_OR_APPEND)

IndexWriter提供的修改方法有如下：
- [addDocument](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#addDocument-java.lang.Iterable-)（添加一个document）
- [deleteDocuments(Term...)](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#deleteDocuments-org.apache.lucene.index.Term...-)（删除一些documents）
- [deleteDocuments(Query)](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#deleteDocuments-org.apache.lucene.search.Query...-)（使用查询来删除一些documents）
- [updateDocument](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#updateDocument-org.apache.lucene.index.Term-java.lang.Iterable-)（其实就是先删除掉原先的老document，然后再写入新的document）

在完成修改后必须调用[close](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#close--)，如果在修改操作中途就想让现在当前的修改可见，则可以调用[commit()](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#commit--)
[IndexWriter](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html)对索引的修改将会被缓存到内存中，然后定量地周期性地flush到[Directory](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/store/Directory.html)上。
可通过下面的设置来调整flush策略：
[IndexWriterConfig.setRAMBufferSizeMB(double)](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriterConfig.html#setRAMBufferSizeMB-double-)
[IndexWriterConfig.setMaxBufferedDocs(int)](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriterConfig.html#setMaxBufferedDocs-int-)
[IndexWriterConfig.DEFAULT_RAM_BUFFER_SIZE_MB ](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriterConfig.html#DEFAULT_RAM_BUFFER_SIZE_MB)
另外删除操作不会触发flush。
flush只是让修改从IndexWriter的缓存中转移到了index上，但是这些修改在调用上文中提到的[commit()](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#commit--)或[close()](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#close--)之前，仍然是不可见的。
根据[MergeScheduler](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/MergeScheduler.html)的[策略](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/MergePolicy.html)，flush可能会触发segment merge。
打开一个IndexWriter会在索引所在的directory上创建一个文件锁，如果此时再对同一个directory打开另一个IndexWriter，就会收到[LockObtainFailedException](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/store/LockObtainFailedException.html)，对于es的运维人员可能会在日志中看到。
#### 4.2.1、AddDocument
```java
public long addDocument(Iterable<? extends IndexableField> doc)
                 throws IOException
```
ES使用IndexWriter的addDocument/addDocuments来为lucene index添加一个document。当这个方法被调用后，要被写入的document（s）被加入到IndexWriter的缓存中。根据参数的设置，可能会在document（s）加入缓存之后触发merge/flush。

```java
private long updateDocuments(final DocumentsWriterDeleteQueue.Node<?> delNode, Iterable<? extends Iterable<? extends IndexableField>> docs) throws IOException {
    ensureOpen();
    boolean success = false;
    try {
    //使用在构建时创建的docWriter进行update
      final long seqNo = maybeProcessEvents(docWriter.updateDocuments(docs, analyzer, delNode));
      success = true;
      return seqNo;
    } catch (VirtualMachineError tragedy) {
      tragicEvent(tragedy, "updateDocuments");
      throw tragedy;
    } finally {
      if (success == false) {
        if (infoStream.isEnabled("IW")) {
          infoStream.message("IW", "hit exception updating document");
        }
        maybeCloseOnTragicEvent();
      }
    }
  }
```
其DW中的UpdateDocuments如下：

```java
long updateDocuments(final Iterable<? extends Iterable<? extends IndexableField>> docs, final Analyzer analyzer,
                       final DocumentsWriterDeleteQueue.Node<?> delNode) throws IOException {
    boolean hasEvents = preUpdate();

    final DocumentsWriterPerThread dwpt = flushControl.obtainAndLock();
    final DocumentsWriterPerThread flushingDWPT;
    long seqNo;

    try {
      // This must happen after we've pulled the DWPT because IW.close
      // waits for all DWPT to be released:
      ensureOpen();
      final int dwptNumDocs = dwpt.getNumDocsInRAM();
      try {
        //dwpt —— document writer per thread
        seqNo = dwpt.updateDocuments(docs, analyzer, delNode, flushNotifications);
      } finally {
        if (dwpt.isAborted()) {
          flushControl.doOnAbort(dwpt);
        }
        // We don't know how many documents were actually
        // counted as indexed, so we must subtract here to
        // accumulate our separate counter:
        numDocsInRAM.addAndGet(dwpt.getNumDocsInRAM() - dwptNumDocs);
      }
      final boolean isUpdate = delNode != null && delNode.isDelete();
      flushingDWPT = flushControl.doAfterDocument(dwpt, isUpdate);
    } finally {
      if (dwpt.isFlushPending() || dwpt.isAborted()) {
        dwpt.unlock();
      } else {
        perThreadPool.marksAsFreeAndUnlock(dwpt);
      }
      assert dwpt.isHeldByCurrentThread() == false : "we didn't release the dwpt even on abort";
    }

    if (postUpdate(flushingDWPT, hasEvents)) {
      seqNo = -seqNo;
    }
    return seqNo;
  }
```
dwpt.updateDocuments请参考[此处](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/DocumentsWriterPerThread.java#L239)，总的来讲，dwpt通过调用DocConsumer中的processDocument()来对更改进行缓存，默认的实现将修改缓存在hash表中，被成为termsHash，详情参考[此处](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/DefaultIndexingChain.java#L384)
##### DocumentWriterPerThread
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020060720500913.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDI0MDM3MA==,size_16,color_FFFFFF,t_70)
#### 4.2.2、Commit
只要调用了commit()，目前缓存在IndexWriter中的改动会全部可见。然后用户可以直接利用该writer去进行其它的修改而无需关闭它。commit时将会按照policy来进行flush和merge。

```java
@Override
public final long commit() throws IOException {
    ensureOpen();
    return commitInternal(config.getMergePolicy());
}

private final long commitInternal(MergePolicy mergePolicy) throws IOException {

    if (infoStream.isEnabled("IW")) {
      infoStream.message("IW", "commit: start");
    }

    long seqNo;

    //这里有一个commitLock
    synchronized(commitLock) {
      ensureOpen(false);

      if (infoStream.isEnabled("IW")) {
        infoStream.message("IW", "commit: enter lock");
      }

      if (pendingCommit == null) {
        if (infoStream.isEnabled("IW")) {
          infoStream.message("IW", "commit: now prepare");
        }
        seqNo = prepareCommitInternal();
      } else {
        if (infoStream.isEnabled("IW")) {
          infoStream.message("IW", "commit: already prepared");
        }
        seqNo = pendingSeqNo;
      }

      finishCommit();
    }

    // we must do this outside of the commitLock else we can deadlock:
    if (maybeMerge.getAndSet(false)) {
      maybeMerge(mergePolicy, MergeTrigger.FULL_FLUSH, UNBOUNDED_MAX_MERGE_SEGMENTS);      
    }
    
    return seqNo;
  }
```

```java
  private long prepareCommitInternal() throws IOException {
    startCommitTime = System.nanoTime();
    synchronized(commitLock) {
      ensureOpen(false);
      if (infoStream.isEnabled("IW")) {
        infoStream.message("IW", "prepareCommit: flush");
        infoStream.message("IW", "  index before flush " + segString());
      }

      if (tragedy.get() != null) {
        throw new IllegalStateException("this writer hit an unrecoverable error; cannot commit", tragedy.get());
      }

      if (pendingCommit != null) {
        throw new IllegalStateException("prepareCommit was already called with no corresponding call to commit");
      }

      doBeforeFlush();
      testPoint("startDoFlush");
      SegmentInfos toCommit = null;
      boolean anyChanges = false;
      long seqNo;

      // This is copied from doFlush, except it's modified to
      // clone & incRef the flushed SegmentInfos inside the
      // sync block:

      try {

        synchronized (fullFlushLock) {
          boolean flushSuccess = false;
          boolean success = false;
          try {
            //使用document wirter进行flush
            seqNo = docWriter.flushAllThreads();
            if (seqNo < 0) {
              anyChanges = true;
              seqNo = -seqNo;
            }
            if (anyChanges == false) {
              // prevent double increment since docWriter#doFlush increments the flushcount
              // if we flushed anything.
              flushCount.incrementAndGet();
            }
            publishFlushedSegments(true);
            // cannot pass triggerMerges=true here else it can lead to deadlock:
            processEvents(false);
            
            flushSuccess = true;

            applyAllDeletesAndUpdates();
            synchronized(this) {
              writeReaderPool(true);
              if (changeCount.get() != lastCommitChangeCount) {
                // There are changes to commit, so we will write a new segments_N in startCommit.
                // The act of committing is itself an NRT-visible change (an NRT reader that was
                // just opened before this should see it on reopen) so we increment changeCount
                // and segments version so a future NRT reopen will see the change:
                changeCount.incrementAndGet();
                segmentInfos.changed();
              }

              if (commitUserData != null) {
                Map<String,String> userData = new HashMap<>();
                for(Map.Entry<String,String> ent : commitUserData) {
                  userData.put(ent.getKey(), ent.getValue());
                }
                segmentInfos.setUserData(userData, false);
              }

              // Must clone the segmentInfos while we still
              // hold fullFlushLock and while sync'd so that
              // no partial changes (eg a delete w/o
              // corresponding add from an updateDocument) can
              // sneak into the commit point:
              toCommit = segmentInfos.clone();

              pendingCommitChangeCount = changeCount.get();

              // This protects the segmentInfos we are now going
              // to commit.  This is important in case, eg, while
              // we are trying to sync all referenced files, a
              // merge completes which would otherwise have
              // removed the files we are now syncing.    
              filesToCommit = toCommit.files(false); 
              deleter.incRef(filesToCommit);
            }
            success = true;
          } finally {
            if (!success) {
              if (infoStream.isEnabled("IW")) {
                infoStream.message("IW", "hit exception during prepareCommit");
              }
            }
            assert holdsFullFlushLock();
            // Done: finish the full flush!
            docWriter.finishFullFlush(flushSuccess);
            doAfterFlush();
          }
        }
      } catch (VirtualMachineError tragedy) {
        tragicEvent(tragedy, "prepareCommit");
        throw tragedy;
      } finally {
        maybeCloseOnTragicEvent();
      }
     
      try {
        if (anyChanges) {
          //视情况进行merge
          maybeMerge.set(true);
        }
        startCommit(toCommit);
        if (pendingCommit == null) {
          return -1;
        } else {
          return seqNo;
        }
      } catch (Throwable t) {
        synchronized (this) {
          if (filesToCommit != null) {
            try {
              deleter.decRef(filesToCommit);
            } catch (Throwable t1) {
              t.addSuppressed(t1);
            } finally {
              filesToCommit = null;
            }
          }
        }
        throw t;
      }
    }
  }
```

```java
@SuppressWarnings("try")
  private void finishCommit() throws IOException {

    boolean commitCompleted = false;
    String committedSegmentsFileName = null;

    try {
      synchronized(this) {
        ensureOpen(false);

        if (tragedy.get() != null) {
          throw new IllegalStateException("this writer hit an unrecoverable error; cannot complete commit", tragedy.get());
        }

        if (pendingCommit != null) {
          final Collection<String> commitFiles = this.filesToCommit;
          try (Closeable finalizer = () -> deleter.decRef(commitFiles)) {

            if (infoStream.isEnabled("IW")) {
              infoStream.message("IW", "commit: pendingCommit != null");
            }

            committedSegmentsFileName = pendingCommit.finishCommit(directory);

            // we committed, if anything goes wrong after this, we are screwed and it's a tragedy:
            commitCompleted = true;

            if (infoStream.isEnabled("IW")) {
              infoStream.message("IW", "commit: done writing segments file \"" + committedSegmentsFileName + "\"");
            }

            // NOTE: don't use this.checkpoint() here, because
            // we do not want to increment changeCount:
            deleter.checkpoint(pendingCommit, true);

            // Carry over generation to our master SegmentInfos:
            //更新segment info
            segmentInfos.updateGeneration(pendingCommit);

            lastCommitChangeCount = pendingCommitChangeCount;
            rollbackSegments = pendingCommit.createBackupSegmentInfos();

          } finally {
            notifyAll();
            pendingCommit = null;
            this.filesToCommit = null;
          }
        } else {
          assert filesToCommit == null;
          if (infoStream.isEnabled("IW")) {
            infoStream.message("IW", "commit: pendingCommit == null; skip");
          }
        }
      }
    } catch (Throwable t) {
      if (infoStream.isEnabled("IW")) {
        infoStream.message("IW", "hit exception during finishCommit: " + t.getMessage());
      }
      if (commitCompleted) {
        tragicEvent(t, "finishCommit");
      }
      throw t;
    }

    if (infoStream.isEnabled("IW")) {
      infoStream.message("IW", String.format(Locale.ROOT, "commit: took %.1f msec", (System.nanoTime()-startCommitTime)/1000000.0));
      infoStream.message("IW", "commit: done");
    }
  }
```
#### 4.2.3、Flush
一旦IndexWriter开始flush，它会调用自身的documentWriter的[flushAllThreads](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/DocumentsWriter.java#L639)方法：

```java
/*
   * FlushAllThreads is synced by IW fullFlushLock. Flushing all threads is a
   * two stage operation; the caller must ensure (in try/finally) that finishFlush
   * is called after this method, to release the flush lock in DWFlushControl
   */
  long flushAllThreads()
    throws IOException {
    final DocumentsWriterDeleteQueue flushingDeleteQueue;
    if (infoStream.isEnabled("DW")) {
      infoStream.message("DW", "startFullFlush");
    }

    long seqNo;
    synchronized (this) {
      pendingChangesInCurrentFullFlush = anyChanges();
      flushingDeleteQueue = deleteQueue;
      /* Cutover to a new delete queue.  This must be synced on the flush control
       * otherwise a new DWPT could sneak into the loop with an already flushing
       * delete queue */
      seqNo = flushControl.markForFullFlush(); // swaps this.deleteQueue synced on FlushControl
      assert setFlushingDeleteQueue(flushingDeleteQueue);
    }
    assert currentFullFlushDelQueue != null;
    assert currentFullFlushDelQueue != deleteQueue;
    
    boolean anythingFlushed = false;
    try {
      DocumentsWriterPerThread flushingDWPT;
      // Help out with flushing:
      while ((flushingDWPT = flushControl.nextPendingFlush()) != null) {
        //使用DWPT实际执行flush
        anythingFlushed |= doFlush(flushingDWPT);
      }
      // If a concurrent flush is still in flight wait for it
      flushControl.waitForFlush();  
      if (anythingFlushed == false && flushingDeleteQueue.anyChanges()) { // apply deletes if we did not flush any document
        if (infoStream.isEnabled("DW")) {
          infoStream.message("DW", Thread.currentThread().getName() + ": flush naked frozen global deletes");
        }
        assert assertTicketQueueModification(flushingDeleteQueue);
        ticketQueue.addDeletes(flushingDeleteQueue);
      }
      // we can't assert that we don't have any tickets in teh queue since we might add a DocumentsWriterDeleteQueue
      // concurrently if we have very small ram buffers this happens quite frequently
      assert !flushingDeleteQueue.anyChanges();
    } finally {
      assert flushingDeleteQueue == currentFullFlushDelQueue;
      flushingDeleteQueue.close(); // all DWPT have been processed and this queue has been fully flushed to the ticket-queue
    }
    if (anythingFlushed) {
      return -seqNo;
    } else {
      return seqNo;
    }
  }
```

```java
private boolean doFlush(DocumentsWriterPerThread flushingDWPT) throws IOException {
    boolean hasEvents = false;
    while (flushingDWPT != null) {
      assert flushingDWPT.hasFlushed() == false;
      hasEvents = true;
      boolean success = false;
      DocumentsWriterFlushQueue.FlushTicket ticket = null;
      try {
        assert currentFullFlushDelQueue == null
            || flushingDWPT.deleteQueue == currentFullFlushDelQueue : "expected: "
            + currentFullFlushDelQueue + "but was: " + flushingDWPT.deleteQueue
            + " " + flushControl.isFullFlush();
        /*
         * Since with DWPT the flush process is concurrent and several DWPT
         * could flush at the same time we must maintain the order of the
         * flushes before we can apply the flushed segment and the frozen global
         * deletes it is buffering. The reason for this is that the global
         * deletes mark a certain point in time where we took a DWPT out of
         * rotation and freeze the global deletes.
         * 
         * Example: A flush 'A' starts and freezes the global deletes, then
         * flush 'B' starts and freezes all deletes occurred since 'A' has
         * started. if 'B' finishes before 'A' we need to wait until 'A' is done
         * otherwise the deletes frozen by 'B' are not applied to 'A' and we
         * might miss to deletes documents in 'A'.
         */
        //delete操作需要顺序一致，多线程flush情况下需要维持有序
        try {
          assert assertTicketQueueModification(flushingDWPT.deleteQueue);
          // Each flush is assigned a ticket in the order they acquire the ticketQueue lock
          ticket = ticketQueue.addFlushTicket(flushingDWPT);
          final int flushingDocsInRam = flushingDWPT.getNumDocsInRAM();
          boolean dwptSuccess = false;
          try {
            // flush concurrently without locking
            final FlushedSegment newSegment = flushingDWPT.flush(flushNotifications);
            ticketQueue.addSegment(ticket, newSegment);
            dwptSuccess = true;
          } finally {
            subtractFlushedNumDocs(flushingDocsInRam);
            if (flushingDWPT.pendingFilesToDelete().isEmpty() == false) {
              Set<String> files = flushingDWPT.pendingFilesToDelete();
              flushNotifications.deleteUnusedFiles(files);
              hasEvents = true;
            }
            if (dwptSuccess == false) {
              flushNotifications.flushFailed(flushingDWPT.getSegmentInfo());
              hasEvents = true;
            }
          }
          // flush was successful once we reached this point - new seg. has been assigned to the ticket!
          success = true;
        } finally {
          if (!success && ticket != null) {
            // In the case of a failure make sure we are making progress and
            // apply all the deletes since the segment flush failed since the flush
            // ticket could hold global deletes see FlushTicket#canPublish()
            ticketQueue.markTicketFailed(ticket);
          }
        }
        /*
         * Now we are done and try to flush the ticket queue if the head of the
         * queue has already finished the flush.
         */
        if (ticketQueue.getTicketCount() >= perThreadPool.size()) {
          // This means there is a backlog: the one
          // thread in innerPurge can't keep up with all
          // other threads flushing segments.  In this case
          // we forcefully stall the producers.
          flushNotifications.onTicketBacklog();
          break;
        }
      } finally {
        flushControl.doAfterFlush(flushingDWPT);
      }
     
      flushingDWPT = flushControl.nextPendingFlush();
    }

    if (hasEvents) {
      flushNotifications.afterSegmentsFlushed();
    }

    // If deletes alone are consuming > 1/2 our RAM
    // buffer, force them all to apply now. This is to
    // prevent too-frequent flushing of a long tail of
    // tiny segments:
    final double ramBufferSizeMB = config.getRAMBufferSizeMB();
    if (ramBufferSizeMB != IndexWriterConfig.DISABLE_AUTO_FLUSH &&
        flushControl.getDeleteBytesUsed() > (1024*1024*ramBufferSizeMB/2)) {
      hasEvents = true;
      if (applyAllDeletes() == false) {
        if (infoStream.isEnabled("DW")) {
          infoStream.message("DW", String.format(Locale.ROOT, "force apply deletes after flush bytesUsed=%.1f MB vs ramBuffer=%.1f MB",
                                                 flushControl.getDeleteBytesUsed()/(1024.*1024.),
                                                 ramBufferSizeMB));
        }
        flushNotifications.onDeletesApplied();
      }
    }

    return hasEvents;
  }
```
DWPT中的flush操作较为复杂，不在此处列出，详情请参考[此处](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/DocumentsWriterPerThread.java#L367)
#### 4.2.4、Merge
Merges会暂时性地消耗directory的磁盘空间。如果索引没有打开任何readers/searchers ，那么将会暂时性消耗所有被merge的segment所使用的空间。如果索引的所有即将被merge的segment都有被打开的readers/searchers，merge操作将会暂时性消耗所有被merge的segment所使用的空间的2倍（详情参考[forceMerge(int)](https://lucene.apache.org/core/7_7_2/core/org/apache/lucene/index/IndexWriter.html#forceMerge-int-)）

```java
final void maybeMerge(MergePolicy mergePolicy, MergeTrigger trigger, int maxNumSegments) throws IOException {
    ensureOpen(false);
    if (updatePendingMerges(mergePolicy, trigger, maxNumSegments)) {
      mergeScheduler.merge(this, trigger);
    }
  }

  private synchronized boolean updatePendingMerges(MergePolicy mergePolicy, MergeTrigger trigger, int maxNumSegments)
    throws IOException {

    // In case infoStream was disabled on init, but then enabled at some
    // point, try again to log the config here:
    messageState();

    assert maxNumSegments == UNBOUNDED_MAX_MERGE_SEGMENTS || maxNumSegments > 0;
    assert trigger != null;
    if (stopMerges) {
      return false;
    }

    // Do not start new merges if disaster struck
    if (tragedy.get() != null) {
      return false;
    }
    boolean newMergesFound = false;
    final MergePolicy.MergeSpecification spec;
    if (maxNumSegments != UNBOUNDED_MAX_MERGE_SEGMENTS) {
      assert trigger == MergeTrigger.EXPLICIT || trigger == MergeTrigger.MERGE_FINISHED :
      "Expected EXPLICT or MERGE_FINISHED as trigger even with maxNumSegments set but was: " + trigger.name();

      spec = mergePolicy.findForcedMerges(segmentInfos, maxNumSegments, Collections.unmodifiableMap(segmentsToMerge), this);
      newMergesFound = spec != null;
      if (newMergesFound) {
        final int numMerges = spec.merges.size();
        for(int i=0;i<numMerges;i++) {
          final MergePolicy.OneMerge merge = spec.merges.get(i);
          merge.maxNumSegments = maxNumSegments;
        }
      }
    } else {
      spec = mergePolicy.findMerges(trigger, segmentInfos, this);
    }
    newMergesFound = spec != null;
    if (newMergesFound) {
      final int numMerges = spec.merges.size();
      for(int i=0;i<numMerges;i++) {
        registerMerge(spec.merges.get(i));
      }
    }
    return newMergesFound;
  }
```
registerMerge之后将会由mergeScheduler来实际决定merge时机，并实际执行。
mergeScheduler是一个抽象类，其实现是ConcurrentMergeScheduler。其篇幅较大不在此处展开，其详细[merge](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/ConcurrentMergeScheduler.java#L497)过程参考[此处](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/ConcurrentMergeScheduler.java#L497)。下方也会相对详细地介绍。
#### 4.2.5、Close
```java
 @Override
  public void close() throws IOException {
    closed = true;
    IOUtils.close(flushControl, perThreadPool);
  }
```

```java
@Override
    public synchronized void close() throws IOException { // synced to prevent double closing
      assert closed == false : "we should never close this twice";
      closed = true;
      // it's possible that we close this queue while we are in a processEvents call
      if (writer.getTragicException() != null) {
        // we are already handling a tragic exception let's drop it all on the floor and return
        queue.clear();
      } else {
        // now we acquire all the permits to ensure we are the only one processing the queue
        try {
          permits.acquire(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
          throw new ThreadInterruptedException(e);
        }
        try {
          processEventsInternal();
        } finally {
          permits.release(Integer.MAX_VALUE);
        }
      }
    }
  }
```
close将会完成所有DWPT的工作，完成未完成的flush，注册所有merge，释放directory锁。close后该IW不能再修改索引。详情参考[此处](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/IndexWriter.java#L352)
### 4.3、IndexWriter构建
由上文介绍可知，只要能够构建一个IndexWriter对象，它就可以调用lucene的各处api来完成索引的写入，所以IndexWriter对象的构建即为写入索引之前的前置工作。
IndexWriter的构造函数定义如下：

```java
public IndexWriter(Directory d, IndexWriterConfig conf) throws IOException
```

可见需要Directory与IndexWriterConfig两个类的对象作为辅助
#### 4.3.1、Directory
[Directory](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/store/Directory.java#L50)是一个抽象意义上的目录，它代表一系列文件（它不包含任何子文件夹），对于一般的应用，它可以是存放于硬盘上的目录、流式目录。对于一个常规的lucene index，它是一个磁盘上的目录，它存放lunene index涉及到的所有文件（segment）。
一般的创建索引的需求下，用户可以建立一个[FSDirectory](https://github.com/apache/lucene-solr/blob/624f5a3c2f5ab25a44b3e3843dbef36d4ed70602/lucene/core/src/java/org/apache/lucene/store/FSDirectory.java)：

```java
protected FSDirectory(Path path, LockFactory lockFactory) throws IOException {
    super(lockFactory);
    // If only read access is permitted, createDirectories fails even if the directory already exists.
    if (!Files.isDirectory(path)) {
      Files.createDirectories(path);  // create directory, if it doesn't exist
    }
    directory = path.toRealPath();
  }
```
建立Directory并传递给IW后，索引的存储位置就固定了。
#### 4.3.2、IndexWriterConfig
[IndexWriterConfig](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/IndexWriterConfig.java#L58)可以定义IW的打开模式，使用的analyzer，index的lucene版本，lucene codec，mergeScheduler和delete/flush/mergePolicy等等，它决定了IW的所有行为模式。它的代码结构为getter和setter组成的表结构，详情请参考[此处](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/IndexWriterConfig.java#L58)。
其中[analyzer](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/analysis/Analyzer.java#L85)analyzer是一种接口，它可以定义分词规则。使得某一field中的文本产生多个term。ES自身自带了多种analyzer的实现用以应对各种实际场景中的分词需求：
- [Standard Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html)
- [Simple Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-simple-analyzer.html)
- [Whitespace Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-whitespace-analyzer.html)
- [Stop Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-stop-analyzer.html)
- [Keyword Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-keyword-analyzer.html)
- [Pattern Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-pattern-analyzer.html)
- [Fingerprint Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-fingerprint-analyzer.html)

这些analyzer主要应对英文语境内的分词需求，对于中文或其它应用，可以自己实现自定义的analyzer，用户需要实现：
1. Character filters：在文字流传入tokenizer之前进行初步处理
2. tokenizer：对已经进行预处理的文字流进行分词
3. Token filters：对完成分词后的token集合进行后处理
用户可以通过插件的形式为es安装各种analyzer，并通过配置在建立IW的时候投入使用。
### 4、IndexWriter使用的ConcurrentMergeScheduler
ConcurrentMergeScheduler是一种MergeScheduler的多线程并行实现，也是实际使用的MergeScheduler。它来实际意义上对已经注册的merge进行操作。

```java
@Override
  public synchronized void merge(IndexWriter writer, MergeTrigger trigger) throws IOException {

    assert !Thread.holdsLock(writer);

    initDynamicDefaults(writer);

    if (trigger == MergeTrigger.CLOSING) {
      // Disable throttling on close:
      targetMBPerSec = MAX_MERGE_MB_PER_SEC;
      updateMergeThreads();
    }

    // First, quickly run through the newly proposed merges
    // and add any orthogonal merges (ie a merge not
    // involving segments already pending to be merged) to
    // the queue.  If we are way behind on merging, many of
    // these newly proposed merges will likely already be
    // registered.

    if (verbose()) {
      message("now merge");
      message("  index: " + writer.segString());
    }
    
    // Iterate, pulling from the IndexWriter's queue of
    // pending merges, until it's empty:
    while (true) {

      if (maybeStall(writer) == false) {
        break;
      }

      OneMerge merge = writer.getNextMerge();
      if (merge == null) {
        if (verbose()) {
          message("  no more merges pending; now return");
        }
        return;
      }

      boolean success = false;
      try {
        if (verbose()) {
          message("  consider merge " + writer.segString(merge.segments));
        }

        // OK to spawn a new merge thread to handle this
        // merge:
        final MergeThread newMergeThread = getMergeThread(writer, merge);
        mergeThreads.add(newMergeThread);

        updateIOThrottle(newMergeThread.merge, newMergeThread.rateLimiter);

        if (verbose()) {
          message("    launch new thread [" + newMergeThread.getName() + "]");
        }

        newMergeThread.start();
        updateMergeThreads();

        success = true;
      } finally {
        if (!success) {
          writer.mergeFinish(merge);
        }
      }
    }
  }
```
mergeThread是一个merge进程，它的定义参考[此处](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/ConcurrentMergeScheduler.java#L659)
在mergeThread中，它会调用它所对应的IW的merge方法：

```java
public void merge(MergePolicy.OneMerge merge) throws IOException {

    boolean success = false;

    final long t0 = System.currentTimeMillis();

    final MergePolicy mergePolicy = config.getMergePolicy();
    try {
      try {
        try {
          mergeInit(merge);

          if (infoStream.isEnabled("IW")) {
            infoStream.message("IW", "now merge\n  merge=" + segString(merge.segments) + "\n  index=" + segString());
          }

          mergeMiddle(merge, mergePolicy);
          mergeSuccess(merge);
          success = true;
        } catch (Throwable t) {
          handleMergeException(t, merge);
        }
      } finally {
        synchronized(this) {

          mergeFinish(merge);

          if (success == false) {
            if (infoStream.isEnabled("IW")) {
              infoStream.message("IW", "hit exception during merge");
            }
          } else if (!merge.isAborted() && (merge.maxNumSegments != UNBOUNDED_MAX_MERGE_SEGMENTS || (!closed && !closing))) {
            // This merge (and, generally, any change to the
            // segments) may now enable new merges, so we call
            // merge policy & update pending merges.
            updatePendingMerges(mergePolicy, MergeTrigger.MERGE_FINISHED, merge.maxNumSegments);
          }
        }
      }
    } catch (Throwable t) {
      // Important that tragicEvent is called after mergeFinish, else we hang
      // waiting for our merge thread to be removed from runningMerges:
      tragicEvent(t, "merge");
      throw t;
    }

    if (merge.info != null && merge.isAborted() == false) {
      if (infoStream.isEnabled("IW")) {
        infoStream.message("IW", "merge time " + (System.currentTimeMillis()-t0) + " msec for " + merge.info.info.maxDoc() + " docs");
      }
    }
  }
```
其中的[mergeMiddle](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/IndexWriter.java#L4381)解释了merge的本质：计算所有涉及到的segment的update/delete操作，然后再将所有segment中的数据重新排列，写入到新的segment中。其中使用了[segmentMerger](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/SegmentMerger.java#L43)进行实际的处理，该类处理较为复杂不在此处展开，详情参考[此处](https://github.com/apache/lucene-solr/blob/a11b78e06a5947ffb43a9b66b37033ebe64753e0/lucene/core/src/java/org/apache/lucene/index/SegmentMerger.java#L43)。
