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
在翻譯版本中提到，使用devfs跟udev可以解決。目前udev是devfs的user-space的解決方法。是使用kernel發出hotplug出法user-space的process，此process會從sysfs取得新的device的資訊，並動態建立device node。最新狀況是udev已經被整理到systemd當中，有興趣的可以參考以下連結：http://man7.org/linux/man-pages/man7/udev.7.html。
```
{% endmethod %}
{% method %}
旁邊可以看到scull是如何決定mijor number。
{% sample lang="kernel 2.6" %}
``` c
if (scull_major) {
 dev = MKDEV(scull_major, scull_minor);
 result = register_chrdev_region(dev, scull_nr_devs, "scull");
} else {
 result = alloc_chrdev_region(&dev, scull_minor, scull_nr_devs, "scull");
 scull_major = MAJOR(dev);
}
if (result < 0) {
 printk(KERN_WARNING "scull: can't get major %d\n", scull_major);
 return result;
}
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

## Some Important Data Structure
Device主要會涉及三種data structure:
* file
``` c 
struct file {
         union {
                 struct llist_node       fu_llist;
                 struct rcu_head         fu_rcuhead;
         } f_u;
         struct path             f_path;
         struct inode            *f_inode;       /* cached value */
         const struct file_operations    *f_op;
 
         /*
          * Protects f_ep_links, f_flags.
          * Must not be taken from IRQ context.
          */
         spinlock_t              f_lock;
         atomic_long_t           f_count;
         unsigned int            f_flags;
         fmode_t                 f_mode;
         struct mutex            f_pos_lock;
         loff_t                  f_pos;
         struct fown_struct      f_owner;
         const struct cred       *f_cred;
         struct file_ra_state    f_ra;
 
         u64                     f_version;
 #ifdef CONFIG_SECURITY
         void                    *f_security;
 #endif
         /* needed for tty driver, and maybe others */
         void                    *private_data;
 
 #ifdef CONFIG_EPOLL
         /* Used by fs/eventpoll.c to link all the hooks to this file */
         struct list_head        f_ep_links;
         struct list_head        f_tfile_llink;
 #endif /* #ifdef CONFIG_EPOLL */
         struct address_space    *f_mapping;
 } __attribute__((aligned(4)));  /* lest something weird decides that 2 is OK */
```
* file_operations
``` c 
struct file_operations {
         struct module *owner;
         loff_t (*llseek) (struct file *, loff_t, int);
         ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
         ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
         ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
         ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
         int (*iterate) (struct file *, struct dir_context *);
         int (*iterate_shared) (struct file *, struct dir_context *);
         unsigned int (*poll) (struct file *, struct poll_table_struct *);
         long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
         long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
         int (*mmap) (struct file *, struct vm_area_struct *);
         int (*open) (struct inode *, struct file *);
         int (*flush) (struct file *, fl_owner_t id);
         int (*release) (struct inode *, struct file *);
         int (*fsync) (struct file *, loff_t, loff_t, int datasync);
         int (*fasync) (int, struct file *, int);
         int (*lock) (struct file *, int, struct file_lock *);
         ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
         unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
         int (*check_flags)(int);
         int (*flock) (struct file *, int, struct file_lock *);
         ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
         ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
         int (*setlease)(struct file *, long, struct file_lock **, void **);
         long (*fallocate)(struct file *file, int mode, loff_t offset,
                           loff_t len);
         void (*show_fdinfo)(struct seq_file *m, struct file *f);
 #ifndef CONFIG_MMU
         unsigned (*mmap_capabilities)(struct file *);
 #endif
         ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
                         loff_t, size_t, unsigned int);
         int (*clone_file_range)(struct file *, loff_t, struct file *, loff_t,
                         u64);
         ssize_t (*dedupe_file_range)(struct file *, u64, u64, struct file *,
                         u64);
 };
```
* inode
``` c 
struct inode {
         umode_t                 i_mode;
         unsigned short          i_opflags;
         kuid_t                  i_uid;
         kgid_t                  i_gid;
         unsigned int            i_flags;
 
 #ifdef CONFIG_FS_POSIX_ACL
         struct posix_acl        *i_acl;
         struct posix_acl        *i_default_acl;
 #endif
 
         const struct inode_operations   *i_op;
         struct super_block      *i_sb;
         struct address_space    *i_mapping;
 
 #ifdef CONFIG_SECURITY
         void                    *i_security;
 #endif
 
         /* Stat data, not accessed from path walking */
         unsigned long           i_ino;
         /*
          * Filesystems may only read i_nlink directly.  They shall use the
       * following functions for modification:
          *
          *    (set|clear|inc|drop)_nlink
          *    inode_(inc|dec)_link_count
          */
         union {
                 const unsigned int i_nlink;
                 unsigned int __i_nlink;
         };
         dev_t                   i_rdev;
         loff_t                  i_size;
         struct timespec         i_atime;
         struct timespec         i_mtime;
         struct timespec         i_ctime;
         spinlock_t              i_lock; /* i_blocks, i_bytes, maybe i_size */
         unsigned short          i_bytes;
         unsigned int            i_blkbits;
         blkcnt_t                i_blocks;
 
 #ifdef __NEED_I_SIZE_ORDERED
         seqcount_t              i_size_seqcount;
 #endif
 
         /* Misc */
         unsigned long           i_state;
         struct rw_semaphore     i_rwsem;
 
         unsigned long           dirtied_when;   /* jiffies of first dirtying */
         unsigned long           dirtied_time_when;
 
         struct hlist_node       i_hash;
         struct list_head        i_io_list;      /* backing dev IO list */
 #ifdef CONFIG_CGROUP_WRITEBACK
         struct bdi_writeback    *i_wb;          /* the associated cgroup wb */
 
         /* foreign inode detection, see wbc_detach_inode() */
         int                     i_wb_frn_winner;
         u16                     i_wb_frn_avg_time;
         u16                     i_wb_frn_history;
 #endif
         struct list_head        i_lru;          /* inode LRU list */
         struct list_head        i_sb_list;
         struct list_head        i_wb_list;      /* backing dev writeback list */
         union {
                 struct hlist_head       i_dentry;
                 struct rcu_head         i_rcu;
         };
         u64                     i_version;
         atomic_t                i_count;
         atomic_t                i_dio_count;
         atomic_t                i_writecount;
 #ifdef CONFIG_IMA
         atomic_t                i_readcount; /* struct files open RO */
 #endif
         const struct file_operations    *i_fop; /* former ->i_op->default_file_ops */
         struct file_lock_context        *i_flctx;
         struct address_space    i_data;
         struct list_head        i_devices;
         union {
                 struct pipe_inode_info  *i_pipe;
                 struct block_device     *i_bdev;
                 struct cdev             *i_cdev;
                 char                    *i_link;
                 unsigned                i_dir_seq;
         };
 
         __u32                   i_generation;
 
 #ifdef CONFIG_FSNOTIFY
         __u32                   i_fsnotify_mask; /* all events this inode cares about */
         struct hlist_head       i_fsnotify_marks;
 #endif
 
 #if IS_ENABLED(CONFIG_FS_ENCRYPTION)
         struct fscrypt_info     *i_crypt_info;
 #endif
 
         void                    *i_private; /* fs or device private pointer */
 };
```

scull divice driver指實作最重要的methods，file_operations初始化如下:
``` c 
struct file_operations scull_fops = {
    .owner =    THIS_MODULE,
    .llseek =   scull_llseek,
    .read =     scull_read,
    .write =    scull_write,
    .unlocked_ioctl =    scull_ioctl,
    .open =     scull_open,
    .release =  scull_release,
};
```

## Char Device Registration
------

{% method %}
Kernel主要是使用struct cdev代表char device。若要讓kernel操作device1，就必須註冊一個以上的cdev。並使用cdev_alloc()配置，並將cdev->ops指向f_ops。
{% sample lang="kernel 2.6" %}

``` c
#include <linux/cdev.h>
struct cdev *my_cdev=cdev_alloc();
my_cdev->ops=&my_fops;
```
{% sample lang="kernel 4.\*" %}
``` c

```
{% endmethod %}

{% method %}
Scull需要將cdev放入設計的特殊data structure，因此在這種特殊情況，需要使用cdev_init()來得到cdev。
{% sample lang="kernel 2.6" %}

``` c
#include <linux/cdev.h>
void cdev_init(struct cdev *cdev, struct file_operations *fops);
```
{% sample lang="kernel 4.\*" %}
``` c

```
{% endmethod %}
這邊的cdev->owner跟file_operations一樣被設定為`THIS_MODULE`。
{% method %}
最後一個步驟是使用cdev_add()將cdev加入kernel。
{% sample lang="kernel 2.6" %}
``` c
#include <linux/cdev.h>
int cdev_add(struct cdev *dev, dev_t num, unsigned int count)
```
**deev**是設定好的cdev structure，**num**是第一個device number，**count*是device number的總數。
{% sample lang="kernel 4.\*" %}
``` c

```
{% endmethod %}


