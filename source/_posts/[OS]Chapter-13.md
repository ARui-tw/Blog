---
title: "[OS] Chapter 13 — I/O System"
date: 2022-01-31 18:12:24
tags:
    - Operating System
    - Notes
categories: Operating System Notes
---

{% note default %}
[Notion 好讀版](https://www.notion.so/arui-tw/Chapter-13-I-O-System-7f1b37167d5b4a77bc4e18c96b74ed95)
{% endnote %}

# Overview

- 電腦不是在做 I/O 就是在做計算
- **I/O devices**: tape, HD, mouse, joystick, network card, screen, flash disks, etc
- **I/O subsystem**: the methods to **control all I/O devices**
- Two conflicting trends
    - 我們希望他是 interface，這樣才可以 standardization 的控制
    - 但是他又很多樣性
- **Device drivers**: a uniform device-access **interface** to the I/O subsystem
    
    ⇒ Similar to system calls between apps and OS
    
- Device categories
    - Storage devices: disks, tapes
    - Transmission devices: network cards, modems
    - Human-interface devices: keyboard, screen, mouse
    - Specialized devices: joystick, touchpad

# I/O Hardware

- **Port**: A **connection point** between I/O devices and the host. 每個 port 都有自己的 ID，那就是 device 對於電腦的 identifier。
    
    → E.g.: USB ports
    
- **Controller**: A collection of electronics that can operate a port, a bus, or a device
    
    ⇒ A controller could have its own processor, memory, etc. (E.g.: SCSI controller)
    
- **Bus**: A set of **wires and a well-defined protocol** that specifies messages sent over the wires（連接的方式，除了物理相接，還有 protocol 的部分）
    
    → E.g.: PCI bus

<!-- more -->

![](/assets/[OS]Chapter-13/Untitled.png)

![](/assets/[OS]Chapter-13/Untitled%201.png)


# I/O Methods

- 透過 `port address` 做 `什麼事`
- Port address 用一個簡單的 table 就好了，反正也沒有很大
- Each I/O port consists of four registers (1~4 Bytes)
    - Data-in register — In buffer
    - Data-out register — Out buffer
    - Status register — 狀態的值
    - Control register — command

## I/O Methods Categorization

- Depending on how to address a device
    - Port-mapped I/O
        - 直接用 I/O instruction (e.g. `IN`, `OUT`)
    - Memory-mapped I/O
        - Access by standard data-transfer instruction (e.g. `MOV`)
        
        🙂 More efficient for **large memory I/O** (e.g. graphic card)
        
        ☹️ Vulnerable to accidental modification, error
        
- Depending on how to interact with a device:
    - **Poll** (busy-waiting): processor periodically check status register of a device. 不停地確認好了沒，啥時可以作下一個
        
        → 如果量不多，用這個
        
    - **Interrupt**: device notify processor of its completion. I/O 好了他會直接 call interrupt 跟你的程式講。
    
    ⇒ 通常會 Memory-mapped I/O ＋ Interrupt 或是另外兩個相加，不然會互相抵銷好處。
    
- Depending on who to control the transfer:
    - **Programmed I/O**: transfer controlled by CPU
        
        → 需要一直用 CPU
        
    - **Direct memory access** (DMA) I/O: controlled by **DMA controller** (a special purpose controller)
        
        → 只負責 I/O
        
        - Design for **large data transfer**

## Interrupt

- Interrupt Vector Table
    
    ![數字越小越重要](/assets/[OS]Chapter-13/Untitled%202.png)
    
- Interrupt Prioritization
    - Maskable interrupt — 有一些不重要的 interrupt 會被 mask 掉
    - Non-maskable interrupt (NMI): highest-priority, never masked
        
        → Often used for power-down, memory error
        
- Interrupt-Driven I/O
    
    ![](/assets/[OS]Chapter-13/Untitled%203.png)
    
- CPU and device Interrupt handshake
    1. Device asserts **interrupt request** (IRQ) → I/O 要註冊他的 interrupt，這樣 OS 才知道要去處理誰
    2. CPU checks the **interrupt request line** at the beginning of each instruction cycle
    3. Save the status and address of interrupted process
    4. CPU acknowledges the interrupt and search the **interrupt vector** table for interrupt handler routines
    5. CPU fetches the next instruction from the **handler routine**
    6. Restore interrupted process after executing interrupt handler routine

## DMA (Direct Memory Access)

![](/assets/[OS]Chapter-13/Untitled%204.png)

File System → （我要讀多少資料，要讀到 X）→ device driver → （說「我要讀多少資料，要讀到 X」，並 register 一個 interrupt 說「做完之後跟我說欸」） → disk controller → （初始化 DMA，因為他只能傳很少的資料，所以 DMA 就開始不停地送搬移的指令到 disk controller，然後存到 X）→ DMA → （Through interrupt 到 CPU 說我搬完了）→ CPU → 看到搬完了，開心：）

注意那個 X 通常是 physical 的 address，不然就要去看 page table，那就需要 CPU 去跑 OS 來看。

→ 這樣 Virtual Memory 就有點失效，因為那塊就被 fix 住了。

## Review Slides ( I )

- Definition of I/O port? Bus? Controller?
- **I/O device and CPU communication?**
    - **Port-mapped vs. Memory-mapped**
    - **Poll vs. Interrupt**
    - **Programmed I/O vs. DMA**
- Steps to handle an interrupt I/O and DMA request?

# Kernel I/O Subsystem

- **I/O Scheduling**
- **Buffering — Bridge 速差**
- **Caching —** Always just a copy; Key to performance
- **Spooling — 像 buffer，但是是 all or nothing**
- **Error handling —** when I/O error happens
- **I/O protection —** Privileged instructions
- Device-status Table
    
    ![](/assets/[OS]Chapter-13/Untitled%205.png)
    
- Life Cycle of An I/O Request
    
    ![](/assets/[OS]Chapter-13/Untitled%206.png)
    

# Performance

- Intercomputer Communications

# Application Interface

# Textbook Questions

![](/assets/[OS]Chapter-13/Untitled%207.png)

![](/assets/[OS]Chapter-13/Untitled%208.png)

{% note warning %}  
此筆記為清華大學周志遠教授作業系統之課堂筆記，所有內容及圖片皆取材於課堂內容。<br />
如內容有誤，歡迎來信 [mail@arui.dev](mailto:mail@arui.dev)。
{% endnote %}