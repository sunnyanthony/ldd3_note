# Char Device Driver

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



