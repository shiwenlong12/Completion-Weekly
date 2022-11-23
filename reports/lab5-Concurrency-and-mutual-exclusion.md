

## 实现过程

1、在syscall/sync.rs中增加相应的函数，来检测当前的死锁检测机制是否已经开启。
    pub fn sys_enable_deadlock_detect(_enabled: usize) -> isize {
        let process = current_process();
        let mut process_inner = process.inner_exclusive_access();
        match _enabled{
            0 => {
                process_inner.deadlock_det = false;
                0
            }
            1 => {
                process_inner.deadlock_det = true;
                0
            }
            _ => -1,
        }
    }

2、在进程控制块中加入死锁检测需要的结构,并且进行初始化和赋值。

    //task/process.rs
    pub struct ProcessControlBlockInner {
        ...
        pub deadlock_det: bool,
        pub available: Vec<i32>,
        pub _available: Vec<i32>,
        pub allocation: Vec<Vec<i32>>,
        pub _allocation: Vec<Vec<i32>>,
        pub need: Vec<Vec<i32>>,
        pub _need: Vec<Vec<i32>>,
        pub id: i32,
    }

                    deadlock_det_enable: false,
                    mutex_available: Vec::new(),
                    semaphore_available: Vec::new(),
                    mutex_allocation: vec![Vec::new(); 1],
                    semaphore_allocation: vec![Vec::new(); 1],
                    mutex_need: vec![Vec::new(); 1],
                    semaphore_need: vec![Vec::new(); 1],
                    id: -1,

    //syscall/thread.rs
    let m1 = process_inner.mutex_list.len();
    let m2 = process_inner.semaphore_list.len();
    process_inner.mutex_need.push(vec![0; m1]);
    process_inner.semaphore_need.push(vec![0; m2]);
    process_inner.mutex_allocation.push(vec![0; m1]);
    process_inner.semaphore_allocation.push(vec![0; m2]);

    sync/mutex.rs
    fn lock(&self) {
        let process = current_process();
        let mut process_inner = process.inner_exclusive_access();
        let id = process_inner.get_id() as usize;
        let task = current_task().unwrap();
        let task_inner = task.inner_exclusive_access();
        let tid = task_inner.res.as_ref().unwrap().tid;
        let mut mutex_inner = self.inner.exclusive_access();
        if mutex_inner.locked {
            mutex_inner.wait_queue.push_back(current_task().unwrap());
            drop(mutex_inner);
            drop(process_inner);
            drop(task_inner);
            block_current_and_run_next();
        } else {
            process_inner.mutex_available[id] -= 1;
            process_inner.mutex_need[tid][id] -= 1;
            process_inner.mutex_allocation[tid][id] += 1;
            mutex_inner.locked = true;
            drop(mutex_inner);
            drop(process_inner);
            drop(task_inner);
        }
    }

    /sync/semaphore.re
    pub fn up(&self) {
        let process = current_process();
        let mut process_inner = process.inner_exclusive_access();
        let id = process_inner.get_id() as usize;
        let task = current_task().unwrap();
        let task_inner = task.inner_exclusive_access();
        let tid = task_inner.res.as_ref().unwrap().tid;
        let mut inner = self.inner.exclusive_access();
        inner.count += 1;
        process_inner.semaphore_allocation[tid][id] -= 1;
        process_inner.semaphore_available[id] += 1;
        drop(process_inner);
        drop(process);
        drop(task_inner);
        drop(task);
        if inner.count <= 0 {
            if let Some(task) = inner.wait_queue.pop_front() {
                add_task(task);
            }
        }
    }

    pub fn down(&self) {
        let process = current_process();
        let mut process_inner = process.inner_exclusive_access();
        let id = process_inner.get_id() as usize;
        let task = current_task().unwrap();
        let task_inner = task.inner_exclusive_access();
        let tid = task_inner.res.as_ref().unwrap().tid;
        let mut inner = self.inner.exclusive_access();
        inner.count -= 1;
        if inner.count < 0 {
            inner.wait_queue.push_back(current_task().unwrap());
            drop(inner);
            drop(process_inner);
            drop(task_inner);
            block_current_and_run_next();
        }
        else {
            process_inner.semaphore_available[id] -= 1;
            process_inner.semaphore_need[tid][id] -= 1;
            process_inner.semaphore_allocation[tid][id] += 1;
            drop(inner);
            drop(process_inner);
            drop(task_inner);
        }
    }

3、在sys_mutex_lock()和sys_semaphore_down()中进行银行家算法检测。
    //syscall/sync.rs
    // LAB5 HINT: Return -0xDEAD if deadlock is detected
    pub fn sys_mutex_lock(mutex_id: usize) -> isize {
        let process = current_process();
        let mut process_inner = process.inner_exclusive_access();
        process_inner.id = mutex_id as i32;
        let task = current_task().unwrap();
        let task_inner = task.inner_exclusive_access();
        let tid = task_inner.res.as_ref().unwrap().tid;
        drop(task_inner);
        drop(task);
        process_inner.mutex_need[tid][mutex_id] += 1;
        if process_inner.deadlock_det {
            if process_inner.deadlock_detect() {
                drop(process_inner);
                drop(process);
                -0xDEAD
            }
            else {
                let mutex = Arc::clone(process_inner.mutex_list[mutex_id].as_ref().unwrap());
                drop(process_inner);
                drop(process);
                mutex.lock();
                0
            }
        }
        else {
            let mutex = Arc::clone(process_inner.mutex_list[mutex_id].as_ref().unwrap());
            drop(process_inner);
            drop(process);
            mutex.lock();
            0
        }
    }

    pub fn sys_semaphore_create(res_count: usize) -> isize {
        let process = current_process();
        let mut process_inner = process.inner_exclusive_access();
        let id = if let Some(id) = process_inner
            .semaphore_list
            .iter()
            .enumerate()
            .find(|(_, item)| item.is_none())
            .map(|(id, _)| id)
        {
            process_inner.semaphore_list[id] = Some(Arc::new(Semaphore::new(res_count)));
            process_inner.semaphore_available[id] = res_count as i32;
            for v in process_inner.semaphore_need.iter_mut() {
                (*v)[id] = 0;
            }
            for v in process_inner._allocation.iter_mut() {
                (*v)[id] = 0;
            }
            id
        } else {
            process_inner
                .semaphore_list
                .push(Some(Arc::new(Semaphore::new(res_count))));
            process_inner.semaphore_available.push(res_count as i32);
            for v in process_inner.semaphore_need.iter_mut() {
                (*v).push(0);
            }
            for v in process_inner._allocation.iter_mut() {
                (*v).push(0);
            }
            process_inner.semaphore_list.len() - 1
        };
        id as isize
    }

4、进行死锁检测。
    task/process.rs
    pub fn deadlock_detect(&self) -> bool{
        let n =  self.thread_count();
        let m1 = self.mutex_available.len();
        let m2 = self.semaphore_available.len();
        //步骤1
        let mut mutex_work: Vec<i32> = self.mutex_available.clone();
        let mut semaphore_work: Vec<i32> = self.semaphore_available.clone();
        let mut Finish: Vec<bool> = vec![false; n];
        let mut find_available: bool = false;
        let mut find_unfinish: bool = false;
        loop {
            find_unfinish = false;
            for i in 0..n {
                if Finish[i] == false {
                    find_unfinish = true;
                    find_available = true;
                    //步骤2
                    for j in 0..m1 {
                        if self.mutex_need[i][j] > mutex_work[j] {
                            find_available = false;
                            break;
                        }
                    }
                    for j in 0..m2 {
                        if self.semaphore_need[i][j] > semaphore_work[j] {
                            find_available = false;
                            break;
                        }
                    }
                    //步骤3
                    if find_available {
                        for j in 0..m1 {
                            mutex_work[j] = mutex_work[j] + self.mutex_allocation[i][j];
                        }
                        for j in 0..m2 {
                            semaphore_work[j] = semaphore_work[j] + self.semaphore_allocation[i][j];
                        }
                        Finish[i] = true;
                        break;
                    }
                }
            }
            步骤4
            if find_unfinish == false {
                println!("detect no dead");
                return false;
            }
            if find_available == false {
                println!("detect dead");
                return true;
            }
        }
    }


  