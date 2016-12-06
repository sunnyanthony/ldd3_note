# Ch9.硬體的操作

## I/O port and I/O memory
通常一個device會有許多registers，並且可以在__一段連續範圍__的address存取到這些registers。這些address會有兩種address space:     
  * Memory address space
  * I/O address space     

Linux虛構了一組I/O port的存取機制，即使target plantform的CPU只有單一address space。實際程序是依照CPU跟BUS的Chipset決定的。

#### I/O register and 傳統memory
雖然register跟memory很相似，但在program存取I/O port時，不能將I/O port當成normal memory。這是因為compiler在最佳化過程中會影響實際的存取順序，這樣會導致預期的I/O行為被改變。     
這是因為讀取I/O registers時，並__不一定是要讀取register的內容值__，而是藉由__read/write的動作__來__改變device__的state。當我們將normal memory的最佳化技術(cache, reschedule instruction, etc.)
