---
title: Abstract Linked List
date: 2019-03-17 16:06:20
tags: abstract linkedList
categories: abstract
---
![circula rlinked list](abstract-linked-list/circularlinkedlist.png)
<!-- more -->

# Linked List

## [Leecode P141](https://leetcode.com/problems/linked-list-cycle/): 判断链表是否有环
### 描述
    Given a linked list, determine if it has a cycle in it.

    To represent a cycle in the given linked list, we use an integer pos which represents the position (0-indexed) in the linked list where tail connects to. If pos is -1, then there is no cycle in the linked list.

    Example 1:
    Input: head = [3,2,0,-4], pos = 1
    Output: true
    Explanation: There is a cycle in the linked list, where tail connects to the second node.

    Example 2:
    Input: head = [1,2], pos = 0
    Output: true
    Explanation: There is a cycle in the linked list, where tail connects to the first node.

    Example 3:
    Input: head = [1], pos = -1
    Output: false
    Explanation: There is no cycle in the linked list.

### 解题实录

1. 暴力解题，在规定时间内没有完成遍历则有环
2. 使用Set存储，在遍历过程中发现有已经遍历的节点则有环
3. 快慢指针，如果慢指针追上了快指针则有环

### 代码

1. 思路2
```go
func hasCycleMap(head *ListNode) bool {
	m := make(map[*ListNode]bool)
	p := head
	for p != nil {

		if _, ok := m[p]; !ok {
			m[p] = true
		} else {
			return true
		}
		p = p.Next
	}
	return false
}
```

2. 思路3
```go
func hasCycleFL(head *ListNode) bool {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow = slow.Next
		fast = fast.Next.Next
		if slow == fast {
			return true
		}
	}
	return false
}
```
### Feedback

