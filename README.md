# EEE4775 HW6
**Name**: Andrew VanAuken

## Part A – Spot the race
### Snippet 1:
```c
static int balance = 100;
void deposit(int amount) {
    balance += amount;
}
```

**Race Condition:** Yes. `balance += amount` is not atomic, so simultaneous updates can overwrite each other and produce an incorrect balance.

**Fix:** Protect the update with a mutex or use an atomic increment.
```c
static int balance = 100;
static SemaphoreHandle_t balance_mutex;

void init_balance(void) {
    balance_mutex = xSemaphoreCreateMutex();
}

void deposit(int amount) {
    if (xSemaphoreTake(balance_mutex, portMAX_DELAY) == pdTRUE) {
        balance += amount;
        xSemaphoreGive(balance_mutex);
    }
}
```

### Snippet 2:
```c
static volatile struct sensor_reading_t latest_reading;
void IRAM_ATTR sensor_isr(void) {
    latest_reading.x = read_x();
    latest_reading.y = read_y();
    latest_reading.z = read_z();
    latest_reading.timestamp = esp_timer_get_time();
}
void main_task(void) {
    process(latest_reading);  // wants a coherent snapshot
}
```

**Race Condition:** Yes. The ISR can update `latest_reading` while `main_task()` is reading it, resulting in a partially updated (incoherent) snapshot. `volatile` does not make the structure update atomic.

**Fix:** Use an ISR-safe critical section to protect the structure update and the task’s snapshot copy.
```c
static struct sensor_reading_t latest_reading;
static portMUX_TYPE reading_lock = portMUX_INITIALIZER_UNLOCKED;

void IRAM_ATTR sensor_isr(void) {
    struct sensor_reading_t new_reading;

    new_reading.x = read_x();
    new_reading.y = read_y();
    new_reading.z = read_z();
    new_reading.timestamp = esp_timer_get_time();

    portENTER_CRITICAL_ISR(&reading_lock);
    latest_reading = new_reading;
    portEXIT_CRITICAL_ISR(&reading_lock);
}

void main_task(void) {
    struct sensor_reading_t snapshot;

    portENTER_CRITICAL(&reading_lock);
    snapshot = latest_reading;
    portEXIT_CRITICAL(&reading_lock);

    process(snapshot);
}
```

### Snippet 3:
```c
static SemaphoreHandle_t mux;
void worker(void *p) {
    for (;;) {
        if (xSemaphoreTake(mux, 0) == pdTRUE) {
            update_state();
            // no give
            vTaskDelay(pdMS_TO_TICKS(10));
        }
    }
}
```

**Race Condition:** No traditional data race, but there is a synchronization bug. The mutex is never released, which can permanently block future access.

**Fix:** Release the mutex after `update_state()` and place the delay outside the protected section.
```c
static SemaphoreHandle_t mux;

void worker(void *p) {
    for (;;) {
        if (xSemaphoreTake(mux, portMAX_DELAY) == pdTRUE) {
            update_state();
            xSemaphoreGive(mux);
        }

        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

---

## Part B – Choose your primitive
1. ISR signals a task that "one button press happened"
**Primitive:** Direct task notification

**Justification:** A direct task notification is lightweight and efficiently unblocks one waiting task without requiring a separate semaphore object. It can also count multiple button presses when used with `vTaskNotifyGiveFromISR()` and `ulTaskNotifyTake()`.

2. A single global integer that Core 0 task and Core 1 task both increment
**Primitive:** Atomic operation

**Justification:** The increment operation is a read-modify-write sequence and is not inherently thread-safe. An atomic operation guarantees that each increment completes indivisibly across both cores without the overhead of a mutex.

3. A pool of 4 DMA buffers; producer borrows, consumer returns
**Primitive:** Counting semaphore

**Justification:** A counting semaphore can be initialized to four to represent the number of available buffers. The producer takes one token before borrowing a buffer, and the consumer gives one back when returning it.

4. Wait for all 3 sensor tasks to finish their first read before starting fusion
**Primitive:** Event group

**Justification:** Each sensor task can set its own event bit after completing its first reading. The fusion task can block until all three bits are set, creating a synchronization barrier.

5. A long-running task should be killable from another task
**Primitive:** Direct task notification

**Justification:** Another task can send a stop request through a task notification. The long-running task periodically checks for the notification, safely releases its resources, and then deletes itself, avoiding the risks of forced termination.

---

## Part C – Priority inversion forensics
**Step 1:** Task L (Priority 5) is running and successfully takes `shared_mutex` to write a telemetry log payload.

**Step 2:** Task H (Priority 15) wakes up due to a sensor hardware interrupt. Because H has the highest priority, the scheduler immediately preempts Task L and context-switches to Task H.

**Step 3:** Task H executes its initial processing steps until it hits a section requiring the shared resource. It calls `xSemaphoreTake(shared_mutex, portMAX_DELAY)`.

**Step 4:** Because Task L still holds `shared_mutex`, Task H cannot acquire it. The scheduler places Task H into the Blocked state, waiting for the mutex to clear.

**Step 5:** With Task H blocked, Task L becomes the highest-priority ready task and resumes executing its critical section.

**Step 6:** While Task L is executing inside the critical section, an asynchronous network event wakes up Task M (Priority 10).

**Step 7:** Because Task M's priority (10) is higher than Task L's priority (5), the scheduler immediately preempts Task L and hands execution control to Task M.

**Step 8:** Task M continues executing while Task L remains preempted.

**The Inversion:** Because Task M does not require the shared mutex, it can continue executing while Task L is unable to finish its critical section and release the mutex. Consequently, Task H remains blocked by a medium-priority task, creating a classic unbounded priority inversion.

### Worst-Case Wait for Task H (With Priority Inheritance Enabled)
When priority inheritance is enabled, the worst-case wait for Task H transitions from unbounded to bounded.

The moment Task H attempts to take shared_mutex and blocks (Step 4 above), FreeRTOS temporarily boosts the priority of the mutex owner (Task L) from 5 to 15 (matching Task H).

Because Task L now runs at Priority 15, Task M (Priority 10) can no longer preempt it when it wakes up. Task L executes uninterrupted until it releases the mutex. Upon release, Task L’s priority drops back to 5, and Task H instantly takes the lock and wakes up.

With priority inheritance enabled, the worst-case wait for Task H is bounded by the remaining execution time of Task L's critical section plus the small scheduling overhead required to switch tasks.

Worst-Case Wait = T<sub>L_crit</sub> + T<sub>ctx</sub>

Where:

T<sub>L_crit</sub> = Maximum remaining execution time of Task L's critical section while it owns the mutex.

T<sub>ctx</sub> = FreeRTOS context-switch and scheduling overhead.

Priority inheritance ensures that medium-priority tasks, such as Task M, cannot further delay Task H. As a result, Task H only waits for the mutex-owning task to complete its critical section, making the blocking time predictable and bounded.

---

## Part D – Industry anchor
Priority-inheritance bugs are typically caught before deployment through a combination of static analysis, runtime tracing, and code reviews. Static analysis tools such as Coverity and Klocwork examine source code without executing it and can identify issues such as incorrect mutex usage, missing lock releases, and unsafe synchronization patterns. Runtime tracing tools like Percepio Tracealyzer allow developers to visualize task scheduling, mutex ownership, and blocking events on the target system. By examining execution traces, engineers can quickly identify cases where a high-priority task is blocked while a lower-priority task holds a mutex or where a medium-priority task causes priority inversion. Code reviews also play an important role by verifying that mutexes are used correctly, critical sections remain short, and every lock has a corresponding release. For safety-critical embedded systems, formal verification tools such as TLA+, SPIN, or UPPAAL can model concurrent task behavior and verify properties such as deadlock freedom and bounded blocking before the software is deployed.

### References
https://www.blackduck.com/static-analysis-tools-sast/coverity.html

https://www.perforce.com/products/klocwork

https://percepio.com/tracealyzer/

https://lamport.azurewebsites.net/tla/tla.html

https://spinroot.com/spin/whatispin.html

https://uppaal.org/

---

## AI Usage

ChatGPT was used primarily to help interpret the assignment requirements and improve understanding of synchronization concepts, race conditions, mutexes, semaphores, priority inversion, and FreeRTOS synchronization primitives. ChatGPT also helped with explanations of race-condition fixes, selecting appropriate synchronization mechanisms for different scenarios, and understanding priority inheritance behavior. Google AI was used to research industry practices for detecting concurrency bugs before deployment.
