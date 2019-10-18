# OS MP1: System Call
###### tags:`OS2019`
---
[toc]

---
:::info
- **GROUP**:Team 36 
- **MEMBER**:
    - 106034061 曾靖渝
    - 106070038 杜葳葳

:::
:::danger
	
Note：

1. [ Modify testcase ]
NachOS-4.0_MP1/code/test/fileIO_test1.c : line12 修正為 if (fid < 0) MSG("Failed on opening file");
NachOS-4.0_MP1/code/test/fileIO_test2.c : line11 修正為 if (fid < 0) MSG("Failed on opening file");

2. [  I/O system calls Fail situation]
如果失敗 return -1 , 會發生失敗的情況 : 我們考慮錯誤使用 fd , size 大小.

- **still success**
:::
## Trace Code

### (a) SC_Halt

#### Purpose & Detail:

- Machine::Run() (machine/mipssim.cc): execute an instruction (leave kernal level program to user level)
    - Machine::OneInstruction: fetch the instruction(PC+4), call **RaiseException()**
    - OneTick(): add time and check if there is any interrupt, if there is interrupt, handle it by changing the mode from user mode to kernal mode
        - **Arguments of system call pass from usermode to kernalmode here**

- Machine::RaiseException() (machine/machine.cc): 
    - transfer control from user mode to kernal mode
    - execute InterruptHandler: call **ExceptionHandler()**
    - when completeing return to user mode

- ExceptionHandler() (userprog/exception.cc): print out the Debug massage and call the the specific functions.
    - call **SysHalt()** (Implementations)

- SysHalt() (userprog/ksyscall.h): Kernel interface for system calls 
    - call **Interrupt::Halt()**

- Interrupt::Halt() (machine/interrupt.cc): Shut down Nachos and print out performance statistics.
    - kernel->stats->Print();




---

### (b) SC_Create
#### Purpose & Detail:

- ExceptionHandler() (userprog/exception.cc): print out the Debug massage and call the the specific functions.
    - **SC_Create**
        - call the **Create()**

- SysCreate()(userprog/ksyscall.h)
    - kernel->fileSystem->Create(filename)

- FileSystem::Create() (filesys/filesys.h):check if the file can be opened.

#### How the arguments of system calls are passed from user program to kernel
- In Machine::RaiseException() (machine.cc), when     **interrupt** happens, change mode from user mode to kernal mode


---
### (c ) SC_PrintInt
#### Purpose & Detail:
- ExceptionHandler():
    - **SC_PrintInt**
        - get the value in read register[4]
        - call  the function **SysPrintInt(val)**
        - print the messages
        - update process counter(+4)
        - store process counter
    - **SC_Create**
        - call the **Create**
        - print out messages.

- SysPrintInt()
    - call the function **PutInt()** in the class **synchConsoleOut**
- synchConsoleOut::PutInt()
    - use a loop to consistently call the **ConsoleOutput::PutChar()**
- synchConsoleOut::PutChar()
    - call the **ConsoleOutput::PutChar()**
    - and wait if necessary
- ConsoleOutput::PutChar()
    - call the  **WriteFile()**
    - call the **Interrupt** **Schedule()**
- Interrupt:: Schudeule()
    - Arrange for the CPU to be interrupted when simulated time reaches "now + when".
    - Implementation: just put it on a sorted list.
    - **toCall** is the object to call when the interrupt occurs
    - **fromNow** is how far in the future (in simulated time) the interrupt is to occur
    - **type** is the hardware device that generated the interrupt
:::warning
- NOTE: the Nachos kernel should not call this routine directly. Instead, it is only called by the hardware device simulators.
:::
- Machine::Run()
    - Simulate the execution of a user-level program on Nachos. 
    - Called by the kernel when the program starts up; never returns.
    - This routine is re-entrant, in that it can be called multiple times concurrently -- one for each thread executing user code.
- Interrupt::OneTick()
    - Occure in two case
        - a user insturction is executed
        - interrupts are re-enable
    - simulat the time of interrupt
- Interrupt::CheclIfDue()
    - check if any interrupt are scheduled to occur
        - **Arguments of system call pass from usermode to kernalmode here**
    
- ConsoleOutput::CallBack()
    - Simulate calls this when the next char can be output to the display
    - First check to make sure character is available.
    - Then invoke the "callBack" registered by whoever wants the character.
- SynchConsoleOutput::CallBack()
    - Interrupt handler called when it's safe to send the next character can be sent to the display.
    


### How the arguments of system calls are passed from user program to kernel
- Case 1:
    - if there is an interrupt happenning, the function **<Machine::OneInstruction(Instruction instr)>** will call the function **In Machine::RaiseException() (machine.cc)**
    - **In Machine::RaiseException() (machine.cc)**, when **interrupt** happens, change mode from user mode to kernal mode
- Case 2:
    - if there are any pending interrupts to be called, call the function "Interrupt::OneTick()" (interrupt.cc)
---

## Implementation
- OpenFileId Open(char *name)
    - test/fileIO_test1.c
    ```c++=
        fid = Open("file1.test");
    ```
    - test/start.S
    ```c++=
        .globl  Open
	    .ent    Open
        Open:
            addiu $2,$0,SC_Open
            syscall
            j	$31
            .end Open
    ```    
    - userprog/syscall.h
    ```c++=
        #define SC_Open	6 
     ```
    - userprog/exception.cc
    ```c++=
        case SC_Open:
            val = kernel->machine->ReadRegister(4);
            {
                char *filename = &(kernel->machine->mainMemory[val]);
                status = SysOpen(filename);
                kernel->machine->WriteRegister(2, (int) status);
            }
            kernel->machine->WriteRegister(PrevPCReg, kernel->machine->ReadRegister(PCReg));
            kernel->machine->WriteRegister(PCReg, kernel->machine->ReadRegister(PCReg) + 4);
            kernel->machine->WriteRegister(NextPCReg, kernel->machine->ReadRegister(PCReg)+4);
            return;
            ASSERTNOTREACHED();
	    break;
    ```
    
    - userprog/ksyscall.h
    ```c++=
        int SysOpen(char *filename){
            // return value is OpenFileId
            // -1: failed
            return kernel->fileSystem->OpenAFile(filename);
        }
    ```
    - filesys/filesys.h
    ```c++=
        OpenFileId OpenAFile(char *name) {
            int fileDescriptor = OpenForReadWrite(name, FALSE);
            if(fileDescriptor != -1){
                OpenFile *file=new OpenFile(fileDescriptor);
                fileDescriptorTable[fileDescriptor % 20] = file;
            }
      	    return fileDescriptor;
        }
    ```

- int Write(char *buffer, int size, OpenFileId id)

    - test/fileIO_test1.c
    ```c++=
        for (i = 0; i < 26; ++i) {
            int count =  Write(test + i, 1, fid);
            if (count != 1) MSG("Failed on writing file");
	    }
    ```
   - test/start.S
   ```c++=
        .globl  Write
	    .ent	Write
        Write:
            addiu $2,$0,SC_Write
            syscall
            j	$31
            .end Write
    ```
    - userprog/syscall.h
    ```c++=
        #define SC_Write 8 
    ```
    - userprog/exception.cc
    ```c++=
        case SC_Write: 
            val = kernel->machine->ReadRegister(4);
            {
                char *buffer = &(kernel->machine->mainMemory[val]);
                int size = kernel->machine->ReadRegister(5);
                int fid = kernel->machine->ReadRegister(6);
                int count = SysWrite(buffer, size, fid);
                DEBUG(dbgTraCode, "### buffer addr:"<<buffer<< " sise:" << size);
                kernel->machine->WriteRegister(2,(int)count);
            }
            kernel->machine->WriteRegister(PrevPCReg, kernel->machine->ReadRegister(PCReg));
            kernel->machine->WriteRegister(PCReg, kernel->machine->ReadRegister(PCReg) + 4);
            kernel->machine->WriteRegister(NextPCReg, kernel->machine->ReadRegister(PCReg)+4);
            return;
            ASSERTNOTREACHED();
        break;
        
    ```

    - userprog/ksyscall.h
    ```c++=
        int SysWrite(char *buffer, int size, int fid){
            return kernel->fileSystem->Write(buffer, size, fid);
        }
    ```
    - filesys/filesys.h
    ```c++=
        int Write(char *buffer, int size, OpenFileId id{
            if(size < 0 ) return -1;
            OpenFile *file = fileDescriptorTable[id % 20];
            return file->Write(buffer, size);	
       }
       
    ```

----
- int Read(char *buffer, int size, OpenFileId id); 
    - test/fileIO_test2.c
    ```c++=
        count = Read(test, 26, fid);
    ```
    - test/start.S
    ```c++=
        Read:
            addiu $2,$0,SC_Read
            syscall
            j	$31
            .end Read
    ```
    - userprog/syscall.h
    ```c++=
        #define SC_Read 9
    ```
    - userprog/exception.cc
    ```c++=
        case SC_Read:
            //Arguments of a system call are in Registers 4, 5, 6 
            val = kernel->machine->ReadRegister(4);
            {
                char *buffer = &(kernel->machine->mainMemory[val]);
                int size = kernel->machine->ReadRegister(5);
                int fid = kernel->machine->ReadRegister(6);
                int count = Read(buffer, size, fid);
                DEBUG(dbgTraCode, "### buffer addr:"<<buffer<< " sise:" << size);
                kernel->machine->WriteRegister(2,(int)count);
            }
            kernel->machine->WriteRegister(PrevPCReg, kernel->machine->ReadRegister(PCReg));
            kernel->machine->WriteRegister(PCReg, kernel->machine->ReadRegister(PCReg) + 4);
            kernel->machine->WriteRegister(NextPCReg, kernel->machine->ReadRegister(PCReg)+4);
            return;
            ASSERTNOTREACHED();
        break;
                        
    ```
    - userprog/ksyscall.h
    ```c++=
        int SysRead(char *buffer, int size, int fid){
            return kernel->fileSystem->Read(buffer, size, fid);
        }
    ```

    - filesys/filesys.h
    ```c++=
        int Read(char *buffer, int size, OpenFileId id){
            if(size < 0 ) return -1;
            OpenFile *file = fileDescriptorTable[id % 20];
            return file->Read(buffer, size);
        }
    ```

- int Close(OpenFileId id); 
    - test/fileIO_test2.c
    ```c++=
        success = Close(fid);
    ```
    - test/start.S
    ```c++=
        Close:
            addiu $2,$0,SC_Close
            syscall
            j	$31
            .end Close
    ```
    - userprog/syscall.h
    ```c++=
        #define SC_Close 10
    ```
    - userprog/exception.cc
    ```c++=
        case SC_Close:
            int fid = kernel->machine->ReadRegister(4);
            SysClose(fid);
            kernel->machine->WriteRegister(PrevPCReg, kernel->machine->ReadRegister(PCReg));
            kernel->machine->WriteRegister(PCReg, kernel->machine->ReadRegister(PCReg) + 4);
            kernel->machine->WriteRegister(NextPCReg, kernel->machine->ReadRegister(PCReg)+4);
            return;
            ASSERTNOTREACHED();
        break;
    ```
    - userprog/ksyscall.h
    ```c++=
        int SysClose(int fid){
            return kernel->fileSystem->Close(fid);
        }
    ```



    - filesys/filesys.h
    ```c++=
       int Close(OpenFileId id){
            OpenFile *file = QfileDescriptorTable[id % 20];
            if(file == NULL) return -1;
            file->~OpenFile();
            FileDescriptorTable[id % 20] = NULL;
            return 1;
        }
    ````
---
## the flow of system call (Write for EX.)
```flow
st=>start:   <Write(test + i, 1, fid)> in (test/fileIOtest.c)
op1=>operation:  <addiu $2,$0,SC_Write> in (test/start.S) 
op2=>operation:  <Machine::Run()> in (machine/mipssim.c) 
op3=>operation:  <OneInstruction(instr)> in (machine/mipssim.c) 
op4=>operation: <RaiseException(SyscallException, 0))> in (machine/mipssim.c) 
op5=>operation: <ExceptionHandler(which))> in (machine/machine.c) 
op6=>operation: <#define SC_Write	8)> in (userprog/syscall.h) 
op7=>operation: <int count = SysWrite(buffer, size, fid)> in (userprog/exception.cc) 
op8=>operation: <kernel->fileSystem->Write(char *buffer, int size, OpenFileId id)> in (userprog/ksyscall.h) 
e=>end: <int Write(char *buffer, int size, OpenFileId id)> in (filesys/filesys.h)
st->op1->op2->op3->op4->op5->op6->op7->op8->e

```

---
## Result
- test1
    - ![](https://i.imgur.com/mkKyjkh.png)


- test2
    - ![](https://i.imgur.com/aqP45fF.png)


---
## Contribution
:::success
- 曾靖渝 50%
    - Read() and Close() implementation

- 杜葳葳 50%
    - Open() and Write() implementation
:::
## Reference
- [System Call Guide](http://people.cs.uchicago.edu/~odonnell/OData/Courses/CS230/NACHOS/code-syscall.html)

---

## Others

- 當 user program 載入後, Fetch > Decode > Execute 的流程如下, 而 ExceptionHandler() 函數則是用來作為 System Call 的 dispatcher
    - ![](https://i.imgur.com/DF4mA66.png)
    - ![](https://i.imgur.com/LKYuwWI.png)
- MIPS instruction set
    - JAL:jump and link
    - SLL:shift left by distance
    - SW:store the word
    - JR:jump to instruction
    - code of main()(halt.c) locate at 272
    - code of Exit(start.s) locate at 256 
    - code of Halt(start.s) locate at 16
    - code of ...() locate at 292








