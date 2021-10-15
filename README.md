# 1. pola linux_driver step by step
    In there，you can complete the linux driver step by step.
# 2. Use input with stream video
    "./app /dev/test 2021_09_30/pola_layer_info.bin  2021_09_30/pola_total_tile_info.bin  2021_09_30/pola_total_weight.bin udp://140.117.176.62:5000  2021_09_30/pola_Output_Offset.txt"
# 3. Use input with picture
    "./app /dev/test 2021_09_30/pola_layer_info.bin  2021_09_30/pola_total_tile_info.bin  2021_09_30/pola_total_weight.bin ./../../ship.jpg  2021_09_30/pola_Output_Offset.txt"
# 4. mmap func
    記憶體映射函數 mmap 的使用方法

    該函數主要用途有三個：
    1、將一個普通文件映射到核心中，通常在需要對文件進行頻繁讀寫時使用，這樣用核心讀寫取代I/O讀寫，以獲得較高的性能；
    2、將特殊文件進行匿名核心映射，可以為關聯進程提供共享核心空間；
    3、為無關聯的進程提供共享核心空間，一般也是將一個普通文件映射到核心中。

    函數：void *mmap(void *start,size_t length, int prot, int flags, int fd, off_t offsize);

    參數start：指向欲映射的核心起始位址，通常設為NULL，代表讓系統自動選定位址，核心會自己在進程位址空間中選擇合適的位址建立映射。
    映射成功後返回該位址。如果不是NULL，則給核心一個提示，應該從什麼位址開始映射，核心會選擇start之上的某個合適的位址開始映射。
    建立映射後，真正的映射位址通過返回值可以得到。

    參數length：代表映射的大小。將文件的多大長度映射到記憶體。

    參數prot：映射區域的保護方式。可以為以下幾種方式的組合：
    PROT_EXEC 映射區域可被執行
    PROT_READ 映射區域可被讀取
    PROT_WRITE 映射區域可被寫入
    PROT_NONE 映射區域不能存取

    參數flags：影響映射區域的各種特性。在調用mmap()時必須要指定MAP_SHARED 或MAP_PRIVATE。
    MAP_FIXED 如果參數start所指的位址無法成功建立映射時，則放棄映射，不對位址做修正。通常不鼓勵用此旗標。
    MAP_SHARED 允許其他映射該文件的行程共享，對映射區域的寫入數據會複製回文件。
    MAP_PRIVATE 不允許其他映射該文件的行程共享，對映射區域的寫入操作會產生一個映射的複製(copy-on-write)，對此區域所做的修改不會寫回原文件。
    MAP_ANONYMOUS 建立匿名映射。此時會忽略參數fd，不涉及文件，而且映射區域無法和其他進程共享。
    MAP_DENYWRITE 只允許對映射區域的寫入操作，其他對文件直接寫入的操作將會被拒絕。
    MAP_LOCKED 將映射區域鎖定住，這表示該區域不會被置換(swap)。

    參數fd：由open返回的文件描述符，代表要映射到核心中的文件。如果使用匿名核心映射時，即flags中設置了MAP_ANONYMOUS，fd設為-1。
    有些系統不支持匿名核心映射，則可以使用fopen打開/dev/zero文件，然後對該文件進行映射，可以同樣達到匿名核心映射的效果。

    參數offset：從文件映射開始處的偏移量，通常為0，代表從文件最前方開始映射。
    offset必須是分頁大小的整數倍(在32位體系統結構上通常是4K)。

    返回值：
    若映射成功則返回映射區的核心起始位址，否則返回MAP_FAILED(-1)，錯誤原因存於errno 中。
    