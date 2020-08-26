---
title: MyVector
date: 2020-08-26 07:13:53
tags: Cpp
categories:
cover: https://www.adobe.com/content/dam/cc/us/en/creativecloud/design/discover/vector-file/vector-file_P1_900x420.jpg.img.jpg
---
<meta name="referrer" content="no-referrer" />


```cpp
#pragma once
#include <cstddef>
#include <algorithm>
#include <memory>

template <typename T>
class Vector
{
private:
    size_t length;
    size_t capacity;
    T *buffer;

    struct Deleter
    {
        void operator()(T *buffer) const
        {
            operator delete(buffer);
        }
    };

public:
    Vector(size_t capacity = 100) : capacity(capacity), length(0), buffer(static_cast<T *>(operator new(sizeof(T) * capacity))) {}

    ~Vector()
    {
        // using unique_ptr to release the buffer place
        std::unique_ptr<T, Deleter> deleter(buffer, Deleter());

        for (int i = 0; i < length; ++i)
            buffer[i].~T();
    }

    // copy constructor
    Vector(const Vector &copy) : capacity(copy.length), length(0), buffer(static_cast<T *>(operator new(sizeof(T) * capacity)))
    {
        try
        {
            for (int i = 0; i < copy.length; ++i)
                push_back(copy.buffer[i]);
        }
        catch (...)
        {
            // destroy everything that was to created to make it exception safe
            ~Vector();

            // make sure the execptions continue propagating after the cleanup has completed
            throw;
        }
    }

    // copy assignment : using swap to prevent the error from the exception occurrence while this process is running
    Vector &operator=(const Vector &copy)
    {
        // Copy and Swap idiom
        Vector tmp(copy);
        tmp.swap(*this);
        return *this;
    }

    // move constructor
    Vector(Vector &&move) noexcept
        : capacity(0), length(0), buffer(nullptr)
    {
        move.swap(*this);
    }

    // move assignment
    Vector &operator=(Vector &&move) noexcept
    {
        move.swap(*this);
        return *this;
    }

    void push_back(const T &value)
    {
        resizeIfRequire();
        pushBackInternal(value);
    }

    void push_back(T &&value)
    {
        resizeIfRequire();
        pushBackInternal(std::forward<T>(value));
    }

    void pop_back()
    {
        --length;
        buffer[length].~T();
    }

    void swap(Vector &other) noexcept
    {
        using std::swap;
        swap(capacity, other.capacity);
        swap(length, other.length);
        swap(buffer, other.buffer);
    }

    void reserve(size_t capacityUpperBound)
    {
        if (capacityUpperBound > capacity)
            reserveCapacity(capacityUpperBound);
    }

    T operator[](size_t index) const
    {
        return at(index);
    }

    T at(size_t index) const
    {
        return buffer[index];
    }

private:
    void resizeIfRequire()
    {
        if (length == capacity)
        {
            size_t newCapacity = std::max(2.0, capacity * 1.5);
            reserveCapacity(newCapacity);
        }
    }

    void pushBackInternal(T const &value)
    {
        new (buffer + length) T(value);
        ++length;
    }

    void moveBackInternal(T &&value)
    {
        new (buffer + length) T(std::move(value));
        ++length;
    }

    void reserveCapacity(size_t newCapacity)
    {
        Vector<T> tmpBuffer(newCapacity);

        // if we can guarantee that the move constructor does not throw
        simpleCopy<T>(tmpBuffer);

        tmpBuffer.swap(*this);
    }

    // SFINAE (substitution failure is not an error) method overload
    template <typename X>
    typename std::enable_if<std::is_nothrow_constructible<X>::value == false>::type
    simpleCopy(Vector<T> &dst)
    {
        std::for_each(buffer, buffer + length, [&dst](T &v) {
            dst.pushBackInternal(v);
        });
    };

    template <typename X>
    typename std::enable_if<std::is_nothrow_constructible<X>::value == true>::type
    simpleCopy(Vector<T> &dst)
    {
        std::for_each(buffer, buffer + length, [&dst](T &v) {
            dst.moveBackInternal(std::move(v));
        });
    };
};
```

