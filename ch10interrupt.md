# Ch10.中斷處理

CPU不可能枯等Devices的events，因此OS必須要提供一種方式讓Device發生event時能夠通知CPU。這種通知方式我們稱為**interrupt。**

在user-space使用的software interrupt\( trap or exception \)是使用signal。在Linux處理interrupt的方式跟sin gal一樣，device只需要使用handler就可以在CPU收到device發出interrupt的時候，執行driver所註冊的ISR\(interrupt service routine\)。最後，ISR有個重要的特點，也就是它本身要能與其他的process同時執行，因此他必定會面對concurrent問題，以及hardware資料結構的race condition。

---

## Preparing the Parallel Port
{% method %}
這邊interrupt使用前一張提出的short模組(Parallel Port)來示範，因為不interrupt不使用hardware的話，是無法發出訊號的。
大多數device若沒有事先設定，通常是不會主動發出interrupr。然而Parallel Port可以設立port2(0x37a, 0x27a)的bit 4，可以開啟interrupt的reporting。在short.c當中，是使用`outb()`來設定此bit。  
{% sample lang="kernel 2.6" %}
```c
void outb(unsigned char value, unsigned short int port);
/*DESCRIPTION
   This family of functions is used to do low-level port input and  output.
   The  out* functions do port output, the in* functions do port input; the
   b-suffix functions are byte-width and the w-suffix functions word-width;
   the _p-suffix functions pause until the I/O completes.*/
```  
{% endmethod %}  
 
{% method %}
當interrupt功能被開啟後，parallel interface會在pin 10(the so-called ACK bit)發生了electrical signal(電壓由low到high)的時候產生了一個interrupt signal。
由此可知，引起interrupr的最簡單方式是將pin 9跟pin 10連接起來。只要data的MSB (most significant bit)也就是pin 9，只要將資料寫入到/dev/short0，就可以產生interrupr。但是寫入的資料室ASCII將無法產生interrupr，這是因為ASCII的MSB都是0。   
{% sample lang="Figure" %}
![Figure9-1](f9_1.jpg)  
{% endmethod %}  

---
## Installing an Interrupt Handler
{% method %}
System上必須要有___interrupt handler___(___interrupt service routine，ISR___)才能夠處理interrupt。若是沒有相對應的ISR，則CPU只會回傳ACK signal給device，當作處理interrupt的方式。
在某些PC上interrupt channel(___interrupt request，IRQ___)只有15或16條，所以需要小心處理避免浪費。而Linux kernel內部有一個registry of interrupt lines，用來記錄IRQ跟ISR之間的對應。若要使用特定的IRQ module的話，就必須先向kernel註冊。大部分情況底下，module會傾向多個drivers共用一組IRQ。
{% sample lang="kernel 2.6" %}
```c
#include <linux/interrupt.h>
int request_irq(unsigned int irq,
                irqreturn_t (*handler)(int, void *, struct pt_regs *),
                unsigned long flags,
                const char *dev_name,
                void *dev_id);
void free_irq(unsigned int irq, void *dev_id);
```
透過`request_irq`來註冊IRQ，並且在失敗的時候會取得負數的錯誤碼，常見的錯誤碼是 -EBUSY，這是IRQ已經被其他driver給佔用。並使用`free_irq`來歸還已註冊的IRQ。
```c
unsigned int irq
```
想要取得的interrupt編號。
```c 
irqreturn_t (*handler)(int, void *, struct pt_regs *)
```
指向要被處理的ISR function。
```
unsigned long flags
//-------------------------
SA_INTERRUPT     表示要安裝的ISR是快速型，在中段失效期間執行完ISR的工作
SA_SHIRQ         代表是否可以被不同device共享
SA_SAMPLE_RANDOM 表示interrupt對entropy pool有所貢獻(幫助產生亂數)，並且被使用在/dev/random和/dev/urandom
                 在有週期性的或是可能被攻擊的device不得設立此flag
```
關於管理中斷選項的mask。
```c
const char *dev_name
```
這字串被使用在/proc/interrupts底下，用來表示ISR。
```c
void *dev_id
```
一個識別碼，用來共享IRQ。在free_irq會使用到。  
此辨識碼可用來辨識是哪一個device發出的interrupr。若想要獨佔IRQ，可將此識別碼設為NULL，也可將它指向device struct。
{% endmethod %}  
IRQ盡量在要使用的時候才去註冊ISR，因為IRQ是限量的，若在一開始就註冊很容易閒置而浪費。所以在device第異次被啟用時才註冊IRQ，這樣可以減少佔用浪費的情況。以下是一個只有快速型的範例：
```c
if (short_irq >= 0) {
   result = request_irq(short_irq, short_interrupt,
   SA_INTERRUPT, "short", NULL);
   if (result) {
       printk(KERN_INFO "short: can't get assigned irq %i\n",
              short_irq);
       short_irq = -1;
   }
   else { /* actually enable it -- assume this *is* a parallel port */
       outb(0x10,short_base+2);
       //short_base是parell interface的I/O起始位址，寫入port2
       //寫入0x10(register2)，可以開啟回報interrupt
   }
}
```
#### The /proc Interface  
 當CPU接收到hardware interrupt時，對應的IRQ中的counter就會被累加一次。可以從/proc/interrupts當中看各個device的出中斷次數。ˋ面試書上的範例：  
root@montalcino:/bike/corbet/write/ldd3/src/short# m /proc/interrupts  
        CPU0       CPU1   
  0: 4848108        34  IO-APIC-edge timer      
  2:       0         0  XT-PIC cascade  
  8:       3         1  IO-APIC-edge rtc  
 10:    4335         1  IO-APIC-level aic7xxx  
 11:    8903         0  IO-APIC-level uhci_hcd  
 12:      49         1  IO-APIC-edge i8042  
NMI:       0         0  
LOC: 4848187   4848186  
ERR:       0  
MIS:       0  