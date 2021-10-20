# 1. pola linux_driver step by step
    In there，you can complete the linux driver step by step.
# 2. Use input with stream video
    "./app /dev/test 2021_09_30/pola_layer_info.bin  2021_09_30/pola_total_tile_info.bin  2021_09_30/pola_total_weight.bin udp://140.117.176.62:5000  2021_09_30/pola_Output_Offset.txt"
# 3. Use input with picture
    "./app /dev/test 2021_09_30/pola_layer_info.bin  2021_09_30/pola_total_tile_info.bin  2021_09_30/pola_total_weight.bin ./../../ship.jpg  2021_09_30/pola_Output_Offset.txt"
# 4. mmap()
```c++
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
```
# 5. check_have_final()
```c++
const size_t HW_SHIFT = 1;                                                                                  *主要要算總共要傳輸幾次給硬體，如果是ZCU102一次可以128bit，ZEDBOARD一次64bit //Bus data width 128 bit = 1,64 bit = 2
const uint32_t offset = ((uint32_t)(layer.data[5]&0b11111111)+((layer.data[6]&0b00001111)<<8))>>HW_SHIFT;   *在layer.data一組是8bit，這邊要取tile_info_number(12bit)，pola_parser說明書裡面會有，然後在除以上傳輸量就是*幾個
const uint32_t *tmp = (uint32_t *)((char *)baseaddr + layer.get_tile_begin_addr());                         *取得你最初的位置，還有你的tile_addr放的地方*

return (tmp[offset*8+4]&0b0100)>>2;                                                                         *這邊要注意，以第一個layer舉例你一共有128個tile_info，offset會是127，*
                                                                                                            *因為AR的關係，但實質上有128個資訊，所以你會是offset*8，因為tmp是一個uint32_t的資料結構，*
                                                                                                            *所以一個tile_info 256bit，一組會有8個那你就可以推出127*8這麼多條，*
                                                                                                            *那為啥還要加4呢，這邊要看成4*32，那就是你現在推到128條的第128bit，*
                                                                                                            *那&100的意思就是*
Is_last_channel								1	[128]
have_accumulate								1	[129]
is_final_tile								1	[130]
這三個bit，剛好是最後一個tile_info的is_final_tile，
如果是回傳1正確結束。
```
# 6. init_addr()
```c++
set_tile_begin_addr(get_tile_begin_addr()+base_addr);                                                       *就單純設定*
```
# 7. get_tile_begin_addr()
```c++
uint32_t tmp = 0;                                                                                           *因為已知道Tile_Info_Addr故透過tmp set uint32_t [52:83]
for(int i = 0; i < 4; i++){                                                                                 *這邊的i 和 j 只是為了跳去這邊的bit拿資料沒有別的意思
    for(int j = 0 ; j < 2 ; j++){
        if(j%2 == 0)                                                                                        *因為在52bit剛好是我們的資料的一半，一個data這邊是8bit所以要一半一半前後抓
            tmp |= (((data[i+6+j]&0b11110000)>>4) << (i*8));                                                *抓完之後要把資料左移(前半部)才可以知道自己的資料再合併
        else
            tmp |= (((data[i+6+j]&0b00001111)<<4) << (i*8));
	    //std::cout << "data[" << i+6+j << "]" << std::hex << tmp << std::endl;
    }
}
return tmp;
```
# 8. set_tile_begin_addr()
```c++
data[10] |= (addr>>(28))&0xff;                                                                              *這邊主要是把你get_tile_begin_addr取到的位置 + base_addr之後再塞回去設定*
data[9] |= (addr>>(20))&0xff;                                                                               *set_tile_begin_addr(get_tile_begin_addr()+base_addr);這邊可以看到緣由*
data[8] |= (addr>>(12))&0xff;
data[7] |= (addr>>(4))&0xff;
data[6] |= ((addr<<(4))&0xff);
```
# 9. parse_file()
```c++
//---------------------data type--------------------
struct layer_info
{
    uint8_t data[20];
    uint32_t get_tile_begin_addr() const;
    void run_inference(const int fd);
    void load_layer_info(const void* src);
    void set_tile_begin_addr(uint32_t addr);
    void init_addr(uint32_t base_addr);
    void get_layer_info();
};
//--------------------------------------------------
int fd = open(filename.c_str(), O_CREAT | O_RDWR | O_SYNC, S_IRUSR | S_IWUSR);                              //開啟檔案
if(fd < 0)
    throw std::invalid_argument("Can not open layer bin file.");
    
std::vector<layer_info> tmp;                                                                                
int file_size = (int) fsize(filename.c_str());                                                              //file size

 
if(file_size%20 != 0){                                                                                      //160 bit
    throw "Layer bin file format error.";
}

tmp.resize(file_size/20);
   
void* map_memory = mmap(0, file_size, PROT_READ, MAP_SHARED, fd, 0);                                        //打開檔案把記憶體起始點，大小打給指標map_memory


memcpy(tmp[0].data, map_memory, file_size);                                                                 //parameter 1 這邊請想成，tmp[0].data->這個資料型態最起始點記憶體位置
                                                                                                            //parameter 2 map_memory這個是上面指標指到要複製的起始點位置
                                                                                                            //parameter 3 需要複製過去的大小
munmap(map_memory, file_size);                                                                              //我複製過去之後當然可以解除拉，所以我需要munmap
/*  
    for(int x = 0; x < 65 ; x+=5){
    std::cout << std::hex << fmt::format("{:08x} ",tmp_test[x+4]);
	std::cout << std::hex << fmt::format("{:08x} ",tmp_test[x+3]);
	std::cout << std::hex << fmt::format("{:08x} ",tmp_test[x+2]);
	std::cout << std::hex << fmt::format("{:08x} ",tmp_test[x+1]);
	std::cout << std::hex << fmt::format("{:08x} ",tmp_test[x]);
    
	std::cout << std::endl;
    }
	std::cout << std::endl;
*/
close(fd);
    
return tmp;
```
# 10. load_inst_data()
```c++
//---------------------data type--------------------                                                        //for tile information
struct inst                                                                                                 //256bit
{                                                                                                           //AXI協定
uint8_t data[32];
void load_data(const void* data);
void set_out_no_max_addr(uint32_t addr);
void set_out_addr(uint32_t addr);
void set_weight_addr(uint32_t addr);
void set_in_addr(uint32_t addr);
uint32_t get_out_no_max_addr();
uint32_t get_out_addr();
uint32_t get_weight_addr();
uint32_t get_in_addr();
};
//-------------------------------------------------
int fd;                                                                                                     
std::vector<inst> tmp;
static_assert(std::is_pod_v<inst>);
static_assert(sizeof(inst) == 32);

fd = open(filename.c_str(), O_CREAT | O_RDWR | O_SYNC, S_IRUSR | S_IWUSR);
if(fd < 0){
    throw std::invalid_argument("Can not open tile info file.");
}

const size_t file_size = (int)fsize(filename.c_str());
void* map_memory = mmap(0, file_size, PROT_READ, MAP_SHARED, fd, 0);
tmp.resize(file_size/sizeof(inst));

//std::cout << "inst = " << (uint32_t)(tmp[0].data[0]&0b11111111) << std::endl;
    
assert(map_memory != nullptr);
assert(file_size%sizeof(inst) == 0);
memcpy(tmp[0].data, map_memory, file_size);
munmap(map_memory, file_size);
close(fd);
return tmp;
```
# 11. run_init()
```c++
std::vector<inst> tile_info = load_inst_data(tile_info_file);
std::vector<layer_info> layer =  parse_file(layer_info_file);
const uint32_t tile_info_offset =  std::ceil(size_in_byte(tile_info)/(double)PAGE_SIZE)*PAGE_SIZE;

const uint32_t weight_offset = load_weight(dst+tile_info_offset, weight);

const uint32_t input_addr  = std::ceil((phy_addr+tile_info_offset+weight_offset)/(double)PAGE_SIZE)*PAGE_SIZE;

const uint32_t input_offset = input_addr - phy_addr;

const uint32_t weight_addr = phy_addr+tile_info_offset;
```