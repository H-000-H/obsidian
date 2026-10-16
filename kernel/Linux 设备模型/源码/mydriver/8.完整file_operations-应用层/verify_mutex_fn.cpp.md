<!-- 源：/home/h-000-h/project/mydriver/7/verify_mutex_fn.cpp -->

```cpp
#include <cstdio>
#include <thread>
#include <atomic>
#include <chrono>
#include <fcntl.h>
#include <unistd.h>
#include <vector>
#include <iostream>

std::atomic<bool> stop{false};
std::atomic<int> tears{0};

/*
写线程用 fd1，只写两种纯色消息："AAAA...\n" 和 "BBBB...\n" 交替
读线程用 fd2（独立 fd，offset 互不干扰），每次 lseek(0) 后整段读回
判据：读到的内容必须纯 A 或纯 B；一旦出现 "AABBAABB" 这种混合，就是 buf 被写线程改花了的铁证——撕裂计数
*/

void writer(int fd)
{
    const char msg[2][18]= {"AAAAAAAAAAAAAAAA\n", "BBBBBBBBBBBBBBBB\n"};
    int i = 0;
    while(!stop)
    {
        if (write(fd, msg[i], 17) != 17) 
        { 
            perror("write"); break; 
        }
        i^=1;
    }
}

void reader(int fd)
{
    std::vector<char>buf(128);
    while(!stop)
    {
        lseek(fd,0,SEEK_SET);
        ssize_t n = read(fd,buf.data(),buf.size()-1);
        if(n<0)
            continue;
        char first = buf[0];
        for(ssize_t j=1;j<n;j++)
        {
             if (buf[j] != first && buf[j] != '\n') 
             { 
                tears++; 
                break; 
             }
        }
    }
}

int main()
{
    int fd_w = open("/dev/hello70", O_RDWR);
    int fd_r = open("/dev/hello70", O_RDWR);//独立 offset 与写分开
    if (fd_w < 0 || fd_r < 0) 
    {
        perror("open");       
        return 1;
    }

    std::thread w(writer,fd_w), r(reader,fd_r);
    std::this_thread::sleep_for(std::chrono::seconds(100));
    stop = true;
    w.join(); r.join();
    std::cout << "撕裂次数: " << tears << "（有 mutex 应为 0）\n";
    return 0;
}
```
