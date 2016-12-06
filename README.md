# LDD3的筆記跟API

LDD3內容雖然是有點舊(2005年,Linux kernel 2.6v)，不過還是有一定的參考價值。
會將需要瞭解的內容都放上來，並記錄哪些情況需要使用kernel的那些header file & struct。

這本書是Creative Commons Attribution-ShareAlike 2.0 license授權，可由以下網址取得https://lwn.net/Kernel/LDD3/  
在gitbook上的內容，是從我購買的<a href="http://www.oreilly.com.tw/product_linux.php?id=a184_errata">中文版</a>內容搭配英文版所完成的。

閱讀本書需要會以下能力，或搭配輔助：
* C language
  * pointer和struct
  * 除了學校所學，可以看看K&R跟expert c programming deep c secrets
* Operating system concept
  * 尤其是同步問題跟中斷
  * 念熟恐龍本就可以
  * 也可以從Nachos當中了解作業系統的原理，這部分可以看我另一個<a href="https://www.gitbook.com/book/sunnyanthony/operating-system-concept/details">gitbook</a>
* Uinx-like programming ＆ environment
  * system call、command 和pipe etc.
  * 念過Advanced Programming in the UNIX Environment會更好，不過只有少部分有提到這本書的內容 ( 我正在看，內容有點多，不過會複習到很多OS的概念 )
* Computer Architecture
  * 可以參考Computer Architecture A Quantitative Approach ( 這本我還沒看完 )
