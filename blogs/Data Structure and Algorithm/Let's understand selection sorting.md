---
title: Let's understand selection sorting
slug: lets-understand-selection-sorting
date: 2026-06-03
author: Tanmoy Saha
tags:
  - series
  - learnings
  - programming
  - dsa
  - cpp
description: Here is all you'll get to know how much DSA I know and what is my learning process.
reading_time: 10
draft: false
series: Data Structure and Algorithm
series_slug: data-structure and-algorithm
series_order: 1
---

# Selection Sorting
Let's understand selection sorting step by step here using this blog. I'm using Striver's A to Z DSA series to learn DSA.

In selection sorting we to sort the numbers we just take the first number and compare it with the lowest number inside the array , once found we swap both number and on the first position we get the lowest number of the array and the we do the name thing with the next numbers till one number before the last number of the array.

Let's sort the below array - 
![[Pasted image 20260606150201.png]]

###### Step 1:
1. We will take the first number as min. For our case `min = 13`.
2. Now we will iterate the array and find the minimum value than our assigned value. 
3. Here in our case the value at index 5 is lower than our assigned value. So, we will swap the value of index 0 with index 5. Then we will get the following result.
![[Pasted image 20260606152244.png]]

###### Step 2:
1. Now the first position of our array has the lowest number of the array and that means the first position is sorted. Now we need to assign our min to the index 1 number. `min = 46`.
2. Now starting from index 1 we will iterate the remaining array again and try to find out the lowest number among the remaining array.
3. Here in our case it's 13 which is at index 5. 
4. Now we will swap our min with the lowest number of the remaining array. That is we will swap the number 46 with number 13.
5. After swapping our array looks like following array.
![[Pasted image 20260606153000.png]]

###### Step 3:
1. Now the second position of our array has the lowest number of the remaining array and that means the second position is sorted. Now we need to assign our min to the index 2 number. `min = 24`.
2. Now starting from index 2 we will iterate the remaining array again and try to find out the lowest number among the remaining array.
3. Here in our case it's 20 which is at index 4. 
4. Now we will swap our min with the lowest number of the remaining array. That is we will swap the number 24 with number 20.
5. After swapping our array looks like following array. 
![[Pasted image 20260606154451.png]]

###### Step 4:
1. Now from index 0 to index 2 is sorted array. Now we need to assign the number of index 3 to min variable. `min = 52`.
2. Now starting from index 3 we will iterate the remaining array again and try to find out the lowest number among the remaining array.
3. Here in our case it's 24 which is at index 4. 
4. Now we will swap our min with the lowest number of the remaining array. That is we will swap the number 52 with number 24.
5. After swapping our array looks like following array. 
![[Pasted image 20260606155010.png|697]]

###### Step 5:
1. We all most completed our swapping. Now from index 0 to index 3 is sorted array. And we need to assign the number of index 4 to min variable. `min = 52`.
2. Now starting from index 4 we will iterate the remaining array again. As we have only 2 numbers left, we will try to find the lowest number from these 2 numbers.
3. Here in our case it's 46 which is at index 5. 
4. Now we will swap our min with the lowest number of the remaining array. That is, we will swap the number 52 with number 46.
5. After swapping our array looks like following array. 
![[Pasted image 20260606155438.png]]

As the remaining one number at last index will always be sorted we need not sort the array more. This is our sorted array - 

![[Pasted image 20260606155539.png]]

#### Short concept explanation - 
1. Swap happened at 0th index and minimum value index. This is from array index **0 to n-1**.
2. Swap happened at 1st index and minimum value index on remaining array. This is from array index **1 to n-1**
3. Swap happened at 2nd index and minimum value index on remaining array. This is from array index **2 to n-1**.
4. Swap happened at 3rd index and minimum value index on remaining array. This is from array index **3 to n-1**
5. Swap happened at 4th index and minimum value index on remaining array. This is from array index **4 to n-1**
6. ...
7. Swap happened at (n-2) index and minimum value index on remaining array. This is from array index **n-2 to n-1**.

