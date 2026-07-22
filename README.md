class Solution:
    # @param A : list of integers
    # @return a list of integers
    def nextGreater(self, A):
        n = len(A)
        # Initialize the result array with -1 (default for no greater element)
        ans = [-1] * n 
        stack = []
        
        # Traverse the array from right to left
        for i in range(n - 1, -1, -1):
            
            # Pop elements from the stack that are smaller than or equal to A[i].
            # They can never be the "next greater" element for A[i] or any element 
            # to the left of A[i] because A[i] is larger and closer to them.
            while stack and stack[-1] <= A[i]:
                stack.pop()
            
            # If the stack is not empty after popping, the top element is the 
            # first strictly greater element to the right.
            if stack:
                ans[i] = stack[-1]
            
            # Push the current element onto the stack to act as a potential 
            # "next greater" candidate for the elements to its left.
            stack.append(A[i])
            
        return ans
