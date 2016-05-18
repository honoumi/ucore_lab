# 实验七：同步互斥
<br/>
## 练习1: 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题
```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```
用 value 表示当前信号量值，用 waitqueue 指向等待队列

```c
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```
释放资源时调用 up 函数，调用__up。 若等待队列为空，value将直接加一。 若不为空，则唤醒一个等待队列中的线程
```c
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```
请求资源时调用 down 函数，调用 __down。 若 value > 0，则直接获得资源，并把value减一。 否则把当前线程加入到等待队列中，调用 schedule 函数调度其它线程

## 练习2: 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题

