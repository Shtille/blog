---
layout: post
title: "Custom spinlock vs pthread one speed comparison"
author: "Shtille"
categories: journal
tags: [thoughts,samples,C++]
---

```cpp
#include <cstdlib>
#include <cstdio>
#include <thread>
#include <string>
#include <chrono>
#include <atomic>

#include <pthread.h>

class ScopeTimer {
	typedef std::chrono::high_resolution_clock clock;
	typedef std::chrono::duration<float> seconds;
public:
	ScopeTimer(const char* message)
	: message_(message)
	{
		start_time_ = clock::now();
	}
	~ScopeTimer()
	{
		end_time_ = clock::now();
		float time = std::chrono::duration_cast<seconds>(end_time_ - start_time_).count();
		printf("%s took %f seconds\n", message_.c_str(), time);
	}
	
private:
	std::string message_;
	std::chrono::time_point<clock> start_time_;
	std::chrono::time_point<clock> end_time_;
};

struct spinlock {
  std::atomic<bool> lock_ = {0};

  void lock() noexcept {
    for (;;) {
      // Optimistically assume the lock is free on the first try
      if (!lock_.exchange(true, std::memory_order_acquire)) {
        return;
      }
      // Wait for lock to be released without generating cache misses
      while (lock_.load(std::memory_order_relaxed)) {
        // Issue X86 PAUSE or ARM YIELD instruction to reduce contention between
        // hyper-threads
        std::this_thread::yield();
      }
    }
  }

  bool try_lock() noexcept {
    // First do a relaxed load to check if lock is free in order to prevent
    // unnecessary cache misses if someone does while(!try_lock())
    return !lock_.load(std::memory_order_relaxed) &&
           !lock_.exchange(true, std::memory_order_acquire);
  }

  void unlock() noexcept {
    lock_.store(false, std::memory_order_release);
  }
};


const int kCount = 100000;
const int kNumThreads = 2;

static int s_counter = 0;
static spinlock s_spin;
static pthread_spinlock_t s_pthread_spin;

void spin_test()
{
	for (int i = 0; i < kCount; ++i)
	{
		s_spin.lock();
		++s_counter;
		s_spin.unlock();
	}
}
void pthread_spin_test()
{
	for (int i = 0; i < kCount; ++i)
	{
		(void)pthread_spin_lock(&s_pthread_spin);
		++s_counter;
		(void)pthread_spin_unlock(&s_pthread_spin);
	}
}

int main()
{
	// Init resources
	if (pthread_spin_init(&s_pthread_spin, PTHREAD_PROCESS_PRIVATE) != 0)
	{
		return 2;
	}


	{
		ScopeTimer timer("custom spin test ");
		spin_test();
	}
	{
		ScopeTimer timer("pthread spin test");
		pthread_spin_test();
	}

	// Release resources
	(void)pthread_spin_destroy(&s_pthread_spin);

	return 0;
}
```

The results are following:

- custom spin test took 0.001570 seconds
- pthread spin test took 0.001758 seconds