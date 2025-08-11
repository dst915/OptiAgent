Memory Access Instrumentation Tool for MC3000
This is a lightweight, header-only C instrumentation tool designed for profiling memory access patterns on the MC3000 supercomputing platform. It helps developers understand how their code accesses memory by tracking key metrics such as access frequency, memory strides, and total memory footprint for specific variables.

The tool is designed for performance-critical environments and uses hthread-specific printing functions (hthread_printf, hthread_snprintf) for compatibility with the MC3000's runtime environment.

Features
Variable-Specific Profiling: Instruments code to monitor individual variables.
Memory Footprint: Calculates the total memory range accessed (address_begin to address_end).
Access Counting: Tracks the total number of recorded memory accesses.
Stride Analysis:
Identifies all unique memory access strides (steps between consecutive accesses).
Counts the occurrence of each stride.
Determines the most frequent stride (max_memory_step) and its proportion (max_memory_step_pro).
Platform-Specific I/O: Utilizes hthread library functions for output, ensuring compatibility with the MC3000 platform.
Low Overhead: Implemented with static inline functions to minimize performance impact, allowing for direct inclusion in performance-critical code.
Data Structure: memory_info_t
The core data structure used to store profiling information for a single instrumented variable.

c
typedef struct {
    char function_name[64];       // Name of the function where the variable is instrumented
    char variable_name[64];       // Name of the instrumented variable
    unsigned long address_begin;  // The first memory address accessed
    unsigned long address_end;    // The last memory address accessed
    unsigned long variable_size;  // Total size of the memory region accessed (end - begin)
    unsigned long accessed;       // Total number of times the variable was accessed
    unsigned long memory_step[MEMORY_PRO_MAX_NUM]; // Array to store unique memory strides
    unsigned long memory_step_pro[MEMORY_PRO_MAX_NUM]; // Array to store the frequency of each stride
    unsigned long max_memory_step;  // The most frequent memory stride observed
    float max_memory_step_pro;      // The proportion of the most frequent stride
} memory_info_t;
MEMORY_PRO_MAX_NUM is a compile-time constant (default: 1024) that defines the maximum number of unique strides the tool can track.

API Usage
The tool provides three primary inline functions for instrumentation.

1. start_instrumentation
Initializes the profiling session for a specific variable.

Signature:

c
static inline void start_instrumentation(const char* var_name, void* var_ptr, memory_info_t* mem_info);
var_name: A string representing the name of the variable to be profiled.
var_ptr: A pointer to the beginning of the variable's memory.
mem_info: A pointer to a memory_info_t struct that will store the results.
Usage: Call this function once before the code block you want to profile.

2. record_memory_access
Records a single memory access event.

Signature:

c
static inline void record_memory_access(memory_info_t* mem_info, unsigned long current_addr);
mem_info: The same memory_info_t struct used in start_instrumentation.
current_addr: The memory address that is being accessed.
Usage: Call this function immediately after each memory access you want to track within your loops or critical sections.

3. end_instrumentation
Finalizes the profiling session and computes summary statistics.

Signature:

c
static inline void end_instrumentation(memory_info_t* mem_info);
mem_info: The same memory_info_t struct.
Usage: Call this function once after the profiled code block has finished executing.

Example
Here is a simple example demonstrating how to profile a for loop that accesses an array.

c
#include <stdio.h>
#include "memory_profiler.h" // Assuming the tool is in this header file

#define ARRAY_SIZE 2048

int main() {
    // 1. Declare the data and the profiling struct
    int data[ARRAY_SIZE];
    memory_info_t my_array_info;

    // Set the function name for context
    strncpy(my_array_info.function_name, "main", sizeof(my_array_info.function_name));

    // 2. Start instrumentation before the loop
    start_instrumentation("data", &data[0], &my_array_info);

    // 3. The code block to be profiled
    for (int i = 0; i < ARRAY_SIZE; i += 2) {
        data[i] = i;
        // Record each memory access
        record_memory_access(&my_array_info, (unsigned long)&data[i]);
    }

    // 4. End instrumentation after the loop
    end_instrumentation(&my_array_info);

    // 5. Print the results using the platform-specific printer
    hprint_memory_info(&my_array_info);

    return 0;
}
Expected Output
The hprint_memory_info function will produce output similar to this:

function_name: main

Variable Name: data
Address Begin: 0x7ffc12345000
Address End: 0x7ffc12346ff8
Variable Size: 8184 bytes
Accessed: 1024 times
Max Memory Step: 8
Max Memory Step Proportion: 1.00000000
Explanation of the output:

function_name: The context is the main function.
Variable Name: The profiled array is data.
Address Begin/End: The memory range touched during the loop.
Variable Size: The total bytes between the first and last access.
Accessed: The loop ran 2048 / 2 = 1024 times, so 1024 accesses were recorded.
Max Memory Step: The step is i += 2, and sizeof(int) is 4 bytes on this system, so the stride is 2 * 4 = 8 bytes.
Max Memory Step Proportion: Since every access had the same stride of 8, this stride accounts for 100% of the accesses.
How to Integrate
Include the Header: Add #include "your_header_name.h" to the C/C++ source file you want to instrument.
Declare memory_info_t: Create an instance of the memory_info_t struct for each variable you want to track.
Instrument Your Code: Wrap your critical code sections with start_instrumentation and end_instrumentation. Place record_memory_access calls at the exact points of memory access.
Print Results: Use hprint_memory_info to print the collected data to the console.
Limitations
The tool can track a maximum of MEMORY_PRO_MAX_NUM (1024) unique memory strides. If more unique strides are encountered, they will not be recorded. This limit can be adjusted by changing the #define.
This tool is designed for profiling memory access patterns, not for precise timing analysis.
The accuracy of stride calculation depends on the correct placement of record_memory_access calls.
