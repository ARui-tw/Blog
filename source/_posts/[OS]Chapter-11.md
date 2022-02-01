---
title: "[OS] Chapter 11 — File System Implementation"
date: 2022-01-31 18:11:24
tags:
    - Operating System
    - Notes
categories: Operating System Notes
---

{% note default %}
[Notion 好讀版](https://arui-tw.notion.site/Chapter-11-File-System-Implementation-3bd2f7997ad041ac8966f8bd52fd1b18)
{% endnote %}

# File-System Structure

- 一個 disk 提供的最小單位是 sectors（usually 512 bytes）
- 但是 mapping 的時候不喜歡用 sector，太小了
- 所以都用 Blocks，通常是四到十幾個 sectors
- 一個 OS 可以 support > 1 個 FS type
- 需要兩個 interface
    - 對上：user programs
    - 對下：physical storage (disk)

## Layered File System

- 最上層是 API
- 最下層（最終要寫到 disk 上面去的）要透過 driver 寫過去，那 driver 要的是單位是 sector
- Logical file system — 根據 file path 找到 file ID，也要去判斷權限（i.e. 判斷 API 是不是合法可執行的東西）
- File-organization module — 每一個 FS 最不一樣的地方。要找到在 logical 中的位址對應到的 physical 的地方。（甚至需要 map 到不同的電腦）
- Basic File system — 最基本的 I/O 的動作（跟檔案管理無關啦）

![右邊的名詞不用記](/assets/[OS]Chapter-11/Untitled.png)

- 之所以一個 `fread` 可以對不同的 file system 作操作就是中間有 FS manager (VFS — Virtual File System)
    
    ![](/assets/[OS]Chapter-11/Untitled%201.png)
    

<!-- more -->

# File System Implementation

## On-Disk Structure

→ 因為 reboot 後，資料還是要在，所以和之前的不一樣

- Boot control **block** ****(per partition) — 關於 booting 的 code 或資訊
    - 裝 OS 他就會幫你 create 起來
    - 一個 partition 只會有一個（也最多一個） boot control block，通常會放在最上面
    - 如果不是開機槽，那就不會有這個 block
    - Unix File System → boot block; NTFS → partition boot sector
- Partition control block ****(per partition) — 關於這個 partition 的資訊
    - details: # of blocks, block size, free-block-list, free FCB pointers, etc
    - UFS → superblock; NTFS → Master File Table
    - 如果沒有 boot control block，那他應該會在最前面
- File control block (per file) — file 的 metadata
    - 注意不存 data，data 會有一個指標指過去
    - UFS — **inode**; NTFS: stored in MFT (relational database)
- Directory structure (per file system): organize files

![](/assets/[OS]Chapter-11/Untitled%202.png)

- `$ stat`
    
    ![](/assets/[OS]Chapter-11/Untitled%203.png)
    


## In-Memory Structure

→ 一樣的，只是哪些東西要被 load 到 memory 裡面

- in-memory partition table — 有人被 mount，他就會存關於這個 partition 的資訊。Unmount  就會被移除。
- in-memory directory structure — 最近 accessed 到的 directory
- system-wide open-file table — 被 open 的 file 的 file control block
- per-process open-file table — process 對於一個 file 需要 keep 的資訊，且 file close 的時候，他就會被清掉，不會寫回去。

**File-Open**

![](/assets/[OS]Chapter-11/Untitled%204.png)

- 如果沒讀過這個 directory，他就會去 disk （secondary storage）的地方把他拉出來
- Directory 的最後接的是 file control block

**File-Read**

![](/assets/[OS]Chapter-11/Untitled%205.png)

- 上面 open 好了，所以兩個 open-file table 都 ready 了
1. 去 per-process 確認有沒有權限（logical file system）
2. 如果有，去 system-wide 找 file control block 裡面存的 data block 的 pointer

**File-Create**

1. OS allocates a new FCB
2. Update directory structure
    1. OS read in the corresponding directory structure into memory
    2. **Updates the dir structure** with the new file name and the FCB
    3. (After file being closed), OS writes back the directory structure back to disk → Close 才會寫回去，因為 disk 太慢了
3. The file appears in user’s dir command

## Virtual File System

- Four main object types defined by Linux VFS Disk Allocation Methods
    - inode (FCB) → an individual **file**
    - file object → an **open file** (per process)
    - superblock object → an entire file system
    - entry object → an individual directory entry

## **Directory Implementation**

大型系統的 directory 太大了，且掉了很慘

- Linear lists
    - use link list and traverse
    - easy to program but poor performance
- Hash table
    - 把 file path 當成 key，FCB 是 value
    - constant time for searching
    - linked list for collisions on a hash entry
    - hash table usually has fixed # of entries

## Review Slides (I)

- Transfer unit between memory and disk?
- App → LFS → FOM → BFS → I/O Control → Devices
- on-disk structure
    - Boot control block, Partition control block
    - File control block, Directory structure
- In-memory
    - Partition table, Directory structure
    - System-wide open-file table
    - Per-process open-file table
- Steps to open file, read/write file and create file?
- Purpose of VFS?

# Allocation Methods

把 file 對應到 block。

## Contiguous Allocation

![右邊的表，每一行就是一個 inode，start 和 length 是 metadata](/assets/[OS]Chapter-11/Untitled%206.png)


- 一個 block 一定只屬於一個 file
- Sequential access 和 random access 都很有效率（根據 I/O 的次數作判斷）→ random 是因為可以用算的飛過去
- ☹️ Problems
    - External fragmentation → compaction（硬碟清理）
    - File cannot grow（會被後面卡住）→ 搬移資料（big cost）

## Extent-Based File System (Ext)

- 用 link list 的方法把 extent 串起來
- 所以除了存 start 和 length，還要多存 pointer to next extent
- ☹️ Problems
    - Random access become more costly
    - “Internal fragmentation” → extent 給他的 blocks 沒用完

## Linked Allocation

![](/assets/[OS]Chapter-11/Untitled%207.png)

- 把下一個 block 的 pointer 存在 data block 裡面，dir 只會有第一個 pointer
- Only good for **sequential-access** files, random access need to traverse through the link list
- Each access to a link list is a disk I/O (because link pointer is stored inside the data block)
- 每一個 data block 都會浪費 4 byte 去存 pointer
- Reliability → 斷了就 GG

## FAT (File Allocation Table) file system

![](/assets/[OS]Chapter-11/Untitled%208.png)

- 把整個 pointers 存起來，所以就變成一個純 link list，存取還是需要 traverse，但是不用 I/O，只會有很快的 memory access
- 超快，但是需要空間
- Link 斷了也 GG

## Indexed Allocation

![](/assets/[OS]Chapter-11/Untitled%209.png)

- 跟剛剛很像，但是他是用 index，不用 traverse
- 很快，random 也超快
- 注意那個 index 也是存在 disk 上了，他也是一個 block
- 所以 inode 會存那個 index 在的 block
- ☹️
    - Index block 很浪費空間，小的 block 根本用不完
    - Index block 如果不夠大？
        - Linked Indexed Scheme
            
            → search time 會變長
            
            ![](/assets/[OS]Chapter-11/Untitled%2010.png)
            
        - Multilevel Scheme (two-level)
            
            → 大 file 這個比較優
            
            → 小 file 就很浪費
            
            ![](/assets/[OS]Chapter-11/Untitled%2011.png)
            

## Combined Scheme: Unix inode

根據 file size 決定用哪個

![](/assets/[OS]Chapter-11/Untitled%2012.png)

# Free-Space Management

- 找到全部的 free block
- Free-space list: records all free disk blocks
- Scheme
    - Bit vector
        
        ![](/assets/[OS]Chapter-11/Untitled%2013.png)
        
        - 用一串 1 和 0 去紀錄他是不是 free 的
        - 🙂 simplicity, efficient
        - ☹️ 因為是 fix size，會浪費 memory 空間
    - Linked list (same as linked allocation)
        
        ![](/assets/[OS]Chapter-11/Untitled%2014.png)
        
    - Grouping (same as linked index allocation)
    - Counting (same as contiguous allocation)
        
        → keep the address of the first free block and # of contiguous free blocks
        

## Review Slides (II)

- Allocation:
    - Contiguous file allocation? Extent-based file system?
    - Linked allocation?
    - Indexed allocation?
        - Linked scheme
        - multilevel index allocation
        - Combine scheme
- Free space:
    - Bit vector, linked list, counting, grouping

# Textbook Questions

![](/assets/[OS]Chapter-11/Untitled%2015.png)

![](/assets/[OS]Chapter-11/Untitled%2016.png)

![](/assets/[OS]Chapter-11/Untitled%2017.png)


{% note warning %}  
此筆記為清華大學周志遠教授作業系統之課堂筆記，所有內容及圖片皆取材於課堂內容。<br />
如內容有誤，歡迎來信 [mail@arui.dev](mailto:mail@arui.dev)。
{% endnote %}