---
title: "Algorithm: Back to Basic"
date: 2024-10-17T21:53:00+08:00
---

Back to basic!

## Quick Sort

```C++
class Solution {
 private:
  void recursive(std::vector<int> &nums, int left, int right) {
    if (left >= right) {
      return;
    }

    int mid = left + (right - left) / 2;
    int val = nums[mid];
    std::swap(nums[mid], nums[right]);
    int low = left;
    int high = right - 1;
    while (low <= high) {
      if (nums[low] > val) {
        std::swap(nums[low], nums[high--]);
      } else {
        ++low;
      }
    }
    std::swap(nums[low], nums[right]);

    recursive(nums, left, low - 1);
    recursive(nums, low + 1, right);
  }

 public:
  void quickSort(std::vector<int> &nums) {
    recursive(nums, 0, nums.size() - 1);
  }
};
```

## Binary Search

```C++
size_t lowerBound(const std::vector<int> &nums, int target) {
  size_t left = 0;
  size_t right = nums.size();
  while (left < right) {
    const size_t mid = left + (right - left) / 2;
    // Find the first element >= the target
    if (nums[mid] >= target) {
      right = mid;
    } else {
      left = mid + 1;
    }
  }
  return left;
}
```

Reference: https://imageslr.com/2020/03/15/binary-search.html

## Priority Queue / Heap

```C++
class PriorityQueue {
 private:
  std::vector<int> data_;

  void siftDown(size_t idx) {
    while (idx < data_.size()) {
      size_t child = 2 * idx + 1;
      if (child >= data_.size()) {
        break;
      }
      if (child + 1 < data_.size() && data_[child] < data_[child + 1]) {
        child += 1;
      }
      if (data_[child] < data_[idx]) {
        break;
      }
      std::swap(data_[idx], data_[child]);
      idx = child;
    }
  }

 public:
  explicit PriorityQueue(size_t capacity) {
    data_.reserve(capacity);
  }
  explicit PriorityQueue(const std::vector<int> &vec) {
    data_ = vec;
    for (size_t i = data_.size() / 2 - 1; i != -1; --i) {
      siftDown(i);
    }
  }

  void push(int val) {
    data_.push_back(val);

    size_t idx = data_.size() - 1;
    size_t parent = (idx - 1) / 2;
    while (idx > 0 && data_[parent] < data_[idx]) {
      std::swap(data_[parent], data_[idx]);
      idx = parent;
      parent = (idx - 1) / 2;
    }
  }

  int pop() {
    std::swap(data_.front(), data_.back());
    int val = data_.back();
    data_.pop_back();
    siftDown(0);
    return val;
  }
};
```

## SharedPtr

```C++
template <typename T>
class SharedPtr {
 public:
  explicit SharedPtr(T *ptr) : ptr_{ptr}, cnt_(nullptr) {
    if (ptr_ != nullptr) {
      cnt_ = new size_t(1);
    }
  }

  SharedPtr(const SharedPtr &other) {
    ptr_ = other.ptr_;
    cnt_ = other.cnt_;
    increaseRef();
  }
  ~SharedPtr() {
    decreaseRef();
  }

  SharedPtr &operator=(const SharedPtr &other) {
    if (&other == this) {
      return *this;
    }
    decreaseRef();
    ptr_ = other.ptr_;
    cnt_ = other.cnt_;
    increaseRef();
    return *this;
  }

  T *get() { return ptr_; }
  T *operator->() { return ptr_; }
  T &operator*() { return *ptr_; }

 private:
  void increaseRef() {
    if (ptr_ != nullptr) {
      ++*cnt_;
    }
  }
  void decreaseRef() {
    if (ptr_ == nullptr) {
      return;
    }
    if (--*cnt_ == 0) {
      delete ptr_;
      delete cnt_;
    }
  }

  T *ptr_;
  size_t *cnt_;  // maybe atomic
};
```

## Related Leetcode Blogs

Stock Problem: https://labuladong.online/algo/dynamic-programming/stock-problem-summary/

