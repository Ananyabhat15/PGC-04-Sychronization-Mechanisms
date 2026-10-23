# Experiment: Analyze Synchronization Mechanisms in Parallel Programs

## Objective

To implement and study common synchronization mechanisms used in shared-memory parallel programs - an unsynchronized (race) update, critical sections, atomic operations, barriers, and semaphores/locks - and to evaluate their correctness, execution time, and scalability as the number of threads increases.

---

## Software Requirements

| S.No. | Software / Tool | Purpose |
|-------|-----------------|---------|
| 1 | VMware Workstation Pro | Virtualization software to host the Ubuntu guest OS |
| 2 | Ubuntu 22.04 LTS | Guest operating system used for compiling and running the programs |
| 3 | GCC Compiler (build-essential) | Compiles the C programs |
| 4 | OpenMP (libgomp, bundled with GCC) | Provides `#pragma omp` directives for parallel loops, critical, atomic, barrier |
| 5 | POSIX Threads (pthread) & Semaphore library | Used by the semaphore-based program |
| 6 | Ubuntu Terminal | To compile and execute all programs and record timing results |

---
## Installation Steps (From Scratch)

1 Install VMware Workstation Pro
2 Create a New Virtual Machine
Open VMware Workstation and click **"Create a New Virtual Machine"**.
Select **"Typical (recommended)"** configuration and click Next.
Browse to the Ubuntu 22.04 LTS ISO file (downloaded from ubuntu.com) as the installer disc image.
Enter a VM name and choose a storage location on the host disk.
Allocate disk space (20 GB or more recommended).
Under **Customize Hardware**, allocate at least 4 CPU cores and 2-4 GB RAM (multiple cores are needed to see real scalability differences).
Click **Finish** to create the VM.
3 Install Ubuntu 22.04 LTS
4 Install GCC and OpenMP


## Programs and Compilation


### Program 1 - Race Condition (No Synchronization)

**File:** `race.c`

```c
#include <stdio.h>
#include <omp.h>
int main(){
 long counter=0;
 #pragma omp parallel for
 for(long i=0;i<10000000;i++) counter++;
 printf("Counter=%ld\n",counter);
}
```

**Compile:**

```bash
gcc race.c -fopenmp -o race
```

**Run:**

```bash
./race
```

**Screenshot - Program 1 Output:**

<img width="1600" height="633" alt="WhatsApp Image 2026-10-23 at 10 25 42 AM" src="https://github.com/user-attachments/assets/28e3f984-bf4b-407f-b9e4-a2f9827baa10" />


---


### Program 2 - Critical Section

**File:** `critical.c`

```c
#include <stdio.h>
#include <omp.h>
int main(){
 long counter=0;
 #pragma omp parallel for
 for(long i=0;i<10000000;i++){
 #pragma omp critical
 {counter++;}
 }
 printf("Counter=%ld\n",counter);
}
```

**Compile:**

```bash
gcc critical.c -fopenmp -o critical
```

**Run:**

```bash
./critical
```

**Screenshot - Program 2 Output:**
<img width="786" height="287" alt="WhatsApp Image 2026-10-23 at 10 25 41 AM (1)" src="https://github.com/user-attachments/assets/f46076a5-fccd-4428-a2eb-b18150cc5578" />

---

### Program 3 - Atomic Operation

**File:** `atomic.c`

```c
#include <stdio.h>
#include <omp.h>
int main(){
 long counter=0;
 #pragma omp parallel for
 for(long i=0;i<10000000;i++){
 #pragma omp atomic
 counter++;
 }
 printf("Counter=%ld\n",counter);
}
```

**Compile:**

```bash
gcc atomic.c -fopenmp -o atomic
```

**Run:**

```bash
./atomic
```

**Screenshot - Program 3 Output:**

<img width="749" height="301" alt="WhatsApp Image 2026-10-23 at 10 25 42 AM (1)" src="https://github.com/user-attachments/assets/9a447891-1c9a-41fc-9b06-ab15955ff150" />


---

### Program 4 - Barrier

**File:** `barrier.c`

```c
#include <stdio.h>
#include <omp.h>
int main(){
 #pragma omp parallel
 {
 printf("Thread %d before barrier\n",omp_get_thread_num());
 #pragma omp barrier
 printf("Thread %d after barrier\n",omp_get_thread_num());
 }
}
```

**Compile:**

```bash
gcc barrier.c -fopenmp -o barrier
```

**Run:**

```bash
./barrier
```

**Screenshot - Program 4 Output:**

<img width="786" height="567" alt="WhatsApp Image 2026-10-23 at 10 25 41 AM" src="https://github.com/user-attachments/assets/4f5131cc-562e-43a8-bd52-63c1a1023ae0" />

---

### Program 5 - Semaphore

**File:** `semaphore.c`

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
sem_t semaphore;
int counter=0;
void* work(void* a){
 for(int i=0;i<100000;i++){
 sem_wait(&semaphore);
 counter++;
 sem_post(&semaphore);
 }
 return NULL;
}
int main(){
 pthread_t t1,t2;
 sem_init(&semaphore,0,1);
 pthread_create(&t1,NULL,work,NULL);
 pthread_create(&t2,NULL,work,NULL);
 pthread_join(t1,NULL);
 pthread_join(t2,NULL);
 printf("Counter=%d\n",counter);
 sem_destroy(&semaphore);
}
```

**Compile:**

```bash
gcc semaphore.c -pthread -o semaphore
```

**Run:**

```bash
./semaphore
```

**Screenshot - Program 5 Output:**

<img width="786" height="512" alt="WhatsApp Image 2026-10-23 at 10 25 40 AM (2)" src="https://github.com/user-attachments/assets/abbe0f3e-7979-4276-9ae0-a4e05c0740a2" />


---

## Comparison Program - Scalability & Execution Time

Program 6 below combines the None, Critical, Atomic, and Lock strategies into a single benchmark. It repeats the same 10,000,000-increment workload for each mechanism across 1, 2, 4, and 8 threads and checking the final counter value against the expected result. This single program is what produces the correctness/performance/scalability data used in Section 7.

### 6.1 Program 6 - Comparison Benchmark

**File:** `compare_sync.c`

```c
/* compare_sync.c
* Compile: gcc compare_sync.c -fopenmp -o compare_sync
* Run: ./compare_sync
*/
#include <stdio.h>
#include <omp.h>
#define N 10000000L
long run_none(int nthreads) {
 long counter = 0;
 omp_set_num_threads(nthreads);
 #pragma omp parallel for
 for (long i = 0; i < N; i++) {
 counter++;
 }
 return counter;
}
long run_critical(int nthreads) {
 long counter = 0;
 omp_set_num_threads(nthreads);
 #pragma omp parallel for
 for (long i = 0; i < N; i++) {
 #pragma omp critical
 {
 counter++;
 }
 }
 return counter;
}
long run_atomic(int nthreads) {
 long counter = 0;
 omp_set_num_threads(nthreads);
 #pragma omp parallel for
 for (long i = 0; i < N; i++) {
 #pragma omp atomic
 counter++;
 }
 return counter;
}
long run_lock(int nthreads) {
 long counter = 0;
 omp_lock_t lock; omp_init_lock(&lock);
 omp_set_num_threads(nthreads);
 #pragma omp parallel for
 for (long i = 0; i < N; i++) {
 omp_set_lock(&lock);
 counter++;
 omp_unset_lock(&lock);
 }
 omp_destroy_lock(&lock);
 return counter;
}
void run_and_report(const char *name, long (*fn)(int), int nthreads) {
 double start = omp_get_wtime();
 long result = fn(nthreads);
 double elapsed = omp_get_wtime() - start;
 const char *correctness = (result == N) ? "Correct" : "INCORRECT";
 printf("%-10s | %7d | %12ld | %10s | %10.4f\n",
 name, nthreads, result, correctness, elapsed);
}
int main(void) {
 int thread_counts[] = {1, 2, 4, 8};
 int num_configs = sizeof(thread_counts) / sizeof(thread_counts[0]);
 printf("Comparing synchronization mechanisms for %ld shared increments\n\n", N);
 printf("Mechanism | Threads | Final Count | Correctness | Time (s)\n");
 printf("---------------------------------------------------------------\n");
 for (int i = 0; i < num_configs; i++) {
 int t = thread_counts[i];
 run_and_report("None", run_none, t);
 run_and_report("Critical", run_critical, t);
 run_and_report("Atomic", run_atomic, t);
 run_and_report("Lock", run_lock, t);
 printf("---------------------------------------------------------------\n");
 }
 printf("\nExpected correct final count = %ld\n", N);
 printf("Observation: 'None' is fastest but incorrect (race condition).\n");
 printf("'Critical' is correct but slowest (coarse-grained locking).\n");
 printf("'Atomic' and 'Lock' are correct; 'Atomic' scales better since it\n");
 printf("uses lightweight hardware instructions instead of OS-level locks.\n");
 return 0;
}
```

**Compile:**

```bash
gcc compare_sync.c -fopenmp -o compare_sync
```

**Run:**

```bash
./compare_sync
```

**Screenshot - Program 6 Output:**

<img width="786" height="568" alt="WhatsApp Image 2026-10-23 at 10 25 40 AM (1)" src="https://github.com/user-attachments/assets/70b8121e-1f4d-4530-af76-b138ac1b6d2f" />


---
### Final Results Table

| Mechanism | Threads | Final Count | Correctness | Time (s) |
|-----------|---------|-------------|-------------|----------|
| None      | 1       | 10000000    | Correct     | 0.0069   |
| Critical  | 1       | 10000000    | Correct     | 0.1150   |
| Atomic    | 1       | 10000000    | Correct     | 0.0663   |
| Lock      | 1       | 10000000    | Correct     | 0.1131   |
| None      | 2       | 5601116     | INCORRECT   | 0.0097   |
| Critical  | 2       | 10000000    | Correct     | 0.3570   |
| Atomic    | 2       | 10000000    | Correct     | 0.1624   |
| Lock      | 2       | 10000000    | Correct     | 0.4419   |
| None      | 4       | 3204969     | INCORRECT   | 0.0120   |
| Critical  | 4       | 10000000    | Correct     | 0.7121   |
| Atomic    | 4       | 10000000    | Correct     | 0.1744   |
| Lock      | 4       | 10000000    | Correct     | 0.9370   |
| None      | 8       | 2004678     | INCORRECT   | 0.0136   |
| Critical  | 8       | 10000000    | Correct     | 0.6481   |
| Atomic    | 8       | 10000000    | Correct     | 0.1858   |
| Lock      | 8       | 10000000    | Correct     | 0.8688   |

Expected correct final count = `10000000`

### Execution Time Summary (seconds) - Mechanism vs. Threads

| Mechanism | 1 Thread | 2 Threads | 4 Threads | 8 Threads |
|-----------|----------|-----------|-----------|-----------|
| None      | 0.0069   | 0.0097    | 0.0120    | 0.0136    |
| Critical  | 0.1150   | 0.3570    | 0.7121    | 0.6481    |
| Atomic    | 0.0663   | 0.1624    | 0.1744    | 0.1858    |
| Lock      | 0.1131   | 0.4419    | 0.9370    | 0.8688    |

### Observation

- `None` is the fastest at every thread count, but it is incorrect for 2, 4 and 8 threads. The final count drops from 10,000,000 to 5,601,116, 3,204,969 and 2,004,678 as threads increase, so the lost updates get worse with more threads.
- `Critical` and `Lock` are always correct, but their time rises sharply with threads (about 6x for Critical and 8x for Lock from 1 to 4 threads), because every thread waits for the same lock.
- `Lock` is the slowest mechanism at 2, 4 and 8 threads, slightly slower than `Critical`.
- `Atomic` is correct and the best of the correct mechanisms. At 4 and 8 threads it is roughly 3.5-5x faster than `Critical` and `Lock`.
- No mechanism got faster with more threads, since every thread updates one shared counter.

---

## Conclusion

Atomic operations provide the best balance of correctness and performance for simple shared-variable updates because they rely on lightweight hardware instructions. Critical sections and locks guarantee correctness but limit scalability since threads must wait in turn to enter the protected region. Barriers do not protect data but synchronize the phases of a parallel computation, at the cost of waiting for the slowest thread. Semaphores generalize mutual exclusion (and can allow a controlled number of threads into a region) at a moderate overhead. The comparison benchmark confirms that unsynchronized access is fastest but unreliable, while the choice among critical/atomic/lock/semaphore is a trade-off between strength of guarantee, overhead, and scalability as thread count grows.
