<!-- 源：/home/h-000-h/project/mydriver/7/main.cpp -->

```cpp
#include <algorithm>
#include <cstdio>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <stdexcept>
#include <string>
#include <vector>
#include "main.hpp"
#include <iostream>
class HelloDev
{
public:
    explicit HelloDev(const char*path): fd_(::open(path,O_RDWR))
    {
        if (fd_<0) 
        {
            throw std::runtime_error(std::string("open")+path+"失败");
        }
    }
    ~HelloDev()
    {
        if(fd_>=0)::close(fd_);
    }
    HelloDev(const HelloDev&)=delete;
    HelloDev& operator=(const HelloDev&) = delete;

    ssize_t write(const std::string&src)
    {
        return ::write(fd_,src.data(),src.size());
    }

    ssize_t read(std::vector<char>&buf)
    {
        ssize_t n =::read(fd_,buf.data(),buf.size());
        if(n>0)
        {
            buf[n]='\0';
        }
        return n;
    }

    void ioctl()
    {
        if (::ioctl(fd_, HELLO_IOCRESET) < 0)
            throw std::runtime_error("ioctl 失败");
    }

    int get_fd()
    {
        return fd_;
    }
private:
    int fd_;
};

int main()
{
    try 
    {
        HelloDev dev("/dev/hello70");
        dev.write("hello from cpp");       

        std::vector<char> buf(128);
        ::lseek(dev.get_fd(), 0, SEEK_SET);     /* offset 还是要手动管 */
        ssize_t n = dev.read(buf);
        std::cout << "读回 " << n << " 字节: " << buf.data() << std::endl;

        dev.ioctl();
        n = dev.read(buf);
        std::cout << "reset 后读到 " << n << " 字节（应为 0）" << std::endl;
    } 
    catch (const std::exception& e) 
    {
        std::cerr << "错误: " << e.what() << std::endl;
        return 1;
    }
    return 0;
}
```
