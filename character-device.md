# Char Device Driver

## Major and Minor Numbers

---

Character device的access是透過file system上的name（special file、device file or node）。Device number是由major+minor所構成，可參考以下範例。

```
ls -l /dev/ 
total 0
                                     major  minor
crw-------  1 root     wheel           17,   1  4 28 17:18 afsc_type5
crw-------  1 root     wheel           10,   0  4 28 10:02 auditpipe
crw-r--r--  1 root     wheel            9,   3  4 28 10:02 auditsessions
crw-------  1 root     wheel           21,   0  4 28 10:02 autofs
crw-------  1 root     wheel           34,   0  4 28 10:02 autofs_control
crw-rw-rw-  1 root     wheel           33,   0  4 28 10:02 autofs_homedirmounter
crw-rw-rw-  1 root     wheel           31,   0  4 28 10:02 autofs_notrigger
crw-rw-rw-  1 root     wheel           22,   6  4 28 10:02 autofs_nowait
```

Major在傳統上是表示device所配合的driver。現在kernel允許多個driver共用一個major number，不過大部分還是會諄造船同上的一對一的編號。然而minor number是讓kernel用來表示哪個device的referred to。

### The Internal Representation of Device Numbers

{% method %}

Device number在kernel是以`dev_t`來表示，但我們不應該假設device number的tpye，因此應該要借用linux提供的macro，MAJOR\(\)跟MINOR\(\)，透過macro可避免未來kernel變動時造成移植上的問題。   
若有需要將minor跟mijor合併，請使用MKDEV()這個macro。

{% sample lang="kernel 2.6" %}

``` c
#include <linux/types.h>
//    major       minor
// |--12-bit--|---20-bit---|

#include <linux/kdev_t.h>
MAJOR(dev_t dev);
MINOR(dev_t dev);
MKDEV(int major, int minor);
```
{% sample lang="kernel 4.\*" %}
``` c
#include <linux/types.h>
//    major       minor
// |--12-bit--|---20-bit---|

#include <linux/kdev_t.h>
MAJOR(dev_t dev);
MINOR(dev_t dev);
MKDEV(int major, int minor);
```
{% endmethod %}

### Allocating and Freeing Device Numbers
{% method %}
Device driver第一件事是取得一個或是多個device number。使用`register_chrdev_region`這個function。   
可在`/proc/devices/`跟`sysfs`下出現。  
不過kernel開發上還是建議使用動態配置，這樣可以減少device跟number之間的固定關係。可改用`alloc_chrdev_region`這個function。最後不使用driver時，透過`unregister_chrdev_region`釋放device number。通常是在unload fuction裡面。
{% sample lang="kernel 2.6" %}

``` c
#include <linux/fs.h>
int register_chrdev_region(dev_t first, unsigned int count,
 char *name);
```
**first**是你所想要配置device number的region的起始位子。**count**是想要申請的連續編號總數。**name**則是你最終會獲取的device name。
``` c
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor,
 unsigned int count, char *name);
```
**dev**是當配置成功時，dev會持有region的第一個device number。**firstminor**是想要申請的第一個minor number(通常是0)。**count**是想要申請的連續編號總數。**name**則是你最終會獲取的device name。
``` c
void unregister_chrdev_region(dev_t first, unsigned int count);
```
{% sample lang="kernel 4.\*" %}
``` c

```
{% endmethod %}
### Dynamic Allocation of Major Numbers
{% method %}
目前常見的device都已經被記錄在`Documentation/devices.txt`當中。因此建議使用`alloc_chrdev_region`來分配mijor nember。但有一缺點是，無法事先建立/dev/node，這是因為動態配置無法保證每次的number都相同。
{% sample lang="kernel 2.6" %}
``` c 
在翻譯版本中提到，使用devfs跟udev可以解決。目前udev是devfs的user-space的解決方法。是使用kernel發出hotplug出法user-space的process，此process會從sysfs取得新的device的資訊，並動態建立device node。
```

{% endmethod %}

## The Design of char device

---

首先，在我們需要定義driver對於user-space' program提供哪些capabilities（mechanism）。記住，device其實只是一塊memory。

這邊提出scull這個device driver temperate，使用抽象及封裝的技巧來方便開發。包含以下類型：

* scull\#

  * 使用memory所構成
  * Global：在被開啟多次的情況下，fd\(file descriptor\)可共享device所擁有的data。
  * Persistent ：就算closed或reopen，data也不會流
  * 可使用cp、cat、I/O redirection操作

* scullpipe\#

  * 使用FIFO的device
  * 類似pipe。一個process寫入，可由另一個process讀取
  * 這邊不使用hardware inperrutp來實作blocking/nonblocking的存取

* scullsingle

  * 一次只能有一個process執行open\(\)

* scullpriv

  * 每個virtual console都可擁有私人的scullpriv
  * 各個virtual console都會得到不同的memory area

* scullid

  * 若device已被佔用，試圖執行open則會發出Device Busy

* scullwuid
  * 若device已被佔用，試圖執行open則會blocking到device被釋放為止

這些driver在mechanism上都完全一樣\(abstract\)，只不過使用不同的policy。

