---
title: "Add ROI to Pin\'s tool pinatrace"
date: 2020-07-22
collection: blog
type: "blog"
permalink: /blog/:title
excerpt_separator: <!--more-->
tags:
  - Tech
  - Tool
---

> I needed to capture the mem trace for one of my projects. Pin has a tool called pinatrace to do that, but the trace dumped is quite large. So I added ROI (region of interest) to it.

<!--more-->

Intel pin has a tool pinatrace, which captures the memory accesses (not the accesses actually goes to mem, any location CPU accesses is captured). The simplest way is to record all mem traces, but the trace file can quickly be very large. Sometimes I only want the trace for certain part of code, where ROI can help.

The code is modified from [Marco Capuccini's answer on stackoverflow](https://stackoverflow.com/questions/32026456/how-can-i-specify-an-area-of-code-to-instrument-it-by-pintool). Following is the original code (formated for compression).

```c
#include <stdio.h>
#include "pin.H"
#include <string>

const CHAR * ROI_BEGIN = "__parsec_roi_begin";
const CHAR * ROI_END = "__parsec_roi_end";

FILE * trace;
bool isROI = false;

// Print a memory read record
VOID RecordMemRead(VOID * ip, VOID * addr, CHAR * rtn){
    // Return if not in ROI
    if(!isROI){ return; }

    // Log memory access in CSV
    fprintf(trace,"%p,R,%p,%s\n", ip, addr, rtn);
}

// Print a memory write record
VOID RecordMemWrite(VOID * ip, VOID * addr, CHAR * rtn){
    // Return if not in ROI
    if(!isROI){ return; }

    // Log memory access in CSV
    fprintf(trace,"%p,W,%p,%s\n", ip, addr, rtn);
}

// Set ROI flag
VOID StartROI(){
    isROI = true;
}

// Set ROI flag
VOID StopROI(){
    isROI = false;
}

// Is called for every instruction and instruments reads and writes
VOID Instruction(INS ins, VOID *v){
    // Instruments memory accesses using a predicated call, i.e.
    // the instrumentation is called iff the instruction will actually be executed.
    //
    // On the IA-32 and Intel(R) 64 architectures conditional moves and REP 
    // prefixed instructions appear as predicated instructions in Pin.
    UINT32 memOperands = INS_MemoryOperandCount(ins);

    // Iterate over each memory operand of the instruction.
    for (UINT32 memOp = 0; memOp < memOperands; memOp++){
        // Get routine name if valid
        const CHAR * name = "invalid";
        if(RTN_Valid(INS_Rtn(ins))){
            name = RTN_Name(INS_Rtn(ins)).c_str();
        }

        if (INS_MemoryOperandIsRead(ins, memOp)){
            INS_InsertPredicatedCall(
                ins, IPOINT_BEFORE, (AFUNPTR)RecordMemRead,
                IARG_INST_PTR,
                IARG_MEMORYOP_EA, memOp,
                IARG_ADDRINT, name,
                IARG_END);
        }
        // Note that in some architectures a single memory operand can be 
        // both read and written (for instance incl (%eax) on IA-32)
        // In that case we instrument it once for read and once for write.
        if (INS_MemoryOperandIsWritten(ins, memOp)){
            INS_InsertPredicatedCall(
                ins, IPOINT_BEFORE, (AFUNPTR)RecordMemWrite,
                IARG_INST_PTR,
                IARG_MEMORYOP_EA, memOp,
                IARG_ADDRINT, name,
                IARG_END);
        }
    }
}

// Pin calls this function every time a new rtn is executed
VOID Routine(RTN rtn, VOID *v){
    // Get routine name
    const CHAR * name = RTN_Name(rtn).c_str();
    printf("Routine Name: %s\n", name);
    // if(strcmp(name,"_GLOBAL__sub_I__Z18__parsec_roi_beginv") == 0) {
    if(strcmp(name,ROI_BEGIN) == 0 || strcmp(name,ROI_BEGIN_CPP) == 0) {
        // Start tracing after ROI begin exec
        RTN_Open(rtn);
        RTN_InsertCall(rtn, IPOINT_AFTER, (AFUNPTR)StartROI, IARG_END);
        RTN_Close(rtn);
    } else if (strcmp(name,ROI_END) == 0 || strcmp(name,ROI_END_CPP) == 0) {
        // Stop tracing before ROI end exec
        RTN_Open(rtn);
        RTN_InsertCall(rtn, IPOINT_BEFORE, (AFUNPTR)StopROI, IARG_END);
        RTN_Close(rtn);
    }
}

// Pin calls this function at the end
VOID Fini(INT32 code, VOID *v){
    fclose(trace);
}

/* ===================================================================== */
/* Print Help Message                                                    */
/* ===================================================================== */

INT32 Usage(){
    PIN_ERROR( "This Pintool prints a trace of memory addresses\n" 
              + KNOB_BASE::StringKnobSummary() + "\n");
    return -1;
}

/* ===================================================================== */
/* Main                                                                  */
/* ===================================================================== */

int main(int argc, char *argv[]){
    // Initialize symbol table code, needed for rtn instrumentation
    PIN_InitSymbols();

    // Usage
    if (PIN_Init(argc, argv)) return Usage();

    // Open trace file and write header
    trace = fopen("roitrace.csv", "w");
    fprintf(trace,"pc,rw,addr,rtn\n");

    // Add instrument functions
    RTN_AddInstrumentFunction(Routine, 0);
    INS_AddInstrumentFunction(Instruction, 0);
    PIN_AddFiniFunction(Fini, 0);

    // Never returns
    PIN_StartProgram();

    return 0;
}
```

The code works fine for instrumenting C code, to use it, add the following line to the code to be instrumented.
```c
// Example c code to be instrumented
#include<stdio.h>

void __parsec_roi_begin(){}
void __parsec_roi_end(){}

int main(){
    int a = 1;
    __parsec_roi_begin();
    for(int i=0; i<500; ++i){
        ++a;
    }
    __parsec_roi_end();
    for(int i=0; i<500; ++i){
        ++a;
    }
    __parsec_roi_end();
    printf("%p\n", &a);
    printf("%i\n", a);
    return 0;
}
```

Now the mem trace is save to file `roitrace.roi`, and when searching for the address of a, 1001 entires are found, which matches with the 500 iteration of incrementing a. if move `__parsec_roi_end();` from line 13 to 16, and search for the address of a again, there will be 2001 entries.

However, this doesn't work for C++, so I printed out the routine name from C and C++ code, the results are as follows:
```bash
# C
Routine Name: __parsec_roi_begin
Routine Name: __parsec_roi_end

# C++
Routine Name: _Z18__parsec_roi_beginv
Routine Name: _Z16__parsec_roi_endv
```
It looks like in C the routine name is simply the function name, while in C++, the routine name has some prefix and postfix added to the function name. so will not be captured by pinatrace tool.

In order to make the tool compatible with C++, I add the varible `ROI_BEGIN_CPP` and `ROI_END_CPP`, and modify the corresponding lines. Recompile the tools (in source code directory, `make obj-intel64/pinatrace.so`) and now it works for both C and C++.

```c
const CHAR * ROI_BEGIN = "__parsec_roi_begin";
const CHAR * ROI_BEGIN_CPP = "_Z18__parsec_roi_beginv";
const CHAR * ROI_END = "__parsec_roi_end";
const CHAR * ROI_END_CPP = "_Z16__parsec_roi_endv";
...
VOID Routine(RTN rtn, VOID *v)
{
    // Get routine name
    const CHAR * name = RTN_Name(rtn).c_str();
    printf("Routine Name: %s\n", name);
    // if(strcmp(name,"_GLOBAL__sub_I__Z18__parsec_roi_beginv") == 0) {
    if(strcmp(name,ROI_BEGIN) == 0 || strcmp(name,ROI_BEGIN_CPP) == 0) {
        // Start tracing after ROI begin exec
        RTN_Open(rtn);
        RTN_InsertCall(rtn, IPOINT_AFTER, (AFUNPTR)StartROI, IARG_END);
        RTN_Close(rtn);
    } else if (strcmp(name,ROI_END) == 0 || strcmp(name,ROI_END_CPP) == 0) {
        // Stop tracing before ROI end exec
        RTN_Open(rtn);
        RTN_InsertCall(rtn, IPOINT_BEFORE, (AFUNPTR)StopROI, IARG_END);
        RTN_Close(rtn);
    }
}
```

This may not be a perfect solution, users have to figure out the routine names of thier functions in C++ by printing them out. I don't know how C++ rename the routine name and the rules behind it, but this solution works for now.

<br>
References:<br>
[Stackoverflow Answer by Marco Capuccini](https://stackoverflow.com/questions/32026456/how-can-i-specify-an-area-of-code-to-instrument-it-by-pintool) <br>

