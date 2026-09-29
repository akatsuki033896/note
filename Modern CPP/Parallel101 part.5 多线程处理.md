---
tags:
  - Parallel101
---
# `std::chrono`：时间标准库(C++11)

利用**强类型**的特点可以区分时间点和时间段
时间点：2023 年 1 月 1 日 12 时 12 分 12 秒
类型：`chrono::steady_clock::time_point`

时间段：1 分 30 秒
类型： `chrono::milliseconds` `chrono::seconds` `chrono::minutes`

运算符重载：时间段+时间点=时间点，时间点-时间点=时间段

```cpp
auto t0 = std::chrono::steady_clock::now(); // 获取当前时间点
auto t1 = std::chrono::steady_clock::now();
auto dt = t1 - t0; // dt 类型为时间段
int64_t ms = std::chrono::duration_cast<std::chrono::milliseconds>(dt).count();
```

## `duration` 时间段：作为`double`类型

- `duration_cast`：可以在任意的`duration`类型之间转换
- `duration<T, R>`：用`T`类型表示，时间单位为`R`，省略不写为秒，毫秒为 `std::milli` 微秒为`std::micro`
- `seconds` 为 `duration<int64_t>` 类型别名
- `milliseconds` 为 `duration<int64_t, std::milli>` 类型别名

```cpp
using double_ms = std::chrono::duration<double, std::milli>;
double ms = std::chrono::duration_cast<double_ms>(dt).count();
```

## `std::this_thread::sleep_for`

可替代类 Unix 系统的`usleep` 让前线程休眠一段时间然后继续，单位可以自己指定

```cpp
std::this_thread::sleep_for(std::chrono::milliseconds(400));
```

## `std::this_thread::sleep_until`

接受一个时间点，让前线程休眠直到某个时间点

```cpp
auto t = std::chrono::steady_clock::now() + std::chrono::milliseconds(400);
std::this_thread::sleep_until(t);
```

# 线程

## 进程和线程的概念

进程是一个应用程序被操作系统拉起，加载到内存之后从开始执行到执行结束的一个过程，是应用程序的一次执行。线程是进程的一次执行，是被系统独立分配和调度的基本单位。

一个进程可以有多个线程，线程共享一片内存空间，开销小，进程的内存空间独立，开销大，在高性能并行计算中多线程更好。线程是 cpu 执行调度的最小单位。

## `std::thread`：创建线程(C++11)

`std::thread` 构造函数的参数可以是任意 lambda 表达式。当线程启动时，就会执行这个 lambda
里的内容

```cpp
void download(std::string file) {
	for (int i = 0; i < 10; i++) {
		std::cout << i * 10 << "%" << std::endl;
		std::this_thread::sleep_for(std::chrono::seconds(2));
	}
std::cout << "Download complete:" << file << std::endl;
}

int main() {
	std::thread t1([&] {
		download("test.zip"); // make thread
	});
	return 0;
}
```

作为一个 C++ 类，`std::thread` 同样遵循 RAII 思想和三五法则：因为管理着资源，他自定义了解构函数，删除了拷贝构造/赋值函数，但是提供了移动构造/赋值函数。因此创建的线程对象退出作用域后线程会被销毁。

## `.join()`：主线程等待子线程结束

想要让主线程不要急着退出，等子线程也结束了再退出，可以用 `std::thread` 类的成员函数 `join()` 来等待该进程结束

```cpp
int main() {
	std::thread t1([&] {
		download("test.zip");
	});
	interact();
	std::cout << "Waiting for child thread..." << std::endl;
	t1.join();
	std::cout << "Child thread exited!" << std::endl;
	return 0;
}
```
## `.detach()`：分离线程

成员函数 `detach() `分离线程，意味着线程的生命周期不再由当前 `std::thread` 对象管理，而是在线程退出以后自动销毁自己。但是 `detach` 的问题是进程退出时候不会等待所有子线程执行完毕。

```cpp
int main() {
	std::thread t1([&] {
		download("test.zip");
	});
	t1.detach() // 分离
	interact();
	return 0;
}
```

## 全局线程池

### 将线程移动到全局线程池

把 `t1` 对象移动到一个全局变量去，从而延长其生命周期到函数体外

```cpp
std::vector<std::thread> pool; // 建立thread的列表
void myfunc() {
	std::thread t1([&] {
		download("test.zip");
	});
	pool.push_back(std::move(t1)); // pushback对象为thread类需要使用move
}
// t1的控制权在全局的pool中
int main() {
	myfunc();
	interact();
	for(auto &t : pool)
		t.join(); // 等待子线程结束
	return 0;
}
```

### 主函数退出后自动 `join` 全部线程
自定义`ThreadPool`类，他的解构函数会在主函数退出后自动调用

```cpp
class ThreadPool {
	std::vector<std::thread> pool;
	public:
	void pushback(std::thread t) {
		pool.push_back(std::move(t));
	}
	~ThreadPool() {
	for (auto &t : pool)
		t.join();
	}
};

ThreadPool tpool;
void myfunc() {
	std::thread t1([&] {
		download("test.zip");
	});
	tpool.pushback(std::move(t1));
}
// t1的控制权在全局的pool中
int main() {
	myfunc();
	interact();
	return 0;
}
```

### `std::jthread`：析构函数里会自动调用 `join()`  (C++20)

符合 RAII 思想，解构函数里会自动调用 `join()` 函数，从而保证 `pool` 解构时会自动等待全部线程执行完毕

```cpp
std::vector<std::jthread> pool;
	void myfunc() {
		std::jthread t1([&] {
		download("test.zip");
	});
	pool.push_back(std::move(t1));
}
int main() {
	myfunc();
	interact();
	return 0;
}
```
# 异步

异步运行指的是多线程之间的同步，比如两个任务，是并行运行的，那么可以把它们放到两个不同的线程里去运行。在具体编程中，就会涉及到线程的同步及数据的共享，标准库当中提供了同步及共享的方案：`std::future std::promise` 这两个库 一般是搭配使用的。

## `std::async`

`std::async` 接受一个带返回值的 `lambda`，自身返回一个 `std::future` 对象，lambda 的函数体将在另一个线程里执行。接下来你可以在 `main` 里面做一些别的事情，lambda的函数体会持续在后台运行。最后调用 `future` 的 `get()` 方法。如果此时函数体执行还没完成，会等待完成，并获取返回值。

```cpp
int download(std::string file) {
	for (int i = 0; i < 10; i++) {
		std::cout << i * 10 << "%" << std::endl;
		std::this_thread::sleep_for(std::chrono::seconds(2));
	}
	std::cout << "Download complete:" << file << std::endl;
	return 404;
}

void interact() {
	std::string user_name;
	std::cin >> user_name;
	std::cout << "Hi, " << user_name << std::endl;
}

int main() {
	std::future<int> fret = std::async([&]{
		return download("test.zip");
	});
	interact();
	int ret = fret.get();
	std::cout << "Download result:" << ret << std::endl;
	return 0;
}
```

## `wait()`：显式的等待

除了`get()` 会等待线程执行完毕外，`wait()`也可以等待他执行完，但是不会返回其值

```cpp
interact();
fret.wait();
std::cout << "Wait returned!" << std::endl;
int ret = fret.get();
std::cout << "Download result:" << ret << std::endl;
```
## `wait_for()` ：等待一段时间

`wait_for()` 则可以指定一个最长等待时间，用 `chrono` 里的类表示单位, 他会返回一个`std::future_status` 表示等待是否成功。如果超过这个时间线程还没有执行完毕，则放弃等待，返回 `future_status::timeout`，如果线程在指定的时间内执行完毕，则认为等待成功，返回`future_status::ready`

同理还有 `wait_until()` 其参数是一个时间点

```cpp
interact();
while (true) {
	std::cout << "Waiting for download completed..." << std::endl;
	auto stat = fret.wait_for(std::chrono::seconds(2));
	if (stat == std::future_status::ready) {
		std::cout << "Future is ready!" << std::endl;
		break;
	}
	else {
		std::cout << "Future is not ready!" << std::endl;
	}
}
int ret = fret.get();
std::cout << "Download result:" << ret << std::endl;
return 0;
}
```

## `std::promise`：手动创建线程

`std::async` 会自动创建线程，使用 `std::promise` 手动创建，在线程返回的时候，用 `set_value()` 设置返回值。在主线程里，用 `get_future()` 获取其 `std::future` 对象，进一步 `get()` 可以等待并获取线程返回值。

```cpp
int main() {
	std::promise<int> pret;
	std::thread t1([&] {
		auto ret = download("test.zip");
		pret.set_value(ret);
	});
	std::future<int> fret = pret.get_future();
	interact();
	int ret = fret.get();
	std::cout << "Download result:" << ret << std::endl;
	t1.join();
	return 0;
}
```

## `std::future` 相关

1. `future` 为了三五法则，删除了拷贝构造/赋值函数。如果需要浅拷贝，实现共享同一个 `future` 对象，可以用 `std::shared_future`
2. 如果不需要返回值，`std::async` 里 lambda的返回类型可以为 `void`， 这时 `future` 对象的类型为 `std::future<void>`
3. 同理有 `std::promise<void>`，他的 `set_value()` 不接受参数，仅仅作为同步用，不传递任何实际的值

```cpp
int main() {
	std::shared_future<void> fret = std::async([&]{
		return download("test.zip");
	});
	auto fret2 = fret, fret3 = fret;
	interact();
	fret3.wait();
	std::cout << "Download complete" << std::endl;
	return 0;
}
```
# 互斥量

## `std::mutex`：上锁

调用 `std::mutex` 的 `lock()` 时，会检测 `mutex` 是否已经上锁，如果没有锁定，则对 `mutex` 进行上锁。如果已经锁定，则陷入等待，直到 `mutex`被另一个线程解锁后，才再次上锁。调用`unlock()` 则会进行解锁操作。

这样，就可以保证`mtx.lock()` 和`mtx.unlock()` 之间的代码段，同一时间只有
一个线程在执行，从而避免数据竞争

```cpp
int main() {
	std::vector<int> arr;
	std::mutex mtx;
	std::thread t1([&] {
		for (int i = 0; i < 1000; i++) {
			mtx.lock();
			arr.push_back(1);
			mtx.unlock();
		}
	});
	
	std::thread t2([&] {
		for (int i = 0; i < 1000; i++) {
			mtx.lock();
			arr.push_back(2);
			mtx.unlock();
		}
	});
	t1.join();
	t2.join();
	return 0;
}
```

对于多个对象，每个对象一个 `mutex`，独立地上锁，可以避免不必要的锁定，提升高并发时的性能
## 符合 RAII 的锁
### `std::lock_guard`: 符合RAII的上锁/解锁

根据RAII 思想，可将锁的持有视为资源，上锁视为锁的获取，解锁视为锁的释放。`std::guard`的构造函数调用`lock()` 解构函数调用`unlock()` 从而退出函数作用域时能够自动解锁，避免程序员粗心不小心忘记解锁。

```cpp
std::mutex mtx;
std::thread t1([&] {
	for (int i = 0; i < 1000; i++) {
		std::lock_guard gcd(mtx);
		arr.push_back(1);
	}
});
```
### `std::unique_lock`: 符合RAII 自由度更高

`std::lock_guard` 严格在解构时 `unlock()`，但是有时候我们会希望提前 `unlock()`。这时可以用 `std::unique_lock`，他额外存储了一个 `flag` 表示是否已经被释放。他会在解构检测这个flag，如果没有释放，则调用`unlock()`，否则不调用。

然后可以直接调用`unique_lock` 的 `unlock()` 函数来提前解锁，但是即使忘记解锁也没关系，退出作用域时候他还会自动检查一遍要不要解锁

```cpp
std::mutex mtx;
std::thread t1([&] {
	for (int i = 0; i < 1000; i++) {
		std::unique_lock gcd(mtx);
		arr.push_back(1);
	}
});
```
#### 构造函数的额外参数 `std::defer_lock`

指定了这个参数的话，`std::unique_lock` 不会在构造函数中调用 `mtx.lock()`，需要之后
再手动调用`grd.lock()` 才能上锁。好处依然是即使忘记 `grd.unlock()` 也能够自动调用 `mtx.unlock()`

```cpp
int main() {
    std::mutex mtx1, mtx2;
    std::thread t1([&] {
		for (int i = 0; i < 5; i++) {
			std::scoped_lock grd(mtx1, mtx2);
		}
	});
    std::thread t2([&] {
		for (int i = 0; i < 5; i++) {
			std::scoped_lock grd(mtx2, mtx1);
		}
	});
    return 0;
}
```
#### 额外参数 `std::try_to_lock`

`mutex` 对象调用`try_lock()`而不是 `lock()` 之后，可以用`grd.owns_lock()` 判断是否上锁成功

```cpp
int main() {
	std::vector<int> arr;
	std::mutex mtx;
	std::thread t1([&] {
		std::unique_lock grd(mtx, std::try_to_lock);
		if (grd.owns_lock())
			std::cout << "t1 success" << std::endl;
		else
		std::cout << "t1 failed" << std::endl;
		std::this_thread::sleep_for(std::chrono::milliseconds(400));
	});
	
	std::thread t2([&] {
		std::unique_lock grd(mtx, std::try_to_lock);
		if (grd.owns_lock())
			std::cout << "t2 success" << std::endl;
		else
			std::cout << "t2 failed" << std::endl;	
		std::this_thread::sleep_for(std::chrono::milliseconds(400));
	});
	
	t1.join();
	t2.join();
	return 0;
}
```
#### 额外参数 `std::adopt_lock`

如果当前 `mutex` 已经上锁了，但是之后仍然希望用 RAII 思想在解构时候自动调用 `unlock()`，可以用 `std::adopt_lock` 作为 `std::unique_lock` 或 `std::lock_guard`的第二个参数，这时他们会默认 `mtx` 已经上锁
### `try_lock`: 上锁失败时不等待

`lock()` 如果发现 `mutex` 已经上锁的话，会等待他直到他解锁。也可以用无阻塞的`try_lock()`，他在上锁失败时不会陷入等待，而是直接返回 `false` 如果上锁成功，则会返回`true`

```cpp
std::mutex mtx;
int main() {
	if (mtx.try_lock())
		std::cout << "success" << std::endl;
	else
		std::cout << "failed" << std::endl;
	if (mtx.try_lock())
		std::cout << "success" << std::endl;
	else
		std::cout << "failed" << std::endl;
	mtx.unlock();
	return 0;
}
```

第一次上锁，因为还没有人上锁，所以成功了，返回 `true`
第二次上锁，由于自己已经上锁，所以失败了，返回 `false`
### `try_lock_for`: 只等待一段时间

`try_lock()` 碰到已经上锁的情况，会立即返回 `false`

如果需要等待，但仅限一段时间，可以用`std::timed_mutex` 的 `try_lock_for()` 函数，
他的参数是最长等待时间，同样是由`chrono` 指定时间单位。超过这个时间还没
成功就会返回 `false`, 如果这个时间内上锁成功则返回 `true`

接受时间点的有`try_lock_until`

```cpp
if (mtx.try_lock_for(std::chrono::milliseconds(400)))
	std::cout << "success" << std::endl;
```

>[!important] 
>
>`std::unique_lock` 和 `std::mutex` 有相同的类型接口。
>`std::unique_lock` 具有 `mutex` 的所有成员函数：`lock(), unlock(), try_lock(), try_lock_for()` 等。除了他会在解构时按需自动调用 `unlock()`

# 死锁

同时执行的两个线程，他们中发生的指令不一定是同步的，因此有可能出现这种情况:
t1 执行 `mtx1.lock()` t2 执行 `mtx2.lock()`，t1 执行 `mtx2.lock()`后失败，陷入等待，t2 执行 `mtx1.lock()` 失败，陷入等待，双方都在等着对方释放锁，但是因为等待而无法释放锁，从而要无限制等下去，则死锁。

## 原因 1: 两个线程同时持有两个锁

### 解决方法 1:不要持有两个锁

一个线程永远不要同时持有两个锁，分别上锁

```cpp
std::mutex mtx1, mtx2;
	std::thread t1([&] {
		for (int i = 0; i < 5; i++) {
			mtx1.lock();
			mtx1.unlock();
			mtx2.lock();
			mtx2.unlock();
		}
	});

	std::thread t2([&] {
		for (int i = 0; i < 5; i++) {
			mtx2.lock();
			mtx2.unlock();
			mtx1.lock();
			mtx1.unlock();
		}
	});
});
```

### 解决方法 2: 上锁顺序一致

```cpp
2.cpp
std::mutex mtx1, mtx2;
	std::thread t1([&] {
		for (int i = 0; i < 5; i++) {
			mtx1.lock();
			mtx2.lock();
			mtx2.unlock();
			mtx1.unlock();
		}
	});

	std::thread t2([&] {
		for (int i = 0; i < 5; i++) {
			mtx1.lock();
			mtx2.lock();
			mtx2.unlock();
			mtx1.unlock();
		}
	});
});

```

### 解决方法 3:`std::lock`同时对多个上锁

接受任意多个 `mutex` 作为参数，并且他保证在无论任意线程中调用的顺序是否相
同，都不会产生死锁问题

```cpp
std::lock(mtx1, mtx2);
mtx1.unlock();
mtx2.unlock();
```
#### `std::lock` 的 RAII 版本：`std::scoped_lock`

和 `std::lock_guard` 相对应，`std::lock` 也有 RAII 的版本 `std::scoped_lock`。只不过他
可以同时对多个 `mutex` 上锁

```cpp
std::scoped_lock grd(mtx1, mtx2);
```

## 原因 2: 同一个线程重复调用 `lock()`

`other`看到`mtx1` 已经上锁，还以为是别的线程上的锁，于是陷入等待。殊不知是调用他的
`func` 上的锁，`other` 陷入等待后 `func` 里的 `unlock()` 永远得不到调用

```cpp
std::mutex mtx;
void other() {
	mtx.lock();
	mtx.unlock();
}
void func() {
	mtx.lock();
	other();
	mtx.unlock();
}
int main() {
	func();
}
```

### 解决方法 1: 将要调用的函数里不要再上锁

把`lock`去掉，在文档中说明这个函数不是多线程安全的，调用这个函数之前要保证某`mutex` 已经上锁
### 解决方法 2: 用 `std::recursive_mutex`

自动判断是不是同一个线程`lock()` 了多次同一个锁，如果是则让计数器加 1，之后 `unlock()` 会让计数器减 1，减到 0 时才真正解锁。但是相比普通的 `std::mutex` 有一定性能损失
同理如果你同时需要 `try_lock_for()` 的话还有 `std::recursive_timed_mutex`

```cpp
std::recursive_mutex mtx;
void other() {
    mtx.lock();
    // ...
    mtx.unlock();
}
void func() {
    mtx.lock();
    other();
    mtx.unlock();
}
int main() {
    func();
    return 0;
}
```
# 多线程的数据结构

多个线程同时访问同一个 `vector` 会出现数据竞争（data-race）现象，但是可以封装出一个线程安全的`vector`。让一个数据结构变得多线程安全，就要使其访问都受到一个 `mutex` 的保护。

```cpp
class MTVector {
    std::vector<int> m_arr;
    std::mutex m_mtx;
public:
    void push_back(int val) {
        m_mtx.lock();
        m_arr.push_back(val);
        m_mtx.unlock();
    }
    const size_t size() {
        m_mtx.lock();
        size_t ret = m_arr.size();
        m_mtx.unlock();
        return ret;
    }
};
```

然而却出错了：因为 `size()` 是 `const` 函数，而 `mutex::lock()` 却不是 `const` 的。为了让 `this` 为 `const` 时仅仅给 `m_mtx` 开后门，可以用 `mutable` 关键字修饰他，从而所有成员里只有他不是 `const` 的，即 `mutable std::mutex m_mtx;`

## 读写锁
我们知道读可以共享，写必须独占，且写和读不能共存。  针对更具体的情况，又发明了读写锁，他允许的状态有：
1. $n$ 个人读取，没有人写入
2. $1$ 个人写入，没有人读取
3. 没有人读取，也没有人写入

### `shared_mutex`：读写锁

上锁时，要指定你的需求，负责调度的读写锁会帮你判断要不要等待。
若需求修改 / 写入数据，使用 `lock()` 和 `unlock()` 的组合
若需求为读取数据，可以共享，使用 `lock_shared()` 和 `unlock_shared()` 的组合

```c++
class MTVector {
    std::vector<int> m_arr;
    // std::mutex m_mtx;
    mutable std::shared_mutex m_mtx;
public:
    void push_back(int val) {
        m_mtx.lock();
        m_arr.push_back(val);
        m_mtx.unlock();
    }
    const size_t size() {
        m_mtx.lock_shared();
        size_t ret = m_arr.size();
        m_mtx.unlock_shared();
        return ret;
    }
};
```

### `std::shared_lock`：符合 RAII 思想的 `lock_share`

正如`std::unique_lock` 针对 `lock()`，也可以用 `std::shared_lock` 针对 `lock_shared()`
这样就可以在函数体退出时自动调用 `unlock_shared()`，更加安全了。`shared_lock` 同样支持 `defer_lock` 做参数，`owns_lock()` 判断等

```cpp
class MTVector {
    std::vector<int> m_arr;
    // std::mutex m_mtx;
    mutable std::shared_mutex m_mtx;
public:
    void push_back(int val) {
        std::unique_lock grd(m_mtx); // lock()
        m_arr.push_back(val);
    }
    const size_t size() {
        std::shared_lock grd(m_mtx); // lock_shard()
        size_t ret = m_arr.size();
        return ret;
    }
};
```

### 访问者模式：只需一次性上锁，且符合 RAII 思想

Accessor 或者说 Viewer 模式，常用于设计 GPU 容器 OpenVDB 数据结构的访问，也是采用了 Accessor 的设计，并且还有 ConstAccessor 和 Accessor 两种，分别对应于读和写

```cpp
class MTVector {
    std::vector<int> m_arr;
    std::mutex m_mtx;
public:
    class Accessor {
        MTVector &m_that;
        std::unique_lock<std::mutex> m_guard;
    public:
        Accessor(MTVector &that) : m_that(that), m_guard(that.m_mtx) {}
        const void push_back(int val) {
            return m_that.m_arr.push_back(val);
        }
        const size_t size() {
            return m_that.m_arr.size();
        }
    };
    Accessor access() {
        return {*this};
    }
};
int main() {
    MTVector arr;
    std::thread t1([&] {
        auto axr = arr.access();
        for (int i = 0; i < 5; i++) {
            axr.push_back(i);
        }
    });
    std::thread t2([&] {
        auto axr = arr.access();
        for (int i = 0; i < 5; i++) {
            axr.push_back(i);
        }
    });
    t1.join();
    t2.join();
    std::cout << arr.access().size() << std::endl;
    return 0;
}
```
# 条件变量

## 等待被唤醒

`wait()` 将会让当前线程陷入等待, 在其他线程中调用 `notify_one()` 则会唤醒那个陷入等待的线程。可以发现 `std::condition_variable` 必须和`std::unique_lock<std::mutex>` 一起用

```cpp
#include <condition_variable>
int main() {
    std::condition_variable cv;
    std::mutex mtx;
    std::thread t1 ([&] {
        std::unique_lock lck(mtx);
        cv.wait(lck);
        std::cout << "t1 is awake" << std::endl;
    });
    std::this_thread::sleep_for(std::chrono::milliseconds(400));
    std::cout << "notifying..." << std::endl;
    cv.notify_one();
    t1.join();
    return 0;
}
```
## 等待某一条件成真

额外指定一个参数，变成`cv.wait(lck, expr)` 的形式，其中 `expr` 是个 `lambda` 表达式，只有其返回值为 `true` 时才会真正唤醒，否则继续等待

```cpp
int main() {
    std::condition_variable cv;
    std::mutex mtx;
    bool ready = false;

    std::thread t1 ([&] {
        std::unique_lock lck(mtx);
        cv.wait(lck, [&] {  return ready;   });
        lck.unlock();
        std::cout << "t1 is awake" << std::endl;
    });

    std::cout << "notify is not ready" << std::endl;
    cv.notify_one(); // useless now. since ready = false

    ready = true;
    std::cout << "notify is ready" << std::endl;
    cv.notify_one(); // awakening t1 since ready = true

    t1.join();
    return 0;
}
```
## 多个等待者

`notify_one()` 只会唤醒其中一个等待中的线程，而 `notify_all()` 会唤醒全部

> [!important]
> 这就是为什么 `wait()` 需要一个 `unique_lock` 作为参数，因为要保证多个线程被唤醒时，
> 只有一个能够被启动。如果不需要，在`wait()` 返回后调用 `lck.unlock()` 即可。`wait()` 的过程中会暂时 `unlock()` 这个锁

```cpp
int main() {
    std::condition_variable cv;
    std::mutex mtx;

    std::thread t1 ([&] {
        std::unique_lock lck(mtx);
        cv.wait(lck);
        std::cout << "t1 is awake" << std::endl;
    });

    std::thread t2 ([&] {
        std::unique_lock lck(mtx);
        cv.wait(lck);
        std::cout << "t2 is awake" << std::endl;
    });

    std::thread t3 ([&] {
        std::unique_lock lck(mtx);
        cv.wait(lck);
        std::cout << "t3 is awake" << std::endl;
    });

    std::this_thread::sleep_for(std::chrono::milliseconds(400));
    std::cout << "notify one" << std::endl; // awakening t1 only
    cv.notify_one();

    std::this_thread::sleep_for(std::chrono::milliseconds(400));
    std::cout << "notify all" << std::endl; // awakening t1 and t2
    cv.notify_all();

    t1.join();
    t2.join();
    t3.join();
    return 0;
}
```

> [!example] 案例：实现生产者-消费者模式

类似于消息队列
生产者：往 `foods` 队列里推送食品，推送后会通知消费者来用餐。
消费者：等待 `foods` 队列里有食品，没有食品则陷入等待，直到被通知

```cpp
int main() {
    std::condition_variable cv;
    std::mutex mtx;
    std::vector<int> foods;

    std::thread t1([&] {
        for (int i = 0; i < 2; i++) {
            std::unique_lock lck(mtx);
            cv.wait(lck, [&] {
                return foods.size() != 0;
            });
            auto food = foods.back();
            foods.pop_back();
            lck.unlock();
            std::cout << "t1 got food: " << food << std::endl;
        }
    });

    std::thread t2 ([&] {
        std::unique_lock lck(mtx);
            cv.wait(lck, [&] {
                return foods.size() != 0;
            });
            auto food = foods.back();
            foods.pop_back();
            lck.unlock();
            std::cout << "t2 got food: " << food << std::endl;
    });

    foods.push_back(42);
    cv.notify_one();
    foods.push_back(233);
    cv.notify_one();

    foods.push_back(15);
    foods.push_back(16);
    cv.notify_all();

    t1.join();
    t2.join();
    return 0;
}
```
## 其他注意事项

1. `std::condition_variable` 仅仅支持 `std::unique_lock<std::mutex>` 作为 `wait` 的参数，如果需要用其他类型的 `mutex` 锁，可以用 `std::condition_variable_any`。
2. 他还有 `wait_for()` 和 `wait_until()` 函数，分别接受 `chrono` 时间段和时间点作为参数。详见： https://en.cppreference.com/w/cpp/thread/condition_variable/wait_for
# 原子操作

> 多线程修改一个计数器
> 多线程往一个计数器 `int` 变量里累加肯定会出错，在 CPU 看来会变成三个指令：
>
> 1.  读取`counter` 变量到 rax 寄存器
> 2.  rax 寄存器的值加上 1
> 3.  把 rax 写入到 `counter` 变量
>
> 有多个线程运行时，这个顺序是不确定的，现代 CPU 还有高速缓存，乱序执行，指令级并
> 行等优化策略，你根本不知道每条指令实际的先后顺序

```cpp
#include <iostream>
#include <thread>
int main() {
    int counter = 0;
    std::thread t1([&] {
        for (int i = 0; i < 1000; i++)
            counter++;
    });
    std::thread t2([&] {
        for (int i = 0; i < 1000; i++)
            counter++;
    });
    t1.join();
    t2.join();
    std::cout << "counter:" << counter << std::endl;
    return 0;
}
```

## 暴力解决：用 `mutex` 上锁

可以防止多个线程同时修改 `counter` 变量，从而不会冲突。但是`mutex` 太过重量级，他会让线程被挂起，从而需要通过系统调用，进入内核层，调度到其他线程执行，有很大的开销。

```cpp
std::mutex mtx;
int main(){
	...
    std::thread t1([&] {
        for (int i = 0; i < 1000; i++){
            mtx.lock();
            counter++;
            mtx.unlock();
        }

    });
}
```
## `atomic`：有专门的硬件指令加持

更轻量级，对他的 `+=` 等操作，会被编译器转换成专门的指令。CPU 识别到该指令时，会锁住内存总线，放弃乱序执行等优化策略（将该指令视为一个同步点，强制同步掉之前所有的内存操作），从而向你保证该操作是原子的（不可分割），不会加法加到一半另一个线程插一脚进来。

把 `int` 改成 `atomic<int>` 即可

```cpp
int main() {
    // int counter = 0;
    std::atomic<int> counter = 0;
    std::mutex mtx;
    std::thread t1([&] {
        for (int i = 0; i < 1000; i++){
            counter++;
        }
    });
    std::thread t2([&] {
        for (int i = 0; i < 1000; i++){
            counter++;
        }
    });
    t1.join();
    t2.join();
    std::cout << "counter:" << counter << std::endl;
    return 0;
}
```

> [!danger]
> 不过要注意了，这种写法：
> `counter = counter + 1;` // 错，不能保证原子性
> `counter += 1;` // OK，能保证原子性
> `counter++; `// OK，能保证原子性

### `fetch_add`：和 `+=` 等价

除了用方便的运算符重载之外，还可以直接调用相应的函数名，比如：

- `fetch_add` 对应于 `+=`
- `store` 对应于 `=`
- `load` 用于读取其中的 `int` 值

```cpp
int main() {
    std::atomic<int> counter;
    counter.store(0);
    std::mutex mtx;
    std::thread t1([&] {
        for (int i = 0; i < 1000; i++){
            counter.fetch_add(1);
        }
    });
    std::thread t2([&] {
        for (int i = 0; i < 1000; i++){
            counter.fetch_add(1);
        }
    });
    t1.join();
    t2.join();
    std::cout << "counter:" << counter.load() << std::endl;
    return 0;
}
```

除了会导致 `atm` 的值增加 `val` 外，还会返回 `atm` 增加前的值，`int old = atm.fetch_add(val)` 存储到 `old`。 这个特点使得他可以用于并行地往一个列表里追加数据：追加写入的索引就是 `fetch_add`， 返回的旧值当然这里也可以 `counter++`，不过要追加多个的话还是得用到 `counter.fetch_add(n)`。
### `exchange`：读取时写入

`exchange(val)` 会把 `val` 写入原子变量，同时返回其旧的值
### `compare_exchange_strong`：读取，比较是否相等，相等则写入

`compare_exchange_strong(old, val)` 会读取原子变量的值，比较他是否和 `old` 相等，如果不相等，则把原子变量的值写入 `old`。如果相等，则把 `val` 写入原子变量。返回一个 `bool` 值，表示是否相等。注意 `old` 这里传的其实是一个引用，因此 `compare_exchange_strong` 可修改他的值。