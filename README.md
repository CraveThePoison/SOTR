# SOTR
Let’s break this down into four parts:

Minimal FreeRTOS “Hello World” (main.c)

Industry-style demo with an intentional leak

Tools & techniques for finding leaks

How to wire up those tools on S32K358/GCC

1. A Minimal FreeRTOS Main
Below is a pared-down main.c that runs two simple tasks on S32K358 using heap_4.c (the thread-safe, coalescing allocator) and the GCC toolchain. One task just toggles a GPIO (e.g. an LED); the other prints a tick count to a UART.

c
Copy
Edit
/* main.c */
#include "FreeRTOS.h"
#include "task.h"
#include "board_io.h"    // your GPIO/UART init wrappers

/* Task prototypes */
static void vLEDTask(void *pvParameters);
static void vUartTask(void *pvParameters);

int main(void)
{
    board_init();  // init clocks, GPIO, UART

    /* Create two tasks. */
    xTaskCreate(vLEDTask, "LED", configMINIMAL_STACK_SIZE, NULL, tskIDLE_PRIORITY + 1, NULL);
    xTaskCreate(vUartTask, "UART", configMINIMAL_STACK_SIZE * 2, NULL, tskIDLE_PRIORITY + 1, NULL);

    /* Start the scheduler. Never returns. */
    vTaskStartScheduler();

    /* Should not reach here */
    for(;;);
}

/* Blink an LED every 500ms */
static void vLEDTask(void *pvParameters)
{
    (void)pvParameters;
    for (;;)
    {
        board_toggle_led();
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

/* Print a counter on UART every second */
static void vUartTask(void *pvParameters)
{
    uint32_t counter = 0;
    (void)pvParameters;
    for (;;)
    {
        board_uart_printf("Tick: %lu\r\n", counter++);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
Key points

Uses heap_4.c by adding it to your FreeRTOS source.

configMINIMAL_STACK_SIZE is defined in FreeRTOSConfig.h.

board_io.h wraps your S32K358 peripheral init (clock, PORT, UART, etc.).

2. Industry-Style Demo with an Intentional Leak
Imagine you have a worker task that processes incoming messages by dynamically allocating a buffer—but never frees it if an error occurs. This simulates a “real” leak.

c
Copy
Edit
/* leak_demo.c */
#include "FreeRTOS.h"
#include "task.h"
#include <string.h>

static void vLeakyTask(void *pvParameters)
{
    for (;;)
    {
        char *pBuffer = pvPortMalloc(256);
        if (pBuffer == NULL)
        {
            /* Oops—allocation failed */
            continue;
        }

        /* Simulate reading a message into the buffer */
        if (receive_uart_message(pBuffer, 256) != 0)
        {
            /* Error path: we forget to free! */
            // vPortFree(pBuffer);  <-- intentionally missing
        }
        else
        {
            /* Process message then free */
            process_message(pBuffer);
            vPortFree(pBuffer);
        }

        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

// In main()
xTaskCreate(vLeakyTask, "Leaky", configMINIMAL_STACK_SIZE * 2, NULL, 2, NULL);
Over time, each error branch eats 256 bytes of heap.

3. Tools & Techniques to Find Embedded Leaks
a) Heap Usage Stats in FreeRTOS
uxTaskGetSystemState() + xPortGetFreeHeapSize(): at runtime you can snapshot free‐heap.

configUSE_MALLOC_FAILED_HOOK: you’ll catch out-of-memory and can log a dump.

b) Custom Malloc/Free Wrappers
Wrap pvPortMalloc/vPortFree to keep a table of live allocations:

c
Copy
Edit
#define pvPortMalloc(size)   tracked_malloc(size, __FILE__, __LINE__)
#define vPortFree(ptr)       tracked_free(ptr)

/* Internals: */
void *tracked_malloc(size_t size, const char *file, int line)
{
    void *p = heap_4_malloc(size);
    if (p) record_allocation(p, size, file, line);
    return p;
}

void tracked_free(void *p)
{
    record_free(p);
    heap_4_free(p);
}
At key points (e.g. every minute or on command), walk your allocation table and print any pointers still live, along with file:line.

c) Static Analysis
Cppcheck, PC-Lint, or Coverity Scan will flag mismatched malloc/free.

Integrate into your CI so every new code check runs the analyzer.

d) Host-Side Simulation with Valgrind
Compile your RTOS code for a POSIX target (stub out hardware calls).

Run under Valgrind or AddressSanitizer to catch leaks before flashing.

e) RTOS Trace & SystemView
Tools like Percepio Tracealyzer or SEGGER SystemView can monitor heap calls live and histogram allocations.

4. Putting It All Together on S32K358/GCC
FreeRTOS Setup

Copy FreeRTOS kernel + portable/GCC/ARM_CM4F + heap_4.c into your project.

Configure FreeRTOSConfig.h for your clock rate, priorities, and enable:

c
Copy
Edit
#define configUSE_MALLOC_FAILED_HOOK    1
#define configUSE_TRACE_FACILITY        1
#define configUSE_STATS_FORMATTING_FUNCTIONS 1
Instrumenting Malloc

Add your tracked_malloc/tracked_free to a new file memtrace.c.

Provide a console command (or trigger via UART) that dumps current heap usage:

c
Copy
Edit
void vMemReport(void)
{
    printf("Free heap: %u bytes\r\n", xPortGetFreeHeapSize());
    dump_allocation_table();
}
CI & Static Analysis

Install Cppcheck (sudo apt install cppcheck), add to your Makefile:

makefile
Copy
Edit
cppcheck --enable=warning,performance,portability --std=c99 src/
Fail the build if new leaks are introduced.

Optional Host Simulation

Create a posix/ build of your code (stub out board_io), then:

bash
Copy
Edit
gcc -g -fsanitize=address -o rtos_sim posix/main.c memtrace.c ... -lpthread
./rtos_sim  # Valgrind/ASAN will catch leaks
On-Target Verification

Flash onto S32K358, open your UART terminal, and invoke vMemReport() periodically.

Watch the “Free heap” drop if leaks occur, and inspect the live allocation table to pinpoint file:line that never freed.

Next Steps for You
Compile and run the minimal demo above to verify FreeRTOS is ticking.

Add the leaky demo and watch your heap shrink in logs.

Implement the malloc/free wrappers and allocation table to pinpoint exactly where leaks happen.

Integrate Cppcheck into your workflow so code doesn’t drift.

Feel free to ask for clarification on any step—building and debugging an RTOS is a big first project, but with incremental testing and these tools you’ll have full visibility into your heap!
/* leak_demo.c */
#include "FreeRTOS.h"
#include "task.h"
#include <string.h>

static void vLeakyTask(void *pvParameters)
{
    for (;;)
    {
        char *pBuffer = pvPortMalloc(256);
        if (pBuffer == NULL)
        {
            /* Oops—allocation failed */
            continue;
        }

        /* Simulate reading a message into the buffer */
        if (receive_uart_message(pBuffer, 256) != 0)
        {
            /* Error path: we forget to free! */
            // vPortFree(pBuffer);  <-- intentionally missing
        }
        else
        {
            /* Process message then free */
            process_message(pBuffer);
            vPortFree(pBuffer);
        }

        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

// In main()
xTaskCreate(vLeakyTask, "Leaky", configMINIMAL_STACK_SIZE * 2, NULL, 2, NULL);

