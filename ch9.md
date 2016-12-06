# Ch9.硬體的操作

## I/O port and I/O memory
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
```C
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

{% sample lang="kernel 4.*" %}

{% endmethod %}

順帶一提，spin_lock, atomic_t 等同步處理也有memory barrier作用。並有些平台容許一個動作就給予一個barrier，可使用以下Macro：
```C
#define set_mb(var, value) do {var = value; mb();} while 0
//ps. 只有少數平台有set_rmb
```
`do ... while`是讓Marco展開後可以在不同環境下正常運作的慣例。有時候展開Marco會跟前後文不小心結合或讓if else判斷跟預期不同。

## I/O port 用法

對於device driver而言，存取I/O port為重要的議題。

#### 配置I/O port
{% method %}
Allocation I/O port是為了獨佔I/O port的使用，kernel提供了一組allocation interface來索取所需的I/O
{% sample lang="kernel 2.6" %}
```C
#include <linux/ioport.h>
struct resource *request_region(unsigned long first, unsigned long n, const char *name);
```
{% endmethod %}
