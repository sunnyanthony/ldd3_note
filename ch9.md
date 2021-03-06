# Ch9.硬體的操作

## I/O port and I/O memory
--------
通常一個device會有許多registers，並且可以在__一段連續範圍__的address存取到這些registers。這些address會有兩種address space:     
  * Memory address space
  * I/O address space     

Linux虛構了一組I/O port的存取機制，即使target plantform的CPU只有單一address space。實際程序是依照CPU跟BUS的Chipset決定的。

#### I/O register and 傳統memory
雖然register跟memory很相似，但在program存取I/O port時，不能將I/O port當成normal memory。這是因為compiler在最佳化過程中會影響實際的存取順序，這樣會導致預期的I/O行為被改變。     
這是因為讀取I/O registers時，並__不一定是要讀取register的內容值__，而是藉由__read/write的動作__來__改變device__的state。當我們將normal memory的最佳化技術(cache, reschedule instruction, etc.)套用在I/O registers上時就會發生無法預期的結果，這是因為原本安排的I/O行為可能不被正確的執行。     

{% method %}
要避免compiler將I/O register最佳化，Linux提供的做法是在可被最佳化的指令與不可被最佳化(必須不受更動的在硬體上執行)之間擺放__momory barrier__，並提供了四個Marco    

使用方法如下：    
```c
writel(dev->registers.addr, io_destination_address);
writel(dev->registers.size, io_size);
writel(dev->registers.operation, DEV_READ);
wmb();
writel(dev->registers.control, DEV_GO);
```
這範例是我們希望在開始動作之前，add, size跟operation的三個register先設定好，才會進行control的`write(..., DEV_GO)`。   

{% sample lang="kernel 2.6" %}
```c
#include <linux/kernel.h>
void barrier(void);
```
此Marco要求在barrier後的instruction要確實將資料寫到memory之中，而非register或cache。但容許reschedule instruction。
```c
#include <asm/system.h> / kernel被compile為SMP時，使用smp_*
void rmb(void); //void smp_rmb(void);
void wmb(void); //void smp_void wmb(void);
void mb(void);  //void smp_mb(void);
void read_barrier_depends(void); //弱化版rmb();
//void smp_read_barrier_depends(void);
```
rmb/wmb保證在barrier之前的read/write動作都會在後續任何read/write動作之前完成。mb則是同時保證read/write的行為。
{% sample lang="kernel 4.* " %}

{% endmethod %}

順帶一提，spin_lock, atomic_t 等同步處理也有memory barrier作用。並有些平台容許一個動作就給予一個barrier，可使用以下Macro：
```c
#define set_mb(var, value) do {var = value; mb();} while 0
//ps. 只有少數平台有set_rmb
```
`do ... while`是讓Marco展開後可以在不同環境下正常運作的慣例。有時候展開Marco會跟前後文不小心結合或讓if else判斷跟預期不同。

## I/O port 用法
--------
對於device driver而言，存取I/O port為重要的議題。

#### 配置I/O port
{% method %}
Allocation I/O port是為了獨佔I/O port的使用，kernel提供了一組allocation interface來索取所需的I/O。  

可透過 /proc/ioports 當中查看被記錄的所有allocation device的address範圍。
{% sample lang="kernel 2.6" %}
```C
#include <linux/ioport.h>
struct resource *request_region(unsigned long first, unsigned long n, const char *name);
```
request_region()是最關鍵的方法，告訴kernel想使用的I/O port是從first～first + n，而name是device的名稱。若回傳值為NULL，表示已被別搶先allocate。
```C
#include <linux/ioport.h>
void release_region(unsigned long start, unsigned long n);
```
可使用release_region()來釋放資源。  
{% endmethod %}

#### 操作I/O port
對I/O進行read/write時，會涉及到bus的寬度問題，因此對於不同的寬度需要不同的function來存取。  
對於只支援__MMIO__(memory-mapped I/O)的平台，可將I/O register mapping到memory address，藉此模擬出I/O port。
  
當我們看到只有unsigned卻沒有明確型別，表示確切型別隨平台而定。為的是提高移植性。  

並沒有64-bit port I/O，就算是64-bit的系統上，I/O port的address space頂多只有32-bit。
{% method %}
Linux kernel在`<asm/io.h>`定義了read/write I/O port的inline function。
{% sample lang="kernel 2.6" %}
```C
#include <asm/io.h>
unsigned inb(unsigned port);
void outb(unsigned char byte, unsigned port);
```
read/write 1-byte port。
```C
unsigned inw(unsigned port);
void outw(unsigned short word, unsigned port);
```
read/write 1-word(16-bit) port。在只支援byte I/O平台上不存在。
```C
unsigned inl(unsigned port);
void outl(unsigned longword, unsigned port);
```
read/write 32-bit port。在只支援byte I/O平台上不存在。
{% endmethod %}

#### 從user-space存取I/O port
GNU libc將可存取I/O port的fuction定義在`<sys/io.h>`。在user-space使用inb()系列fuction時，須遵守：
* 使用-O option編譯，強制展開inlin fuction
* 必須使用ioperm() or iopl() system call 來取得I/O port的權限。只能在intel x86上使用這輛個function。
* program要使用root權限來執行。或危險的設定setuid權限為元。
* target plantform上沒有ioperm() or iopl()，可透過/dev/port device file來存取I/O port。通常不太有用

#### String操作
{% method %}
有些processor還提供能讓一連串同等的bytes/words/longs read/write到同一個I/O port。這就是string instructions，並且它能夠用更快的速度做到C loop的效果。  
若平台不提供此種指令，linux提供的Marco就會以loop實做。  
  *b表示bytes(8-bits)*  
  *w表示words(16-bits word)*  
  *l表示longs(32-bits long word)*  
  *要注意big-edian或是little-edian，可能device跟plantform不同  
{% sample lang="kernel 2.6" %}
```C
#include <asm/io.h>
void insb(unsigned port, void *addr, unsigned long count);
void outsb(unsigned port, void *addr, unsigned long count);
void insw(unsigned port, void *addr, unsigned long count);
void outsw(unsigned port, void *addr, unsigned long count);
void insl(unsigned port, void *addr, unsigned long count);
void outsl(unsigned port, void *addr, unsigned long count);
```
insb從port讀取count個bytes，並存放到addr的memory address上。outsb則是寫入。  
{% endmethod %}

####I/O暫停
避免I/O速度較慢而流失data，可插入delay，或是*_p(inb_p...)這種會暫停的function。

####平台相依性
除了x86外，其他都沒有區分I/O-space跟memory-space。
```
IA-32 (x86)
-x86_64
The architecture supports all the functions described in this chapter. Port num- bers are of type unsigned short.
-IA-64 (Itanium)
All functions are supported; ports are unsigned long (and memory-mapped). String functions are implemented in C.
-Alpha
All the functions are supported, and ports are memory-mapped. The implemen- tation of port I/O is different in different Alpha platforms, according to the chipset they use. String functions are implemented in C and defined in arch/ alpha/lib/io.c. Ports are unsigned long.
-ARM
Ports are memory-mapped, and all functions are supported; string functions are implemented in C. Ports are of type unsigned int.
Cris
M68k
-M68k-nommu
Ports are memory-mapped. String functions are supported, and the port type is unsigned char *.
MIPS
-MIPS64
The MIPS port supports all the functions. String operations are implemented with tight assembly loops, because the processor lacks machine-level string I/O. Ports are memory-mapped; they are unsigned long.
-PA-RISC
All of the functions are supported; ports are int on PCI-based systems and unsigned short on EISA systems, except for string operations, which use unsigned long port numbers.
PowerPC
-PowerPC64
All the functions are supported; ports have type unsigned char * on 32-bit sys- tems and unsigned long on 64-bit systems.
-S390
Similar to the M68k, the header for this platform supports only byte-wide port I/O with no string operations. Ports are char pointers and are memory-mapped.
-Super-H
Ports are unsigned int (memory-mapped), and all the functions are supported.
SPARC
-SPARC64
Once again, I/O space is memory-mapped. Versions of the port functions are defined to work with unsigned long ports.
```
##I/O port的例子
--------
####Parallel Port的介紹
每台PC都會有兩個parallel port，第一個從0x378開始，第二個是從0x278開始。基本是由三個8-bits port register構成：
* data port
  * 第一個是雙向data register，連結到pin2 ~ pin9
* status port
  * 第二個是read-only，可得到device的狀態
* control port
  * 最後一個是write-only，決定是否要發出hardware interrupt等控制  

Figure 9-1是parallel port的規格。有12個ouput跟5個input。並且1,4,11,17的path上有inverter。並且control port的0x10用來決定interrupt。  
![Figure9-1](f9_1.jpg)  
  
自行去看ldd3提供的short driver，並可在板子上插入LED來觀察資料在pin的傳遞。

##使用I/O memory
--------
與device互動的主要機制是將device的registers跟memory mapping到CPU的memory-space。  
I/O memory不一定受到page table的控管，主要是看plantform跟bus來決定。必須要使用page table的I/O memory的情況下，driver必須要告知kernel將physical address搬入到driver的可見範圍(這表示需要先使用__ioremap()__)。  
不管要不要用page table(ioremap)，都要避免直接使用一般的pointer來存取I/O memory。

####I/O Memory Allocation and Mapping
{% method %}
在使用I/O memory之前需要先allocate kernel的存取權。可使用request_mem_region Macro。  
check就算成功也有可能在要配置時失敗，所以不如不要用。  
剛配置好的I/O memory還需要`ioremap()`將I/O memory mapping到virtual address，也就是指定virtual address到I/O memory regions。  
{% sample lang="kernel 2.6" %}
```C
#include <linux/ioport.h>
struct resource *request_mem_region(unsigned long start, unsigned long len, char *name);
void release_mem_region(unsigned long start, unsigned long len);
int check_mem_region(unsigned long start, unsigned long len);
```
要求kernel將start ~ start + len的address保留/歸還/檢查是否被佔用，如果失敗了會得到NULL。name是裝置名稱。  
```C
#include <asm/io.h>
void *ioremap(unsigned long phys_addr, unsigned long size);
void *iounmap(unsigned long phys_addr, unsigned long size);
void *ioremap_nocache(unsigned long phys_addr, unsigned long size);
void iounmap(void * addr);
```
使用ioremap()後需要使用iounmap來釋放。大多數pc上的ioremap()跟ioremap_nocache完全相同。因為所有I/O都是在不可cache的位址上。  
{% endmethod %}
####存取I/O memory

{% method %}  
為了增加移植性，所以要避免使用`ioremap()`回傳的pointer。應該透過`<asm/io.h>`提供的function來存取。
r表示read memory，w表示write memory。
{% sample lang="kernel 2.6" %}
```C
unsigned int ioread8(void *addr);
unsigned int ioread16(void *addr);
unsigned int ioread32(void *addr);
void iowrite8(u8 value, void *addr);
void iowrite16(u16 value, void *addr);
void iowrite32(u32 value, void *addr);
```
addr是ioremap()所回傳的address，或是addr + offset。
```C
void ioread8_rep(void *addr, void *buf, unsigned long count);
void ioread16_rep(void *addr, void *buf, unsigned long count);
void ioread32_rep(void *addr, void *buf, unsigned long count);
void iowrite8_rep(void *addr, const void *buf, unsigned long count);
void iowrite16_rep(void *addr, const void *buf, unsigned long count);
void iowrite32_rep(void *addr, const void *buf, unsigned long count);
```
連續重複讀寫__同一個addr__可使用上述的function。注意count是要被寫入的資料大小，buf是存放read/write資料的memory pointer。
```C
void memset_io(void *addr, u8 value, unsigned int count);
void memcpy_fromio(void *dest, void *source, unsigned int count);
void memcpy_toio(void *dest, void *source, unsigned int count);
```
使用上述function才可操作一段連續的memory。
```C
unsigned readb(address); //8-bit
unsigned readw(address); //16-bit
unsigned readl(address); //32-bit
void writeb(unsigned value, address);
void writew(unsigned value, address);
void writel(unsigned value, address);
/*
*某些64-bits平台提供readq()&writeq()，可用來處理PCI的quad-word(8-bytes)
*/
```
上面的是屬於較舊的kernel所使用，較不安全，因為這些functions並沒有型別檢查。
{% endmethod %}

####Port as I/O memory
{% method %} 
為減少I/O port跟I/O memmory之間的存取差異，kernel提供了`ioport_map()`。
在使用`ioport_map()`之前，仍然要使用`request_region()`來allocatd。
{% sample lang="kernel 2.6" %}
```C
void *ioport_map(unsigned long port, unsigned int count);
void ioport_unmap(unsigned long port, unsigned int count);
```
把count個I/O ports mapping到I/O memory。並回傳這段address的pointer。使用ioport_unmap來解除。
{% endmethod %}
####ISA
自己去看ldd3的範例程式，這邊不贅述。