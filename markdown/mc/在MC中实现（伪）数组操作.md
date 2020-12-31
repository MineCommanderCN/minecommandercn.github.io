# 【TCP|skeleton小白】在MC中实现（伪）数组操作

原作者：skeleton小白

<https://www.mcbbs.net/forum.php?mod=viewthread&tid=901171>

转载于：2020/12/31

原作于：2019/8/12

----

__警告：__ __观看此贴前你可能需要一些编程基础，对基本数据结构有所掌握__

众所周知，MC指令~~作为一款功能强大的编程语言~~已经可以实现许多的计算操作，但是想要编程，必然少不了数组的使用。那么，在MC中是否也可以实现数组操作呢？答案是肯定的。
为方便讲解，以下指令全都基于在~ ~ ~处有一个箱子，其中第一格放了一个物品，物品在tag标签下有一个variables专门用于记录各种变量。
（数组与list并不是一个东西，但因为在这里用途相似，所以下文不做区分）

在nbt中有一个很类似于数组的东西，那就是list。我们可以用data指令很方便地修改或获取其中的数值，例如`/data get block ~ ~ ~ Items[0].tag.variables.array[x]` 就可以获得array这个list的第x位的内容。但是，这个x必须在指令中给出，无法使用变量代替（例如计分板分数）。也就是说，我们无法直接使用变量作为下标获得数组的元素，但我们可以通过`array[0]`获得数组的头元素。你想到了什么？没错，这不就是栈么！所以，在访问数组的第x位元素的时候，我们只需要将前x-1个元素从数组头（栈顶）弹出，然后此时的`array[0]`就是我们要的内容了。

[[下载附件：array.zip]](https://www.mcbbs.net/forum.php?mod=attachment&aid=MTM3MzUzNnw3MmQ5ZTZjNXwxNjA5MzgyMzY1fDI5NDQ2NzB8OTAxMTcx)

以下函数均在这个数据包中。加载数据包后先调用函数`array:initialize`，之后的所有函数或指令都站在生成出的箱子上执行，或者在指令最前面加上`/execute at @e[name=database]`（因为我懒得写exe）

## **数组基本操作**

### 读取操作

使用方式：设置index的variables分数为你要获取的数组元素的下标，然后调用array:get函数，此时result的variables分数即为获取的元素。

>`array:get`

```mcfunction
data modify block ~ ~ ~ Items[0].tag.variables.temp set from block ~ ~ ~ Items[0].tag.variables.array
scoreboard players operation i variables = index variables
execute if score i variables > 0 variables run function array:get_loop
execute store result score result variables run data get block ~ ~ ~ Items[0].tag.variables.array[0]
data modify block ~ ~ ~ Items[0].tag.variables.array set from block ~ ~ ~ Items[0].tag.variables.temp
data modify block ~ ~ ~ Items[0].tag.variables.temp set value []
```

>`array:get_loop`

```mcfunction
scoreboard players remove i variables 1
data remove block ~ ~ ~ Items[0].tag.variables.array[0]
execute if score i variables > 0 variables run function array:get_loop
```

原理：先把原数组丢进temp中备份，然后把array[0]删除index次，此时array[0]就是所求的元素。然后再把temp中的备份丢回去

### 写入操作

使用方式：设置index的variables分数为你要写入的数组元素的下标，将想要写入的值放在store的variables分数中，然后调用array:store函数。

>`array:store`

```mcfunction
scoreboard players operation i variables = index variables
execute if score i variables > 0 variables run function array:store_loop
execute store result block ~ ~ ~ Items[0].tag.variables.array[0] int 1 run scoreboard players get store variables
scoreboard players operation i variables = index variables
execute if score i variables > 0 variables run function array:store_return
```

>`array:store_loop`

```mcfunction
scoreboard players remove i variables 1
data modify block ~ ~ ~ Items[0].tag.variables.temp prepend from block ~ ~ ~ Items[0].tag.variables.array[0]
data remove block ~ ~ ~ Items[0].tag.variables.array[0]
execute if score i variables > 0 variables run function array:store_loop
```

>`array:store_return`

```mcfunction
scoreboard players remove i variables 1
data modify block ~ ~ ~ Items[0].tag.variables.array prepend from block ~ ~ ~ Items[0].tag.variables.temp[0]
data remove block ~ ~ ~ Items[0].tag.variables.temp[0]
execute if score i variables > 0 variables run function array:store_return
```

原理：因为直接备份整个数组再还原的话会把写入的值给一起覆盖掉，所以在写入时我们需要换个思路，每一次栈顶弹出的时候我们把这个值丢进另一个栈temp，赋值结束之后再把这个操作反过来，将temp的栈顶弹出并丢回array，重复直到temp为空。

### 随机化

使用方法：调用函数array:randomize

>`array:randomize`

```mcfunction
scoreboard players operation i variables = size variables
data modify block ~ ~ ~ Items[0].tag.variables.array set value []
execute if score i variables > 0 variables run function array:randomize_add
```

>`array:randomize_add`

```mcfunction
scoreboard players remove i variables 1
data modify block ~ ~ ~ Items[0].tag.variables.array prepend value 0
summon area_effect_cloud ~ ~ ~ {CustomName:""random""}
summon area_effect_cloud ~ ~ ~ {CustomName:""random""}
summon area_effect_cloud ~ ~ ~ {CustomName:""random""}
summon area_effect_cloud ~ ~ ~ {CustomName:""random""}
summon area_effect_cloud ~ ~ ~ {CustomName:""random""}
summon area_effect_cloud ~ ~ ~ {CustomName:""random""}
execute store result block ~ ~ ~ Items[0].tag.variables.array[0] int 1 run data get entity @e[limit=1,sort=random,name=random,type=area_effect_cloud] UUIDMost 0.00000000000000001
execute if score i variables > 0 variables run function array:randomize_add
```

原理：清空array并往里面丢size个随机数。这里用了一个非常暴力的伪随机，生成一堆aec并随机取其中一个的UUIDMost的最后两位。（还不是因为我懒得写随机）

学完了数组存取，那么下面当然是排序了。数据包中还写了两个排序，分别是冒泡排序和快速排序。
我们先来看一下简单一点的冒泡排序

## **冒泡排序**

C++代码：

```c++
for (int i=0;i<size;i++) for (int j=0;j<size-i-1;j++) if (a[j]>a[j+1]) swap(a[j],a[j+1]);
```

没错只有一行。但是因为MC运算的复杂性，这里用了四个函数实现了冒泡排序。
使用方法：调用bubble_sort:sort函数

>`bubble_sort:sort`

```mcfunction
scoreboard players set sort_i variables 0
function bubble_sort:loop1
```

>`bubble_sort:loop1`

```mcfunction
scoreboard players set sort_j variables 0
scoreboard players operation stop variables = size variables
scoreboard players remove stop variables 1
scoreboard players operation stop variables -= sort_i variables
execute if score sort_j variables < stop variables run function bubble_sort:loop2
scoreboard players add sort_i variables 1
execute if score sort_i variables < size variables run function bubble_sort:loop1
```

>`bubble_sort:loop2`

```mcfunction
scoreboard players operation index variables = sort_j variables
function array:get
scoreboard players operation x variables = result variables
scoreboard players add index variables 1
function array:get
scoreboard players operation y variables = result variables
execute if score x variables > y variables run function bubble_sort:swap
scoreboard players add sort_j variables 1
execute if score sort_j variables < stop variables run function bubble_sort:loop2
```

>`bubble_sort:swap`

```mcfunction
scoreboard players operation index variables = sort_j variables
scoreboard players operation store variables = y variables
function array:store
scoreboard players add index variables 1
scoreboard players operation store variables = x variables
function array:store
```

对照C++代码就很好理解这个函数。其中，loop1和loop2分别对应代码中的内外两层循环。为避免与存取函数的变量冲突，这里用sort_i和sort_j来表示排序函数中的循环变量。

你已经掌握了基本的数组操作和排序算法，那么现在请尝试实现更高级的排序算法（指快速排序）

## **快速排序**

快速排序是一种基于分治思想的不稳定排序算法，在这里不详细描述其排序过程，不了解的请百度。

C++代码：

```c++
void sort(int l,int r)
{
    int i=l,j=r,mid=a[(l+r)>>1];
    while (i<=j)
    {
        while (a[i]<mid) i++;
        while (a[j]>mid) j--;
            if (i<=j)
            {
                swap(a[i],a[j]);
                i++,j--;
            }
    }
    if (l<j) sort(l,j);
    if (i<r) sort(i,r);
}
```

使用方法：调用quick_sort:sort函数

快速排序的指令实现较为复杂，在这里就不一一列出函数内容，具体详见数据包。循环的实现方式与冒泡排序相同。因为快速排序涉及到递归传值，所以函数中将variables.stack作为一个人工栈使用以模拟实际程序中的系统栈操作，其中的l和r分别表示每一次递归的l和r值。

你已经学会了一维数组，那为什么不尝试增加一个维度呢？

## **二维数组**

array_2d命名空间下包含了二维数组的存取操作。使用前请先调用`array_2d:initialize`函数。与之前不同的是，这个二维数组的每个元素是个二元组，初始时分别表示该元素的行号和列号。因此读写时不能以计分板作为中间变量。
读取操作：设置indexx和indexy的variables分数分别为想要获取的元素的行号和列号，然后调用`array_2d:get`函数，此时`Items[0].tag.variables.result`即为获得的元素。
写入操作：设置indexx和indexy的variables分数分别为想要写入的元素的行号和列号，设置`Items[0].tag.variables.store`的值为需要写入的值，然后调用`array_2d:store`函数。


具体指令详见数据包

原理：与一维数组的存取类似。获取第x行第y列的元素，只需要先将前x-1行弹出，再将第一行的前y-1列弹出就行了。唯一不同的是，在store操作时，因为同一个list中只能存同一种数据类型的数据（不能同时存int和list），所以要开两个栈分别存储弹出的前x-1行（list）和y-1列（int）。

## **效率分析**

![bubble_sort](https://s3.ax1x.com/2020/12/31/rX56sA.png)

![quick_sort](https://s3.ax1x.com/2020/12/31/rX5RdP.png)

**警告：** **请勿轻易尝试类似的大规模排序操作**

前者（冒泡排序）排了将近5分钟，后者（快速排序）排了大概半分钟。（电脑年代久远，可能时间会偏长）

在实际编程中，冒泡排序是个时间复杂度为O(n2)的算法，而快速排序作为不稳定排序，其时间复杂度介于O(nlogn)和O(n2)之间。但因为MC的数组存取操作是个O(n)的操作（最坏时需要进行n次出栈操作），所以在MC中实现的冒泡排序和快速排序时间复杂度分别为O(n3)和O(n2logn)~O(n3)。从时间复杂度来看效率实在不高。其次还要强调的一点是MC中的一条指令并不等同于电脑的一次运算，所以这两个排序的时间复杂度甚至会更高。因此，如果想用MC来编程的话...也就想想吧。你总不希望电脑编程只需要1秒钟就能跑出来的程序你用MC跑整整一天吧...
