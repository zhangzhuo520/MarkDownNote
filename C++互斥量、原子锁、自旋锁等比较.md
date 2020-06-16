#C++互斥量、原子锁、自旋锁等比较
-------------
现象：

（1）单线程无锁速度最快，但应用场合受限；

（2）多线程无锁速度第二快，但结果不对，未保护临界代码段；

（3）多线程原子锁第三快，且结果正确；

（4）多线程互斥量较慢，慢与原子锁近10倍，结果正确；

（5）多线程自旋锁最慢，慢与原子锁30倍，结果正确。

结论：**原子锁**速度最快，**互斥量**和**自旋锁**都用保护多线程共享资源。
**自旋锁**是一种非阻塞锁，也就是说，如果某线程需要获取自旋锁，但该锁已经被其他线程占用时，该线程不会被挂起，而是在不断的消耗CPU的时间，不停的试图获取自旋锁。**互斥量**是阻塞锁，当某线程无法获取互斥量时，该线程会被直接挂起，该线程不再消耗CPU时间，当其他线程释放互斥量后，操作系统会激活那个被挂起的线程，让其投入运行。在多处理器环境中对持有锁时间较短的程序来说使用自旋锁代替一般的互斥锁往往能提高程序的性能，但是本代码无该效果。
```
#include <iostream>
#include <atomic>
#include <mutex>
#include <thread>
#include <vector>
 
class spin_mutex {
    std::atomic<bool> flag = ATOMIC_VAR_INIT(false);
public:
    spin_mutex() = default;
    spin_mutex(const spin_mutex&) = delete;
    spin_mutex& operator= (const spin_mutex&) = delete;
    void lock() {
        bool expected = false;
        while (!flag.compare_exchange_strong(expected, true))
            expected = false;
    }
    void unlock() {
        flag.store(false);
    }
};
 
long size = 1000000;
long total = 0;
std::atomic_long total2(0);
std::mutex m;
spin_mutex lock;
 
void thread_click()
{
    for (int i = 0; i < size; ++i)
    {
        ++total;
    }
}
 
void mutex_click()
{
    for (int i = 0; i < size; ++i)
    {
        m.lock();
        ++total;
        m.unlock();
    }
}
 
void atomic_click()
{
    for (int i = 0; i < size; ++i)
    {
        ++total2;
    }
}
 
 
void spinlock_click()
{
    for (int i = 0; i < size; ++i)
    {
        lock.lock();
        ++total;
        lock.unlock();
    }
}
 
int main()
{
    int thnum = 100;
    std::vector<std::thread> threads(thnum);
    clock_t start, end;
 
    total = 0;
    start = clock();
    for (int i = 0; i < size * thnum; i++) {
        ++total;
    }
    end = clock();
    std::cout << "single thread result: " << total << std::endl;
    std::cout << "single thread time: " << end - start << std::endl;
 
    total = 0;
    start = clock();
    for (int i = 0; i < thnum; ++i) {
        threads[i] = std::thread(thread_click);
    }
    for (int i = 0; i < thnum; ++i) {
        threads[i].join();
    }
    end = clock();
    std::cout << "multi thread no mutex result: " << total << std::endl;
    std::cout << "multi thread no mutex time: " << end - start << std::endl;
 
    total = 0;
    start = clock();
    for (int i = 0; i < thnum; ++i) {
        threads[i] = std::thread(atomic_click);
    }
    for (int i = 0; i < thnum; ++i) {
        threads[i].join();
    }
    end = clock();
    std::cout << "multi thread atomic result: " << total2 << std::endl;
    std::cout << "multi thread atomic time: " << end - start << std::endl;
 
    total = 0;
    start = clock();
    for (int i = 0; i < thnum; ++i) {
        threads[i] = std::thread(mutex_click);
    }
    for (int i = 0; i < thnum; ++i) {
        threads[i].join();
    }
    end = clock();
    std::cout << "multi thread mutex result: " << total << std::endl;
    std::cout << "multi thread mutex time: " << end - start << std::endl;
 
    total = 0;
    start = clock();
    for (int i = 0; i < thnum; ++i) {
        threads[i] = std::thread(spinlock_click);
    }
    for (int i = 0; i < thnum; ++i) {
        threads[i].join();
    }
    end = clock();
    std::cout << "spin lock result: " << total << std::endl;
    std::cout << "spin lock time: " << end - start << std::endl;
    getchar();
    return 0;
}
 
/*
single thread result: 100000000
single thread time: 231
multi thread no mutex result: 11501106
multi thread no mutex time: 261
multi thread atomic result: 100000000
multi thread atomic time: 1882
multi thread mutex result: 100000000
multi thread mutex time: 16882
spin lock result: 100000000
spin lock time: 45063
*/
```