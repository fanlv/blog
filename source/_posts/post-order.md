---
title: 后续遍历二叉树的几种思路
tags:
  - Arithmetic
  - Sort
categories:
  - Golang
date: 2021-08-12 10:43:05
updated: 2021-08-12 10:43:05
---

## 一、背景

最近在公司面试（一面、二面）候选人的时候，大多数候选人基本都能正确的写出非递归版的`前序遍历`和`中序遍历`二叉树，但是大多数人都不能正确的写出非递归版的`后续遍历`。跟一个曾经拿过`NOI银牌`同事试下讨论了下`后续遍历`算法到底难不难。结论是，说难也难说不难也不难，说不难是因为，如果你看过相关解法，你可以很快就就理解解法的思路。说难，是如果你没看过，或者看了过了很久又忘了，要在15分钟左右写个`Bug free`的版本还是有点难的。

跟同事讨论下二叉树遍历的几种写法，所以就有了这篇文章。

## 二、二叉树几种解法的思考

### 2.1 递归版

前序遍历递归

	func preOrderRecursion(node *TreeNode, ans *[]int) {
		if node == nil {
			return
		}
	
		*ans = append(*ans, node.Val)
		postorderTraversal1(node.Left, ans)
		postorderTraversal1(node.Right, ans)
		return
	}
	

中序遍历递归

	func inOrderRecursion(node *TreeNode, ans *[]int) {
		if node == nil {
			return
		}
	
		postorderTraversal1(node.Left, ans)
		*ans = append(*ans, node.Val)
		postorderTraversal1(node.Right, ans)
		return
	}
	
	
后序遍历递归

	func postOrderRecursion(node *TreeNode, ans *[]int) {
		if node == nil {
			return
		}
	
		postorderTraversal1(node.Left, ans)
		postorderTraversal1(node.Right, ans)
		*ans = append(*ans, node.Val)
		return
	}
	

### 2.2 迭代 - 栈

前序遍历 - 栈

	func preOrder(root *TreeNode) []int {
		res := make([]int, 0)
		stack := []*TreeNode{root}
		for len(stack) != 0 {
			node := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			res = append(res, node.Val)
	
			if node.Right != nil {
				stack = append(stack, node.Right)
			}
			if node.Left != nil {
				stack = append(stack, node.Left)
			}
		}
	
		return res
	}

中序遍历 - 栈
	
	func inOrder(root *TreeNode) []int {
		res := make([]int, 0)
		stack := make([]*TreeNode, 0)
		node := root
	
		for node != nil || len(stack) > 0 {
			if node != nil {
				stack = append(stack, node)
				node = node.Left
				continue
			}
	
			node = stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			res = append(res, node.Val)
			node = node.Right
		}
	
		return res
	}
	
后续遍历

	func postOrder(root *TreeNode) []int {
		res := make([]int, 0)
		node := root
		stack := make([]*TreeNode, 0)
		var prev *TreeNode
		for node != nil || len(stack) > 0 {
			if node != nil {
				stack = append(stack, node)
				node = node.Left
				continue
			}
	
			node = stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			if node.Right == nil || node.Right == prev {
				res = append(res, node.Val)
				prev = node
				node = nil
			} else {
				stack = append(stack, node)
				node = node.Right
			}
		}
	
		return res
	}

### 2.3 Morris 遍历

前序遍历 - Morris

	func preorderTraversal(root *TreeNode) (vals []int) {
	    var p1, p2 *TreeNode = root, nil
	    for p1 != nil {
	        p2 = p1.Left
	        if p2 != nil {
	            for p2.Right != nil && p2.Right != p1 {
	                p2 = p2.Right
	            }
	            if p2.Right == nil {
	                vals = append(vals, p1.Val)
	                p2.Right = p1
	                p1 = p1.Left
	                continue
	            }
	            p2.Right = nil
	        } else {
	            vals = append(vals, p1.Val)
	        }
	        p1 = p1.Right
	    }
	    return
	}

中序遍历 - Morris
	
	func inorderTraversal(root *TreeNode) (res []int) {
		for root != nil {
			if root.Left != nil {
				// predecessor 节点表示当前 root 节点向左走一步，然后一直向右走至无法走为止的节点
				predecessor := root.Left
				for predecessor.Right != nil && predecessor.Right != root {
					// 有右子树且没有设置过指向 root，则继续向右走
					predecessor = predecessor.Right
				}
				if predecessor.Right == nil {
					// 将 predecessor 的右指针指向 root，这样后面遍历完左子树 root.Left 后，就能通过这个指向回到 root
					predecessor.Right = root
					// 遍历左子树
					root = root.Left
				} else { // predecessor 的右指针已经指向了 root，则表示左子树 root.Left 已经访问完了
					res = append(res, root.Val)
					// 恢复原样
					predecessor.Right = nil
					// 遍历右子树
					root = root.Right
				}
			} else { // 没有左子树
				res = append(res, root.Val)
				// 若有右子树，则遍历右子树
				// 若没有右子树，则整颗左子树已遍历完，root 会通过之前设置的指向回到这颗子树的父节点
				root = root.Right
			}
		}
		return
	}

后序遍历 - Morris

	func reverse(a []int) {
	    for i, n := 0, len(a); i < n/2; i++ {
	        a[i], a[n-1-i] = a[n-1-i], a[i]
	    }
	}
	
	func postorderTraversal(root *TreeNode) (res []int) {
	    addPath := func(node *TreeNode) {
	        resSize := len(res)
	        for ; node != nil; node = node.Right {
	            res = append(res, node.Val)
	        }
	        reverse(res[resSize:])
	    }
	
	    p1 := root
	    for p1 != nil {
	        if p2 := p1.Left; p2 != nil {
	            for p2.Right != nil && p2.Right != p1 {
	                p2 = p2.Right
	            }
	            if p2.Right == nil {
	                p2.Right = p1
	                p1 = p1.Left
	                continue
	            }
	            p2.Right = nil
	            addPath(p1.Left)
	        }
	        p1 = p1.Right
	    }
	    addPath(root)
	    return
	}


### 2.4 基于栈帧的思想把递归转成for循环


我们可以把递归版本的迭代，基于函数调用的栈帧思想，转成`for`循环，如下代码，我们知道递归调用对应了`4`行代码：

	func postOrder(node *TreeNode, ans *[]int) {
		if node == nil {return}       // line == 0
		postOrder(node.Left, ans)     // line == 1
		postOrder(node.Right, ans)    // line == 2
		*ans = append(*ans, node.Val) // line == 3
	}


转成`for`循环如下，我们在每个`line`执行上面不同的逻辑操作。这种方法的好处是，无聊是前序、中序、后续算法，我们只要调整下面 `if line == xx`的逻辑就行了。理论上所有的递归转非递归都可以基于这个思想去做。
		
	type DFSNode struct {
		line int   // 表示代码行数
		v    *TreeNode // 表示当前 node 
	}
	
	func postorderTraversal(root *TreeNode) []int {
		stack := []*DFSNode{}
		stack = append(stack, &DFSNode{
			v:    root,
			line: 0,
		})
		
		ans := []int{}
		
		for len(stack) > 0 {
			cur := stack[len(stack)-1]
			if cur.line == 0 {  // 对应上面的 if node == nil {return}
				if cur.v == nil { // 如果 node 为 nil ，出栈
					stack = stack[0 : len(stack)-1]
					continue
				}
			} else if cur.line == 1 {  // 对应上面 postOrder(node.Left, ans)
				stack = append(stack, &DFSNode{ // 函数调用，压栈
					v:    cur.v.Left,
					line: 0, // 从第 0 行 开始
				})
			} else if cur.line == 2 { // postOrder(node.Right, ans)  
				stack = append(stack, &DFSNode{ // 函数调用，压栈
					v:    cur.v.Right,
					line: 0, // 从第 0 行 开始
				})
			} else if cur.line == 3 { // 
				ans = append(ans, cur.v.Val)
				stack = stack[0 : len(stack)-1]
			}
			
			cur.line++ // 执行下一行代码
		}
		return ans
	}
	
