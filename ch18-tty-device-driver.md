# TTY Device Driver

原本只是physical或virtual terminal，但現在變成只要是可以連接serial port，都被統稱為tty。

* tty可分為三種：  
  1. Console  
  2. Serial port  
  3. PTY \(pseudo tty\)  
     ![](/assets/螢幕快照 2017-05-19 下午4.29.50.png)

* 所有已經跟kernel register的tty，都會在 `/sys/class/tty`，若kernel知道有真實的device，則kernel會創建device link到真實的node。

  * 可看下方顯示：
    ![](/assets/螢幕快照 2017-05-19 下午5.06.50.png)

* TTY總共有三層架構，可分為tty core, tty line discipline以及tty driver。
  ![](/assets/螢幕快照 2017-05-19 下午5.43.03.png)

---

## Tiny TTY
{% method %}
TTY device driver透過`tty_driver`跟TTY core進行register跟unregister。
{% sample lang="kernel 2.6" %}
TTY的主要driver都是由tty_driver這個資料結構提供。
``` c
#include <linux/tty_driver.h>
struct tty_driver
```

{% endmethod %}






