---
title: 'Leetcode_simple'
tags:
date: '2022-11-20'
categories:
draft: false
---

## 判断是否平衡二叉树
```go
func isBalanced(root *TreeNode) bool {
    if root == nil{
        return  true
    }
    return abs(depth(root.Left)-depth(root.Right)) <= 1 && isBalanced(root.Left) && isBalanced(root.Right)
}

func isBalancedV2(root *TreeNode) bool {
	return height(root) >= 0
}

func height(root *TreeNode) int {
	if root == nil {
		return 0
	}
	leftHeight := height(root.Left)
	rightHeight := height(root.Right)
	if leftHeight == -1 || rightHeight == -1 || abs(leftHeight-rightHeight) > 1 {
		return -1
	}
	return max(leftHeight, rightHeight) + 1
}
```
## 获取树的高度
```go
func depth(root *TreeNode) int {
    if root == nil{
        return 0
    }
    return max(depth(root.Left),depth(root.Right)) + 1
}

func depthV2(root *TreeNode) (height int) {
	if root == nil {
		return 0
	}
	
	var dfs func(node *TreeNode, depth int)
	dfs = func(node *TreeNode, depth int) {
		if node == nil {
			return
		}
		if depth > height {
			height = depth
		}
		dfs(node.Left, depth+1)
		dfs(node.Right, depth+1)
	}

	dfs(root, 1)
	return height
}

func depthV3(root *TreeNode) (depth int) {
	if root == nil {
		return
	}
	queue := []*TreeNode{root}
	for len(queue) > 0 {
		// 逐层遍历，所以这里 cur 代表每层的 node 个数
		cur := len(queue)
		for i := 0; i < cur; i++ {
			node := queue[0]
			queue = queue[1:]
			if node.Left != nil {
				queue = append(queue, node.Left)
			}
			if node.Right != nil {
				queue = append(queue, node.Right)
			}
		}
		// 遍历完一层
		depth++
	}
	return depth
}
```
## 二叉树的镜像
```go
func mirrorTree(root *TreeNode) *TreeNode {
    if root == nil{
        return nil
    }
    r := mirrorTree(root.Right)
    l := mirrorTree(root.Left)
    root.Right = l
    root.Left = r
    return root
}

func mirrorTreeV2(root *TreeNode) *TreeNode {
	if root == nil {
		return root
	}
	queue := []*TreeNode{root}
	for len(queue) > 0 {
		node := queue[0]
		queue = queue[1:]
		node.Left, node.Right = node.Right, node.Left
		if node.Left != nil {
			queue = append(queue, node.Left)
		}
		if node.Right != nil {
			queue = append(queue, node.Right)
		}
	}
	return root
}
```
## 对称的二叉树
```go
func isSymmetric(root *TreeNode) bool {
    return checkSymmetric(root,root)
}
func checkSymmetric(q,p *TreeNode) bool{
    if q == nil && p == nil{
        return true
    }
    if q == nil || p == nil{
        return false
    }
    return q.Val == p.Val && checkSymmetric(q.Left,p.Right) && checkSymmetric(q.Right,p.Left)
}
```
## 递增顺序搜索树
```go
func increasingBST(root *TreeNode) *TreeNode {
    if root == nil{
        return nil
    }
    dummy := &TreeNode{}
    res := dummy

    var dfs func(*TreeNode)
    dfs = func(r *TreeNode){
        if r == nil{
            return 
        }
        dfs(r.Left)
        res.Right = r
        r.Left = nil
        res = r

        dfs(r.Right)
    }
    dfs(root)
    return dummy.Right
}
```
## 二叉树展开为链表
```go
func flatten(root *TreeNode) {
    if root == nil {
        return
    }
    flatten(root.Left)
    flatten(root.Right)
    
    left := root.Left
    right := root.Right
    
    root.Left = nil
    root.Right = left
    
    p := root
    for p.Right != nil {
        p = p.Right
    }
    p.Right = right
}
func flatten(root *TreeNode) {  
    if root == nil {
        return
    }
    stack := []*TreeNode{root}
    
    var prev *TreeNode
    for len(stack) > 0 {
        curr := stack[len(stack)-1]
        stack = stack[:len(stack)-1]
        
        if prev != nil {
            prev.Left = nil
            prev.Right = curr
        }
        
        if curr.Right != nil {
            stack = append(stack, curr.Right) 
        }
        
        if curr.Left != nil {
            stack = append(stack, curr.Left)
        }  
        prev = curr
    } 
}
func flatten(root *TreeNode) {
    curr := root
    for curr != nil {
        if curr.Left != nil {
            next := curr.Left
            for next.Right != nil {
                next = next.Right
            }
            next.Right = curr.Right
            curr.Right = curr.Left
            curr.Left = nil
        }
        curr = curr.Right 
    }
}
```

## 二叉树的公共祖先
```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil{
        return nil
    }

    if root.Val == p.Val || root.Val == q.Val{
        return root
    }
    l,r := lowestCommonAncestor(root.Left,p,q),lowestCommonAncestor(root.Right,p,q)
    if r != nil && l != nil{
        return root
    }  
    if r == nil{
        return l
    }
    return r
}
func lowestCommonAncestor(root, p, q *TreeNode) (ancestor *TreeNode) {
    ancestor = root
    for {
        if p.Val < ancestor.Val && q.Val < ancestor.Val {
            ancestor = ancestor.Left
        } else if p.Val > ancestor.Val && q.Val > ancestor.Val {
            ancestor = ancestor.Right
        } else {
            return
        }
    }
}
func getPath(root, target *TreeNode) (path []*TreeNode) {
    node := root
    for node != target {
        path = append(path, node)
        if target.Val < node.Val {
            node = node.Left
        } else {
            node = node.Right
        }
    }
    path = append(path, node)
    return
}

func lowestCommonAncestor(root, p, q *TreeNode) (ancestor *TreeNode) {
    pathP := getPath(root, p)
    pathQ := getPath(root, q)
    for i := 0; i < len(pathP) && i < len(pathQ) && pathP[i] == pathQ[i]; i++ {
        ancestor = pathP[i]
    }
    return
}
```
## 是否二叉搜索树
```go
func isValidBST(root *TreeNode) bool {
    return helper(root, math.MinInt64, math.MaxInt64)
}
func helper(root *TreeNode, lower, upper int) bool {
    if root == nil {
        return true
    }
    if root.Val <= lower || root.Val >= upper {
        return false
    }
    return helper(root.Left, lower, root.Val) && helper(root.Right, root.Val, upper)
}
```

## 从上到下 左到右 分层 打印二叉树, 返回二维数组
```go
func levelOrder(root *TreeNode) [][]int {
    ret := [][]int{}
    if root == nil {
        return ret
    }
    q := []*TreeNode{root}
    for i := 0; len(q) > 0; i++ {
        ret = append(ret, []int{})
        p := []*TreeNode{}
        for j := 0; j < len(q); j++ {
            node := q[j]
            ret[i] = append(ret[i], node.Val)
            if node.Left != nil {
                p = append(p, node.Left)
            }
            if node.Right != nil {
                p = append(p, node.Right)
            }
        }
        q = p
    }
    return ret
}
```
## 从上到下 打印二叉树 二
```go
func levelOrder(root *TreeNode) []int {
    ret := []int{}
    if root == nil {
        return ret
    }
    q := []*TreeNode{root}
    for i := 0; len(q) > 0; i++ {
        p := []*TreeNode{}
        for j := 0; j < len(q); j++ {
            node := q[j]
            ret = append(ret, node.Val)
            if node.Left != nil {
                p = append(p, node.Left)
            }
            if node.Right != nil {
                p = append(p, node.Right)
            }
        }
        q = p
    }
    return ret
}
```

## 从上到下 奇偶 打印二叉树 三
```go
func zigzagLevelOrder(root *TreeNode) (ans [][]int) {
    if root == nil {
        return
    }
    queue := []*TreeNode{root}
    for level := 0; len(queue) > 0; level++ {
        vals := []int{}
        q := queue
        queue = nil
        for _, node := range q {
            vals = append(vals, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        // 本质上和层序遍历一样，我们只需要把奇数层的元素翻转即可
        if level%2 == 1 {
            for i, n := 0, len(vals); i < n/2; i++ {
                vals[i], vals[n-1-i] = vals[n-1-i], vals[i]
            }
        }
        ans = append(ans, vals)
    }
    return
}
```
## 二叉树的第 k 大节点
```go
func kthLargest(root *TreeNode, k int) int {
    var dfs func(*TreeNode)
    var res = -1
    dfs = func(node *TreeNode){
        if node == nil{
            return 
        }
        dfs(node.Right)
        k--
        if k == 0 {
            res = node.Val
            return 
        }
        dfs(node.Left)
    }
    dfs(root)
    return res
}
```
## 二叉搜索树中两个节点之和
```go
func findTarget(root *TreeNode, k int) bool {
    set := map[int]struct{}{}
    var dfs func(*TreeNode) bool
    dfs = func(node *TreeNode) bool {
        if node == nil {
            return false
        }
        if _, ok := set[k-node.Val]; ok {
            return true
        }
        set[node.Val] = struct{}{}
        return dfs(node.Left) || dfs(node.Right)
    }
    return dfs(root)
}
```

## 二叉树的直径
```go
func diameterOfBinaryTree(root *TreeNode) (ans int) {
    var depth func(*TreeNode)int
    depth = func(node *TreeNode)int{
        if node == nil{
            return 0
        }
        l, r := depth(node.Left), depth(node.Right)
        ans = max(ans, l + r)
        return max(l, r) + 1
    }
    depth(root)
    return
}
```
## 二叉树最深最左
```go
func findBottomLeftValue(root *TreeNode) (ans int) {
    q := []*TreeNode{root}
    for len(q) > 0 {
        node := q[0]
        q = q[1:]
        if node.Right != nil {
            q = append(q, node.Right)
        }
        if node.Left != nil {
            q = append(q, node.Left)
        }
        ans = node.Val
    }
    return
}

func findBottomLeftValue(root *TreeNode) (curVal int) {
    curHeight := 0
    var dfs func(*TreeNode, int)
    dfs = func(node *TreeNode, height int) {
        if node == nil {
            return
        }
        height++
        dfs(node.Left, height)
        dfs(node.Right, height)
        if height > curHeight {
            curHeight = height
            curVal = node.Val
        }
    }
    dfs(root, 0)
    return
}
```

## 合并二叉树
```go
func mergeTrees(t1, t2 *TreeNode) *TreeNode {
    if t1 == nil {
        return t2
    }
    if t2 == nil {
        return t1
    }
    t1.Val += t2.Val
    t1.Left = mergeTrees(t1.Left, t2.Left)
    t1.Right = mergeTrees(t1.Right, t2.Right)
    return t1
}
```
## 二叉树的坡度
```go
func findTilt(root *TreeNode) (ans int) {
    var dfs func(*TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil {
            return 0
        }
        sumLeft := dfs(node.Left)
        sumRight := dfs(node.Right)
        ans += abs(sumLeft - sumRight)
        return sumLeft + sumRight + node.Val
    }
    dfs(root)
    return
}
```

## 单值二叉树
```go
func isUnivalTree(root *TreeNode) bool {
    return root == nil || (root.Left == nil || root.Val == root.Left.Val && isUnivalTree(root.Left)) &&
                         (root.Right == nil || root.Val == root.Right.Val && isUnivalTree(root.Right))
}
```
## 单值二叉树
```go
func isUnivalTree(root *TreeNode) bool {
    if root == nil{
        return false
    }
    var dfs func(root *TreeNode) bool
    dfs = func(node *TreeNode) bool {
        if node == nil {
            return true
        }
        if node.Val != root.Val{
            return false
        }
        return dfs(node.Left) && dfs(node.Right)
    }
    
    return dfs(root)
}
```
## 重建二叉树
```go
func buildTree(preorder []int, inorder []int) *TreeNode {
    if len(preorder) == 0 {
        return nil
    }
    root := &TreeNode{preorder[0], nil, nil}
    i := 0
    for ; i < len(inorder); i++ {
        if inorder[i] == preorder[0] {
            break
        }
    }
    root.Left = buildTree(preorder[1:len(inorder[:i])+1], inorder[:i])
    root.Right = buildTree(preorder[len(inorder[:i])+1:], inorder[i+1:])
    return root
}
```

## 二叉搜索树，数组是否是后序
```go
func verifyPostorder(postorder []int) bool {
	var recur func(int, int) bool
	recur = func(i, j int) bool {
		if i >= j {
			return true
		}
		p := i
		for postorder[p] < postorder[j] {
			p++
		}
		m := p
		for postorder[p] > postorder[j] {
			p++
		}
		return p == j && recur(i, m-1) && recur(m, j-1)
	}

	return recur(0, len(postorder)-1)
}

func verifyTreeOrder(postorder []int) bool {
	var stack []int
	root := math.MaxInt32

	for i := len(postorder) - 1; i >= 0; i-- {
		if postorder[i] > root {
			return false
		}

		for len(stack) > 0 && stack[len(stack)-1] > postorder[i] {
			root = stack[len(stack)-1]
			stack = stack[:len(stack)-1]
		}
		stack = append(stack, postorder[i])
	}
	return true
}
```

## 二叉树中 和 为某值 的路径
```go
func pathSum(root *TreeNode, target int) (ans [][]int) {
    path := []int{}
    var dfs func(*TreeNode, int)
    dfs = func(node *TreeNode, left int) {
        if node == nil {
            return
        }
        left -= node.Val
        path = append(path, node.Val)
        defer func() { path = path[:len(path)-1] }()
        if node.Left == nil && node.Right == nil && left == 0 {
            ans = append(ans, append([]int(nil), path...))
            return
        }
        dfs(node.Left, left)
        dfs(node.Right, left)
    }
    dfs(root, target)
    return
}
```

## 没有重复项的字符串全排列
```go
func permutation(s string) []string {
    var res []string
    c := []byte(s)
    var dfs func(x []byte, pos int) 
    dfs = func(x []byte, pos int) {
        if pos == len(c) {
            res = append(res, string(x))    
            return
        }
        cache := make(map[byte]bool)
        for i := pos; i < len(c); i++ {  
            if _, has := cache[c[i]]; has {
                continue    
            }
            cache[c[i]] = true
            x[pos], x[i] = x[i], x[pos]
            dfs(x, pos+1)
            x[pos], x[i] = x[i], x[pos]  
        }
    }
    dfs(c, 0)
    return res
}
```


## 反转链表
```go
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head
    for curr != nil {
        next := curr.Next // b->c c nil
        curr.Next = prev // a -> nil, b -> a, c -> b -> a
        prev = curr // a -> nil, b->a
        curr = next // b->c, c
    }
    return prev
}

func reverseList(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseList(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}
```
## 反转链表二
```go
func reverseBetween(head *ListNode, left, right int) *ListNode {
    // 设置 dummyNode 是这一类问题的一般做法
    dummyNode := &ListNode{Val: -1}
    dummyNode.Next = head
    pre := dummyNode
    // 先找到要反转的头结点
    for i := 0; i < left-1; i++ {
        pre = pre.Next
    }
    cur := pre.Next
    for i := 0; i < right-left; i++ {
        next := cur.Next
        cur.Next = next.Next
        next.Next = pre.Next
        pre.Next = next
    }
    return dummyNode.Next
}

```

## 相同的树
```
func isSameTree(p *TreeNode, q *TreeNode) bool {
    if p == nil && q == nil {
        return true
    }
    if p == nil || q == nil {
        return false
    }
    if p.Val != q.Val {
        return false
    }
    return isSameTree(p.Left, q.Left) && isSameTree(p.Right, q.Right)
}
```

## 倒序打印链表
```go
func reversePrint(head *ListNode) []int {
	var out []int
	for head != nil {
		out = append([]int{head.Val}, out...)
		head = head.Next
	}
	return out
}

func reversePrint(head *ListNode) (out []int) {
 if head == nil {
        return []int{}
    }
    return append(reversePrint(head.Next),head.Val)
}
```
## 合并两个有序链表
```go
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    node := dummy
    for l1 != nil && l2 != nil{
        if l1.Val <= l2.Val{
            node.Next = l1
            l1 = l1.Next
        }else{
            node.Next = l2
            l2 = l2.Next
        }
        node = node.Next
    }
    if l1 != nil{
        node.Next = l1
    }else{
        node.Next = l2
    }
    return dummy.Next
}
```
## 链表的公共节点
```go
func getIntersectionNode(headA, headB *ListNode) *ListNode {
    if headA == nil || headB == nil {
        return nil
    }
    pa, pb := headA, headB
    for pa != pb {
        if pa == nil {
            pa = headB
        } else {
            pa = pa.Next
        }
        if pb == nil {
            pb = headA
        } else {
            pb = pb.Next
        }
    }
    return pa
}
```
## 环形链表
```go
func hasCycle(head *ListNode) bool {
    if head == nil || head.Next == nil {
        return false
    }
    slow, fast := head, head.Next
    for fast != slow {
        if fast == nil || fast.Next == nil {
            return false
        }
        slow = slow.Next
        fast = fast.Next.Next
    }
    return true
}
```

## 1的个数
```go
func hammingWeight(num uint32) (ones int) {
    for ; num > 0; num &= num - 1 {
        ones++
    }
    return
}
```

## 旋转数组中的最小值
```go
func minArray(numbers []int) int {
    low := 0
    high := len(numbers) - 1
    for low < high {
        pivot := low + (high - low) / 2
        if numbers[pivot] < numbers[high] {
            high = pivot
        } else if numbers[pivot] > numbers[high] {
            low = pivot + 1
        } else {
            high--
        }
    }
    return numbers[low]
}
```
## 连续子数组的最大和
```go
func maxSubArray(nums []int) int {
    max := nums[0]
    for i := 1; i < len(nums); i++ {
        if nums[i-1] > 0 {
            nums[i] += nums[i-1]
        }
        if nums[i] > max {
            max = nums[i]
        }
    }
    return max
}
```
## 股票最佳时机2
```go
func maxProfit(prices []int) (ans int) {
    for i := 1; i < len(prices); i++ {
        ans += max(0, prices[i]-prices[i-1])
    }
    return
}
```
## 股票最佳时机1
```go
func maxProfit(prices []int) (ans int) {
    min := math.MaxInt64
    max := 0
    for _,v := range prices{
        if v < min{
            min = v
        }
        if v - min > max{
            max = v-min
        }
    }
    return max
}
```

## 杨辉三角
```go
func generate(numRows int) [][]int {
    ans := make([][]int, numRows)
    for i := range ans {
        ans[i] = make([]int, i+1)
        ans[i][0] = 1
        ans[i][i] = 1
        for j := 1; j < i; j++ {
            ans[i][j] = ans[i-1][j] + ans[i-1][j-1]
        }
    }
    return ans
}
```
## 杨辉三角 第 n 行
```go
func getRow(rowIndex int) []int {
    row := make([]int, rowIndex+1)
    row[0] = 1
    for i := 1; i <= rowIndex; i++ {
        for j := i; j > 0; j-- {
            row[j] += row[j-1]
        }
    }
    return row
}
```
## 杨辉三角 第 n 行
```go
func getRow(rowIndex int) []int {
    row := make([]int, rowIndex+1)
    row[0] = 1
    for i := 1; i <= rowIndex; i++ {
        row[i] = row[i-1] * (rowIndex - i + 1) / i
    }
    return row
}
```

## 快速排序

```go
func sortArray(nums []int) []int {
	quickSort(nums, 0, len(nums)-1)
	return nums
}

func quickSort_part(nums []int, l, r int) int {
	// 轴线
	pivot := rand.Intn(r-l) + l
	// 将轴元素移至末尾
	nums[pivot], nums[r] = nums[r], nums[pivot]
	i := l - 1
	// 遍历 （l,r) 区间
	for j := l; j < r; j++ {
		// 比轴元素小的值，放到 l 区间左边
		if nums[j] < nums[r] {
			i++
			nums[j], nums[i] = nums[i], nums[j]
		}
	}
	// 最后，将轴元素交换回停止位置
	i++
	nums[i], nums[r] = nums[r], nums[i]
	return i
}

func quickSort(nums []int, l, r int) {
	if r <= l {
		return
	}
	mid := quickSort_part(nums, l, r)
	quickSort(nums, l, mid-1)
	quickSort(nums, mid+1, r)
}

```

## 青蛙跳台阶
```go
// 可以跳1阶，或者2阶
// f(n) = f(n-1) + f(n-2)
func numWays(n int) int {
	var a, b = 1, 1
	for ; n > 0 ; n-- {
		a, b = b, (a + b) % (1e9 + 7)
	}
	return a
}
// 递归
func jumpFloor(N int) int {
  if N <= 2 {
    return N
  }
​
  return jumpFloor(N-1) + jumpFloor(N-2)
}

// 可以跳1阶，或者3阶
// f(n) = f(n-1) + f(n-3)
func jumpFloor(N int) int {
  if N == 0{
    return 0
  }
  if N <= 2 {
    return 1
  }
​
  return jumpFloor(N-1) + jumpFloor(N-3)
}

func jumpFloor(N int) int {
  if N == 0{
    return 0
  }
  if N <= 2 {
    return 1
  }
​  dp := make([]int, N)
  dp = append(dp, 1, 1, 2, 3)
  for i:=2 ;i < N;i++{
    dp[i] = dp[i-1] + dp[i-3]
  }
  return dp[N]
}


// 可以跳1阶，或者2阶...n阶
func jumpFloor(N int) int {
  if N <= 2 {
    return N
  }

  return jumpFloor(N-1) * 2
}

func jumpFloor(N int) int {
  if N <= 2 {
    return N
  }
​
   b := 2
   for i := 3; i<= N;i++ {
      b = 2 * b
   }
   return b
}

func jumpFloor(N int) int {
  return 1 << (N -1)
}

```


## 最长公共前缀
```go
func longestCommonPrefix(strs []string) string {
	if len(strs) == 0 {
		return ""
	}
	pre := strs[0]
	for i := 1; i < len(strs); i++ {
		pre = commonPrefix(pre, strs[i])
		if pre == "" {
			return ""
		}
	}
	return pre
}

func commonPrefix(s string, t string) string {
	if s == t {
		return s
	}
	a := len(s)
	b := len(t)
	j := 0
	for i := 0; i < a; i++ {
		if i >= b || s[i] != t[i] {
			return s[:i]
		}
		j++
	}
	return s[:j]
}
```

## 移动零
```go
func moveZeroes(nums []int) {
	zeroIndex := 0
	for i := 0; i < len(nums); i++ {
		if nums[i] != 0 {
			nums[i], nums[zeroIndex] = nums[zeroIndex], nums[i]
			zeroIndex++
		}
	}
}
```

## 删除排序数组中的重复项
```go
func removeDuplicates(nums []int) int {
	j := 1
	for i := 1; i < len(nums); i++ {
		if nums[i] != nums[i-1] {
			nums[j] = nums[i]
			j++
		}
	}
	return j
}

func removeDuplicates2(nums []int) int {
	j := 0
	n := 2
	for i := 0; i < len(nums); i++ {
		if j < n || nums[i] != nums[j-n] {
			nums[j] = nums[i]
			j++

		}
	}
	return j
}
```
## 数组加 1
```go
func plusOne(digits []int) []int {
	for i := len(digits) - 1; i >= 0; i-- {
		digits[i]++
		if digits[i] < 10 {
			return digits
		}
		digits[i] = 0
	}
	if digits[0] == 0 {
		return append([]int{1}, digits...)
	}
	return digits
}
```

## 旋转数组
```go
func rotate(nums []int, k int) {
	k %= len(nums)
	reverse(nums, 0, len(nums)-1)
	reverse(nums, 0, k-1)
	reverse(nums, k, len(nums)-1)
}

func reverse(nums []int, n, m int) {
	for n < m {
		nums[n], nums[m] = nums[m], nums[n]
		n++
		m--
	}
}

```

## 合并两个有序数组
```go
func merge(nums1 []int, m int, nums2 []int, n int) {
	i := m + n
	for n > 0 {
		if m > 0 && nums1[m-1] > nums2[n-1] {
			nums1[i-1] = nums1[m-1]
			m--
		} else {
			nums1[i-1] = nums2[n-1]
			n--
		}
		i--
	}
}
```

## 数组相对排序
```go
func relativeSortArray(arr1 []int, arr2 []int) []int {
	rank := map[int]int{}
	for i, v := range arr2 {
		rank[v] = i
	}
	sort.Slice(arr1, func(i, j int) bool {
		x, y := arr1[i], arr1[j]
		rankX, hasX := rank[x]
		rankY, hasY := rank[y]
		if hasX && hasY {
			return rankX < rankY
		}
		if hasX || hasY {
			return hasX
		}
		return x < y
	})
	return arr1
}


func relativeSortArrayV2(arr1 []int, arr2 []int) []int {
	rank := map[int]int{}
	for i, v := range arr2 {
		rank[v] = i - len(arr2)
	}
	sort.Slice(arr1, func(i, j int) bool {
		x, y := arr1[i], arr1[j]
		if r, has := rank[x]; has {
			x = r
		}
		if r, has := rank[y]; has {
			y = r
		}
		return x < y
	})
	return arr1
}
```

## 链表 两数相加
```go
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
	head := &ListNode{}
	node := head
	carry := 0
	for l1 != nil || l2 != nil || carry != 0 {
		if l1 != nil {
			carry += l1.Val
			l1 = l1.Next
		}
		if l2 != nil {
			carry += l2.Val
			l2 = l2.Next
		}
		node.Next = &ListNode{Val: carry % 10}
		node = node.Next
		carry = carry / 10
	}
	return head.Next
}
```

## 无重复字符的最长子串
```go
func lengthOfLongestSubstring(s string) int {
	// 哈希集合，记录每个字符是否出现过
	set := map[int32]int{}
	// 右指针，初始值为 -1，相当于我们在字符串的左边界的左侧，还没有开始移动
	head := -1
	res := 0
	for i, v := range s {
		// 字符重复，且索引大于左指针，左指针刷新
		if index, ok := set[v]; ok && index > head {
			head = index
		} else {
			// 否则 当前索引-左指针，如果长度更大，则置换结果
			if cur := i - head; cur > res {
				res = cur
			}
		}
		// 记录字符的索引，重复的字符刷新索引
		set[v] = i
	}
	return res
}
```

## 最长回文串
```go
func longestPalindrome(s string) int {
	// 记录所有字符出现的次数
	m := map[uint8]int{}
	for i := 0; i < len(s); i++ {
		m[s[i]]++
	}
	r := 0
	// 超过2个的字符，最少有 偶数个可以用
	// 2 ，2/2*2 = 2
	// 3 ，3/2*2 = 2
	// 取多余的一个任何字符最为中间元素
	for _, v := range m {
		r += v / 2 * 2
		// 如果有剩余的单个，并且长度为偶数，则可以加入中心 + 1
		if v%2 == 1 && r%2 == 0 {
			r++
		}
	}
	return r
}
```

## 盛最多水的容器
```go
func maxArea(height []int) int {
	l, r := 0, len(height)-1
	res := 0
	for l < r {
		minH := min(height[l], height[r])
		res = max(res, minH*(r-l))
		for height[l] <= minH && l < r {
			l += 1
		}
		for height[r] <= minH && l < r {
			r -= 1
		}
	}
	return res
}

```

## 数组中重复的数字
```go
func findRepeatNumberV1(nums []int) int {
	set := make([]int, len(nums))
	for _, v := range nums {
		if set[v] != 0 {
			return v
		}
		set[v] = 1
	}
	return -1
}

// 数组原地比较
func findRepeatNumberV2(nums []int) int {
	i := 0
	for i < len(nums) {
		if nums[i] == i {
			i += 1
			continue
		}

		if nums[nums[i]] == nums[i] {
			return nums[i]
		}
		nums[nums[i]], nums[i] = nums[i], nums[nums[i]]
	}
	return -1
}
```
## 二进制中1的个数
```go
// 把一个整数减去1，再和原整数做与运算，会把该整数最右边一个1变成0.那么一个整数的二进制有多少个1，就可以进行多少次这样的操作。
func hammingWeightV2(num uint32) (ones int) {
	for ; num > 0; num &= num - 1 {
		ones++
	}
	return
}
```

## 调整数组顺序使奇数位于偶数前面
```go
func exchange(nums []int) []int {
	l, r := 0, len(nums)-1
	for l < r {
		// 左边数是奇数，跳过
		for nums[l]%2 == 1 && l < r {
			l++
		}
		// 右边数是偶数，跳过
		for nums[r]%2 == 0 && l < r {
			r--
		}
		// 有l是偶数，r是奇数，交换两个的位置
		if l < r {
			nums[l], nums[r] = nums[r], nums[l]
			l++
			r--
		}
	}
	return nums
}
```

## 链表中倒数第k个节点
```go
func getKthFromEnd(head *ListNode, k int) *ListNode {
	var n []*ListNode
	for head != nil {
		n = append(n, head)
		head = head.Next
	}
	return n[len(n)-k]
}

// 链表中倒数第k个节点 快慢指针
func getKthFromEndV2(head *ListNode, k int) *ListNode {
	fast, slow := head, head
	for i := 0; i < k; i++ {
		fast = fast.Next
	}
	for fast != nil {
		fast, slow = fast.Next, slow.Next
	}
	return slow
}
```

## 两个链表的第一个公共节点
```go
func getIntersectionNode(headA, headB *ListNode) *ListNode {
	if headA == nil || headB == nil {
		return nil
	}
	pa, pb := headA, headB
	for pa != pb {
		if pa == nil {
			pa = headB
		} else {
			pa = pa.Next
		}
		if pb == nil {
			pb = headA
		} else {
			pb = pb.Next
		}
	}
	return pa
}
```
## 和为s的两个数字
```go
func twoSum(nums []int, target int) []int {
	m := map[int]struct{}{}
	for _, v := range nums {
		if _, ok := m[target-v]; ok {
			return []int{v, target - v}
		}
		m[v] = struct{}{}
	}
	return nil
}


func twoSumV2(nums []int, target int) []int {
	l, r := 0, len(nums)-1
	for l < r {
		for nums[r] > target {
			r--
		}
		sum := nums[l] + nums[r]
		if sum > target {
			r--
		} else if sum < target {
			l++
		} else {
			return []int{nums[l], nums[r]}
		}
	}
	return nil
}
```

## 顺时针打印矩阵
```go
func spiralOrder(matrix [][]int) []int {
	if len(matrix) == 0 || len(matrix[0]) == 0 {
		return []int{}
	}
	var (
		rows, columns            = len(matrix), len(matrix[0])
		order                    = make([]int, rows*columns)
		index                    = 0
		left, right, top, bottom = 0, columns - 1, 0, rows - 1
	)

	for left <= right && top <= bottom {
		for column := left; column <= right; column++ {
			order[index] = matrix[top][column]
			index++
		}
		for row := top + 1; row <= bottom; row++ {
			order[index] = matrix[row][right]
			index++
		}
		if left < right && top < bottom {
			for column := right - 1; column > left; column-- {
				order[index] = matrix[bottom][column]
				index++
			}
			for row := bottom; row > top; row-- {
				order[index] = matrix[row][left]
				index++
			}
		}
		left++
		right--
		top++
		bottom--
	}
	return order
}


func spiralOrderV3(matrix [][]int) []int {
	var out []int
	if len(matrix) == 0 {
		return out
	}
	l, r, t, b := 0, len(matrix[0])-1, 0, len(matrix)-1

	for {
		// 从左至右
		for i := l; i <= r; i++ {
			// 遍历第 t 行
			out = append(out, matrix[t][i])
		}
		// 上边界向下收
		t++
		if t > b {
			break
		}
		// 从上至下
		for i := t; i <= b; i++ {
			// 遍历第 r 列
			out = append(out, matrix[i][r])
		}
		// 右边界向左收
		r--
		if r < l {
			break
		}
		// 从右至左
		for i := r; i >= l; i-- {
			// 遍历第 b 行
			out = append(out, matrix[b][i])
		}
		// 下边界向上收
		b--
		if b < t {
			break
		}
		// 从下至上
		for i := b; i >= t; i-- {
			// 遍历第 l 列
			out = append(out, matrix[i][l])
		}
		// 左边界向右收
		l++
		if l > r {
			break
		}
	}
	return out
}
```

## 第一个只出现一次的字符
```go
//在字符串 s 中找出第一个只出现一次的字符。如果没有，返回一个单空格。 s 只包含小写字母。
func firstUniqChar(s string) byte {
	var m [26]uint
	for _, n := range s {
		m[n-'a']++
	}
	for _, n := range s {
		if m[n-'a'] == 1 {
			return byte(n)
		}
	}
	return ' '
}
```

## 统计一个数字在排序数组中出现的次数
```go
func search(nums []int, target int) int {
	l, r := 0, len(nums)-1
	for l <= r {
		mid := l + (r-l)/2
		if nums[mid] > target {
			r = mid - 1
		} else if nums[mid] < target {
			l = mid + 1
		} else {
			if nums[l] != target {
				l++
			} else if nums[r] != target {
				r--
			} else {
				break
			}
		}
	}

	return r - l + 1

}
```


## 和为s的连续正数序列
```go
func findContinuousSequence(target int) [][]int {
	i, j := 1, 2
	var out [][]int
	for i < j {
		sum := (i + j) * (j - i + 1) / 2
		if sum < target {
			j++
		} else if sum > target {
			i++
		} else {
			var p []int
			for k := i; k <= j; k++ {
				p = append(p, k)
			}
			out = append(out, p)
			i++
		}
	}
	return out
}
```
## 二维数组中的查找
```go
func findNumberIn2DArray(matrix [][]int, target int) bool {
	x, y := len(matrix), 0

	for x >= 0 && y < len(matrix[0]) {
		if v := matrix[x][y]; v < target {
			y++
		} else if v > target {
			x--
		} else {
			return true
		}
	}
	return false
}
```

## 最小的k个数
```go
func getLeastNumbers(arr []int, k int) (out []int) {
	if k == 0 {
		return
	}

	out = append(out, arr[:k]...)
	sort.Ints(out)
	for _, i := range arr[k:] {
		if out[k-1] > i {
			out = append([]int{i}, out[:k-1]...)
		}
	}
	return
}
```

## 0～n-1中缺失的数字
```go
func missingNumber(nums []int) int {
	for i, num := range nums {
		if num != i {
			return i
		}
	}
	return len(nums)
}

func missingNumberV2(nums []int) int {
	l, r := 0, len(nums)-1

	for l <= r {
		mid := l + (r-l)/2
		if nums[mid] == mid {
			l = mid + 1
		} else {
			r = mid - 1
		}
	}
	return l
}
```

## 左旋转字符串
```go
func reverseLeftWords(s string, n int) string {
	return s[n:] + s[:n]
}

func reverseLeftWordsV2(s string, n int) string {
	var b strings.Builder
	l := len(s)
	for i := n; i < n+l; i++ {
		b.WriteRune(rune(s[i%l]))
	}
	return b.String()
}
```

## 有效的回文
```go
func isPalindrome(s string) bool {
	if s == "" {
		return true
	}
	s = strings.ToLower(s)
	l, r := 0, len(s)-1
	for l < r {
		for l < r && !isalnum(s[l]) {
			l++
		}
		for l < r && !isalnum(s[r]) {
			r--
		}
		if s[l] != s[r] {
			return false
		}
		l++
		r--
	}
	return true
}

func isalnum(ch byte) bool {
	return (ch >= 'a' && ch <= 'z') || (ch >= '0' && ch <= '9')
}
```

## 有效数独
```go
func isValidSudoku(board [][]byte) bool {
	var rows, cols [9][9]int
	var subboxes [3][3][9]int
	for i, row := range board {
		for j, col := range row {
			if col == '.' {
				continue
			}
			index := col - '1'
			rows[i][index]++
			cols[j][index]++
			subboxes[i/3][j/3][index]++
			if rows[i][index] > 1 || cols[j][index] > 1 || subboxes[i/3][j/3][index] > 1 {
				return false
			}
		}
	}
	return true
}
```

## 最长连续序列
```go
func longestConsecutive(nums []int) int {
	set := map[int]bool{}
	// 先将所有数去重存入集合
	for _, i := range nums {
		set[i] = true
	}
	l := 0
	for n := range set {
		// 如果当前值的前一个数不存在，则开始连续序列计数
		if !set[n-1] {
			curN := n
			curl := 1
			// 循环连续序列，直到不连续为止
			for set[curN+1] {
				curN++
				curl++
			}
			// 比较当前连续序列长度与上次连续序列长度，保留大值
			if l < curl {
				l = curl
			}
		}
	}
	return l
}
```

## 岛屿周长
```go
func islandPerimeter(grid [][]int) (ans int) {
	type pair struct{ x, y int }
	var dir4 = []pair{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}

	n, m := len(grid), len(grid[0])
	for i, row := range grid {
		for j, v := range row {
			if v == 1 {
				for _, d := range dir4 {
					if x, y := i+d.x, j+d.y; x < 0 || x >= n || y < 0 || y >= m || grid[x][y] == 0 {
						ans++
					}
				}
			}
		}
	}
	return
}
```

## 岛屿数量
```go
// 深度优先,找到一个 '1'，则遍历上下左右
func numIslands(grid [][]byte) int {
	var dfs func(grid [][]byte, i, j int)
	dfs = func(grid [][]byte, i, j int) {
		if i < 0 || i >= len(grid) ||
			j < 0 || j >= len(grid[0]) ||
			grid[i][j] == '0' {
			return
		}
		grid[i][j] = '0'
		dfs(grid, i+1, j)
		dfs(grid, i, j+1)
		dfs(grid, i-1, j)
		dfs(grid, i, j-1)
	}
	cnt := 0
	for i := 0; i < len(grid); i++ {
		for j := 0; j < len(grid[0]); j++ {
			if grid[i][j] == '1' {
				dfs(grid, i, j)
				cnt++
			}
		}
	}
	return cnt
}

// 广度优先
func numIslandsV2(grid [][]byte) int {
	var bfs func(grid [][]byte, i, j int)
	bfs = func(grid [][]byte, i, j int) {
		queue := [][]int{
			{i, j},
		}
		for len(queue) > 0 {
			i, j := queue[0][0], queue[0][1]
			queue = queue[1:]
			if i >= 0 && i < len(grid) && j >= 0 && j < len(grid[0]) && grid[i][j] == '1' {
				grid[i][j] = '0'
				queue = append(queue, [][]int{{i + 1, j}, {i - 1, j}, {i, j - 1}, {i, j + 1}}...)
			}
		}
	}
	cnt := 0
	for i := 0; i < len(grid); i++ {
		for j := 0; j < len(grid[0]); j++ {
			if grid[i][j] == '1' {
				bfs(grid, i, j)
				cnt++
			}
		}
	}
	return cnt
}
```
## 堆排序 Ο(nlogn)
```go
// 升序 用 大顶堆
// 降序 用 小顶堆
func heapSort(arr []int) []int {
    arrLen := len(arr)
    buildMaxHeap(arr, arrLen)

    // 循环，每次将根节点（最大值）放到堆尾，然后对剩余节点重排最大堆，最后就是升序
    for i := arrLen - 1; i >= 0; i-- {
        // 交换 堆根节点 和 堆尾
        swap(arr, 0, i)
        // 重排剩余的堆节点
        heapify(arr, 0, i)
    }
    return arr
}
// 构建堆
func buildMaxHeap(arr []int, arrLen int) {
    for i := arrLen / 2; i >= 0; i-- {
        heapify(arr, i, arrLen)
    }
}

func heapify(arr []int, i, arrLen int) {
    left := 2*i + 1
    right := 2*i + 2
    largest := i

    // 大于就是大根堆，< 就是小根堆
    if left < arrLen && arr[left] > arr[largest] {
        largest = left
    }
    if right < arrLen && arr[right] > arr[largest] {
        largest = right
    }
    if largest != i {
        swap(arr, i, largest)
        heapify(arr, largest, arrLen)
    }
}

func swap(arr []int, i, j int) {
    arr[i], arr[j] = arr[j], arr[i]
}
```
## 连续子数组和
```go
func checkSubarraySum(nums []int, k int) bool {
    m := len(nums)
    if m < 2 {
        return false
    }
    mp := map[int]int{0: -1}
    remainder := 0
    for i, num := range nums {
        remainder = (remainder + num) % k
        if prevIndex, has := mp[remainder]; has {
            if i-prevIndex >= 2 {
                return true
            }
        } else {
            mp[remainder] = i
        }
    }
    return false
}
```
## 和为k的子数组
```go
func subarraySum(nums []int, k int) int {
    count, pre := 0, 0
    m := map[int]int{}
    m[0] = 1
    for i := 0; i < len(nums); i++ {
        pre += nums[i]
        if _, ok := m[pre - k]; ok {
            count += m[pre - k]
        }
        m[pre] += 1
    }
    return count
} 
```

## 最长递增子序列
```go
func lengthOfLIS(nums []int) int {
    tails := make([]int,len(nums))
    res :=  0
    for _,num := range nums{
        i, j := 0, res
        for i < j{
            m := (i + j) / 2
            // 如果要求非严格递增，将此行 '<' 改为 '<=' 即可。
            if tails[m] < num{
                i = m + 1
            } else{
                j = m
            }
        }
        tails[i] = num
        if j == res{
            res += 1
        }
    }
    return res
}
```

## 二叉树所有路径
```go
func binaryTreePaths(root *TreeNode) []string {
    var res []string
    var dfs func(node *TreeNode,path []string)
    dfs = func(node *TreeNode,path []string){
        if node == nil{
            return
        }
        path = append(path, strconv.Itoa(node.Val))
        if node.Left == nil && node.Right == nil{
            res = append(res,strings.Join(path,"->"))
        }else{
            dfs(node.Left,path)
            dfs(node.Right,path)
        }
    }
    dfs(root,[]string{})
    return res
}

func binaryTreePaths(root *TreeNode) []string {
    paths := []string{}
    if root == nil {
        return paths
    }
    nodeQueue := []*TreeNode{}
    pathQueue := []string{}
    nodeQueue = append(nodeQueue, root)
    pathQueue = append(pathQueue, strconv.Itoa(root.Val))

    for i := 0; i < len(nodeQueue); i++ {
        node, path := nodeQueue[i], pathQueue[i]
        if node.Left == nil && node.Right == nil {
            paths = append(paths, path)
            continue
        }
        if node.Left != nil {
            nodeQueue = append(nodeQueue, node.Left)
            pathQueue = append(pathQueue, path + "->" + strconv.Itoa(node.Left.Val))
        }
        if node.Right != nil {
            nodeQueue = append(nodeQueue, node.Right)
            pathQueue = append(pathQueue, path + "->" + strconv.Itoa(node.Right.Val))
        }
    }
    return paths
}
```

## 数组中消失的数字
```
func findDisappearedNumbers(nums []int) []int {
    n := len(nums)
    for _, v := range nums {
        v = (v - 1) % n
        nums[v] += n
    }
    for i, v := range nums {
        if v <= n {
            ans = append(ans, i+1)
        }
    }
    return nums
}
```

## 其他
```go
func max(a,b int) int{
    if a>b{
        return a
    }
    return b
}
func abs(a int) int{
    if a < 0 {
        return -a
    }
    return a
}
```
