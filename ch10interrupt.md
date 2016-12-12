# Ch10.中斷處理

CPU不可能枯等Devices的events，因此OS必須要提供一種方式讓Device發生event時能夠通知CPU。這種通知方式我們稱為**interrupt。**

在user-space使用的software interrupt\( trap or exception \)是使用signal。在Linux處理interrupt的方式跟sin gal一樣，device只需要使用handler就可以在CPU收到device發出interrupt的時候，執行driver所註冊的ISR\(interrupt service routine\)。最後，ISR有個重要的特點，也就是它本身要能與其他的process同時執行，因此他必定會面對concurrent問題，以及hardware資料結構的race condition。

---

## Preparing the Parallel Port



