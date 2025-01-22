---
title: "Algorithm: Back to Basic"
date: 2024-10-17T21:53:00+08:00
draft: true
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
  explicit SharedPtr(T *data) : data_(data), cnt_(nullptr) {
    if (data_ != nullptr) {
      cnt_ = new std::atomic<size_t>(1);
    }
  }
  ~SharedPtr() { decreaseRef(); }

  SharedPtr(const SharedPtr &other) : data_(other.data_), cnt_(other.cnt_) {
    increaseRef();
  }
  SharedPtr &operator=(const SharedPtr &other) {
    if (this == &other) {
      return *this;
    }
    decreaseRef();
    data_ = other.data_;
    cnt_ = other.cnt_;
    increaseRef();
    return *this;
  }

  SharedPtr(SharedPtr &&other) noexcept : data_(other.data_), cnt_(other.cnt_) {
    other.data_ = nullptr;
    other.cnt_ = nullptr;
  }
  SharedPtr &operator=(SharedPtr &&other) noexcept {
    if (this == &other) {
      return *this;
    }
    data_ = other.data_;
    cnt_ = other.cnt_;
    other.data_ = nullptr;
    other.cnt_ = nullptr;
    return *this;
  }

  T *get() { return data_; }
  T *operator->() { return data_; }
  T &operator*() { return *data_; }

 private:
  void increaseRef() {
    if (data_ == nullptr) {
      return;
    }
    ++*cnt_;
  }
  void decreaseRef() {
    if (data_ == nullptr) {
      return;
    }
    if (--*cnt_ == 0) {
      delete cnt_;
      delete data_;
    }
  }

  T *data_;
  std::atomic<size_t> *cnt_;
};

```

## LRUCache

```C++
template <typename K, typename V>
class LRUCache {
 public:
  explicit LRUCache(size_t capacity) : capacity_(capacity) {}

  void Put(K key, const V &value) {
    const auto it = map_.find(key);
    if (it == map_.end()) {
      if (list_.size() == capacity_) {
        map_.erase(list_.back().first);
        list_.pop_back();
      }
      list_.emplace_front(key, value);
      map_.emplace(key, list_.begin());
    } else {
      it->second->second = value;
      list_.splice(list_.begin(), list_, it->second);
    }
  }
  bool Get(K key, V *value) {
    const auto it = map_.find(key);
    if (it == map_.end()) {
      return false;
    }
    *value = it->second->second;
    list_.splice(list_.begin(), list_, it->second);
    return true;
  }

 private:
  std::list<std::pair<K, V>> list_;
  std::unordered_map<K, typename std::list<std::pair<K, V>>::iterator> map_;
  size_t capacity_;
};
```

## Related Leetcode Blogs

Stock Problem: https://labuladong.online/algo/dynamic-programming/stock-problem-summary/

