# rCore-Camp-2025s ch3报告

## 总结功能

实现一个新的系统调用：
```rust
fn sys_trace(_trace_request: usize, _id: usize, _data: usize) -> isize
```

根据 `_trace_request` 的值完成不同功能：

1. 读取 `_id` 地址处的值
2. 将 `_data` 写入 `_id` 地址处
3. 查询当前任务调用编号为 `_id` 的系统调用的次数

### 实现思路

1. 在 `TaskControlBlock` 中添加 `syscall_times` 数组，用于记录每个系统调用的次数：
```rust
pub struct TaskControlBlock {
    pub task_status: TaskStatus,
    pub task_cx: TaskContext,
    pub syscall_times: [u32; 500], // 支持最多500个系统调用ID
}
```

2. 在 `syscall` 函数中统一处理系统调用计数：
```rust
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    // 在系统调用处理前增加计数
    let mut inner = TASK_MANAGER.inner.exclusive_access();
    let current = inner.current_task;
    inner.tasks[current].syscall_times[syscall_id] += 1;
    drop(inner);
    
    // 处理具体的系统调用
    match syscall_id {
        // ...
    }
}
```

3. 在 `sys_trace` 中实现三种功能：
```rust
pub fn sys_trace(trace_request: usize, id: usize, data: usize) -> isize {
    match trace_request {
        0 => unsafe { *(id as *const u8) as isize },  // 读取内存
        1 => unsafe { *(id as *mut u8) = data as u8; 0 },  // 写入内存
        2 => {  // 查询系统调用次数
            let inner = TASK_MANAGER.inner.exclusive_access();
            inner.tasks[inner.current_task].syscall_times[id] as isize
        }
        _ => -1
    }
}
```

## 简答作业

### 1. 正确进入 U 态后，程序的特征

运行三个 bad 测例 (ch2b_bad_*.rs) 的结果：
```
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
```

错误原因：
1. 第一个错误：尝试写 0 地址，触发 PageFault
2. 第二、三个错误：在 U 态执行需要 S 态权限的指令（sret 和 csrr），触发 IllegalInstruction

### 2. 深入理解 trap.S

#### 1. L40：__restore 的 sp 和两种使用情景

- sp 代表内核栈栈顶
- 两种使用情景：
  1. 开始运行第一个用户程序时
  2. trap 处理完毕返回用户程序时

#### 2. L43-L48：特殊处理的寄存器

处理了三个 CSR 寄存器：
- sstatus：记录 trap 发生前的特权级
- sepc：记录 trap 发生前的指令地址
- sscratch：用于暂存栈指针

#### 3. L50-L56：跳过 x2 和 x4 的原因

- x2 (sp)：由专门的代码维护
- x4 (tp)：线程寄存器，本实验未使用

#### 4. L60：sp 和 sscratch 的值

- sp：指向用户栈
- sscratch：指向内核栈

#### 5. 状态切换

- 切换指令：sret
- 切换机制：
  1. 根据 sstatus.SPP 设置特权级
  2. 跳转到 sepc 指向的指令

#### 6. L13：sp 和 sscratch 的值

- sp：指向内核栈顶
- sscratch：指向用户栈顶

#### 7. U 态到 S 态的切换

通过 ecall 指令触发，用于：
1. 调用 SBI
2. 执行会引发异常的指令

## 荣誉准则

1. 在完成本次实验的过程中，我曾与以下同学交流：
   - mingyang91

2. 参考资料：
   - 问cursor

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。

4. 我从未使用过他人的代码，也未曾向他人复制或公开我的实验代码。我提交的代码均无意于破坏或妨碍任何计算机系统的正常运转。