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
``` clike=
#include <linux/tty_driver.h>
struct tty_driver
```
{% endmethod %}

{% method %}
創建一個tty device時，需呼叫`alloc_tty_driver`
{% sample lang="kernel 2.6" %}
``` clike=
#include <linux/tty_driver.h>
static inline struct tty_driver * alloc_tty_driver(unsigned int lines)
```
lines是要提供tty device driver的device number。
{% endmethod %}


{% method %}
取得了`tty_driver`後，需要設定適當的初始化。可以先看下方的structure再參考右邊初始化需做哪些設定。
* 下方的是tty_driver結構，設定後須透過`extern void tty_set_operations(struct tty_driver *driver,const struct tty_operations *op);`完成初始化。

``` C
struct tty_driver {
	int	magic;		/* magic number for this structure */
	struct kref kref;	/* Reference management */
	struct cdev **cdevs;
	struct module	*owner;
	const char	*driver_name;
	const char	*name;
	int	name_base;	/* offset of printed name */
	int	major;		/* major device number */
	int	minor_start;	/* start of minor device number */
	unsigned int	num;	/* number of devices allocated */
	short	type;		/* type of tty driver */
	short	subtype;	/* subtype of tty driver */
	struct ktermios init_termios; /* Initial termios */
	unsigned long	flags;		/* tty driver flags */
	struct proc_dir_entry *proc_entry; /* /proc fs entry */
	struct tty_driver *other; /* only used for the PTY driver */

	/*
	 * Pointer to the tty data structures
	 */
	struct tty_struct **ttys;
	struct tty_port **ports;
	struct ktermios **termios;
	void *driver_state;

	/*
	 * Driver methods
	 */

	const struct tty_operations *ops;
	struct list_head tty_drivers;
};
```
* 下方的是`tty_operations`，設定完後會被`tty_set_operations`把`tty_driver->ops=tty_operations;`指派。
``` C
struct tty_operations {
	struct tty_struct * (*lookup)(struct tty_driver *driver,
			struct file *filp, int idx);
	int  (*install)(struct tty_driver *driver, struct tty_struct *tty);
	void (*remove)(struct tty_driver *driver, struct tty_struct *tty);
	int  (*open)(struct tty_struct * tty, struct file * filp);
	void (*close)(struct tty_struct * tty, struct file * filp);
	void (*shutdown)(struct tty_struct *tty);
	void (*cleanup)(struct tty_struct *tty);
	int  (*write)(struct tty_struct * tty,
		      const unsigned char *buf, int count);
	int  (*put_char)(struct tty_struct *tty, unsigned char ch);
	void (*flush_chars)(struct tty_struct *tty);
	int  (*write_room)(struct tty_struct *tty);
	int  (*chars_in_buffer)(struct tty_struct *tty);
	int  (*ioctl)(struct tty_struct *tty,
		    unsigned int cmd, unsigned long arg);
	long (*compat_ioctl)(struct tty_struct *tty,
			     unsigned int cmd, unsigned long arg);
	void (*set_termios)(struct tty_struct *tty, struct ktermios * old);
	void (*throttle)(struct tty_struct * tty);
	void (*unthrottle)(struct tty_struct * tty);
	void (*stop)(struct tty_struct *tty);
	void (*start)(struct tty_struct *tty);
	void (*hangup)(struct tty_struct *tty);
	int (*break_ctl)(struct tty_struct *tty, int state);
	void (*flush_buffer)(struct tty_struct *tty);
	void (*set_ldisc)(struct tty_struct *tty);
	void (*wait_until_sent)(struct tty_struct *tty, int timeout);
	void (*send_xchar)(struct tty_struct *tty, char ch);
	int (*tiocmget)(struct tty_struct *tty);
	int (*tiocmset)(struct tty_struct *tty,
			unsigned int set, unsigned int clear);
	int (*resize)(struct tty_struct *tty, struct winsize *ws);
	int (*set_termiox)(struct tty_struct *tty, struct termiox *tnew);
	int (*get_icount)(struct tty_struct *tty,
				struct serial_icounter_struct *icount);
#ifdef CONFIG_CONSOLE_POLL
	int (*poll_init)(struct tty_driver *driver, int line, char *options);
	int (*poll_get_char)(struct tty_driver *driver, int line);
	void (*poll_put_char)(struct tty_driver *driver, int line, char ch);
#endif
	const struct file_operations *proc_fops;
};
```
{% sample lang="kernel 2.6" %}
``` C
#include <linux/tty_driver.h>
static struct tty_operations serial_ops = {
 .open = tiny_open,
 .close = tiny_close,
 .write = tiny_write,
 .write_room = tiny_write_room,
 .set_termios = tiny_set_termios,
};
...
 /* initialize the tty driver */
 tiny_tty_driver->owner = THIS_MODULE;
 tiny_tty_driver->driver_name = "tiny_tty";
 tiny_tty_driver->name = "ttty";
 tiny_tty_driver->devfs_name = "tts/ttty%d";
 tiny_tty_driver->major = TINY_TTY_MAJOR,
 tiny_tty_driver->type = TTY_DRIVER_TYPE_SERIAL,
 tiny_tty_driver->subtype = SERIAL_TYPE_NORMAL,
 tiny_tty_driver->flags = TTY_DRIVER_REAL_RAW | TTY_DRIVER_NO_DEVFS,
 tiny_tty_driver->init_termios = tty_std_termios;
 tiny_tty_driver->init_termios.c_cflag = B9600 | CS8 | CREAD | HUPCL | CLOCAL;
 tty_set_operations(tiny_tty_driver, &serial_ops);
```
lines是要提供tty device driver的device number。
{% endmethod %}












