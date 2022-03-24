---
title: TryReadLock And TryWriteLock In UE4
mathjax: false
tags: 
 - UE4
category: UE4
date: 2022-03-21 10:49:12
description: This article tells you how to implement a `FTryReadScopeLock` or `FTryWriteScopeLock` in UE4. 

---



# Introduction

I simply need to implement `FTryReadScopeLock` and `FTryWriteScopeLock` for some of my features these days. 

For the moment, the only way to `try get lock` is to use `FScopeTryLock`, which is based on a [`critical section`](https://en.wikipedia.org/wiki/Critical_section). 

But unfortunately, there is no `try get lock` method for [`Readers-writer lock`](https://en.wikipedia.org/wiki/Readers%E2%80%93writer_lock) in UE4. 

Thus I've extend the `FRWLock` implementation for both Windows and pthreads. 

# Implementation

## Extending `ScopeRWLock.h`

Add a `FTryReadScopeLock` class like this: 

```cpp

/** Try to get a FRWLock read-locked while this scope lives 
 * This is a utility class that handles scope level locking using TryLock.
 * Scope locking helps to avoid programming errors by which a lock is acquired
 * and never released.
 *
 * <code>
 *	{
 *		// Try to acquire a lock on CriticalSection for the current scope.
 *		FTryReadScopeLock ScopeTryLock(CriticalSection);
 *		// Check that the lock was acquired.
 *		if (ScopeTryLock.IsLocked())
 *		{
 *			// If the lock was acquired, we can safely access resources protected
 *			// by the lock.
 *		}
 *		// When ScopeTryLock goes out of scope, the critical section will be
 *		// released if it was ever acquired.
 *	}
 * </code>
*/
class FTryReadScopeLock
{
public:
	explicit FTryReadScopeLock(FRWLock* InLock)
		: Lock(InLock)
	{
		check(Lock);

		if (!Lock->TryReadLock())
		{
			Lock = nullptr; 
		}
		
	}

	~FTryReadScopeLock()
	{
		if (Lock)
		{
			Lock->ReadUnlock();
		}
		
	}

	FORCEINLINE bool IsLocked() const
	{
		return nullptr != Lock;
	}

private:
	FRWLock* Lock;

	UE_NONCOPYABLE(FTryReadScopeLock);
};
```

## Implementation For `PThreadRWLock` Platform

Add `TryReadLock` and `TryWriteLock` method to `PThreadRWLock.h`: 

```cpp
	bool TryReadLock()
	{
		return 0 == pthread_rwlock_tryrdlock(&Mutex);
	}

	bool TryWriteLock()
	{
		return 0 == pthread_rwlock_trywrlock(&Mutex);
	}

```

This shall work for most of non-Windows platform. 

## Implementation For `Windows `Platform

### Modification For `MinimalWindowsApi.h`

```cpp
MINIMAL_WINDOWS_API bool WINAPI TryAcquireSRWLockExclusive(PSRWLOCK SRWLock);
MINIMAL_WINDOWS_API bool WINAPI TryAcquireSRWLockShared(PSRWLOCK SRWLock);
```

### Modification For `MinimalWindowsApi.cpp`

```cpp
	MINIMAL_WINDOWS_API bool WINAPI TryAcquireSRWLockExclusive(PSRWLOCK SRWLock)
	{
		return ::TryAcquireSRWLockExclusive((::PSRWLOCK)SRWLock);
	}

	MINIMAL_WINDOWS_API bool WINAPI TryAcquireSRWLockShared(PSRWLOCK SRWLock)
	{
		return ::TryAcquireSRWLockShared((::PSRWLOCK)SRWLock);
	}
```

### Modification For `WindowsCriticalSection.h`

```cpp
	FORCEINLINE bool TryReadLock()
	{
		return Windows::TryAcquireSRWLockShared(&Mutex);
	}

	FORCEINLINE bool TryWriteLock()
	{
		return Windows::TryAcquireSRWLockExclusive(&Mutex);
	}
```



And everything is good to go. 
