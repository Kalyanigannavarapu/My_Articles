# Interrupts Explained: From Hardware Signal to Your Handler

**Weekly Post · 22 November 2026**
**Platform focus:** ARM Cortex-M (NVIC)

---

## Contents

1. [What is an interrupt?](#1-what-is-an-interrupt)
2. [Foreground and background architecture](#2-foreground-and-background-architecture)
3. [The vector table](#3-the-vector-table)
4. [CPU modes, privilege and stacks](#4-cpu-modes-privilege-and-stacks)
5. [How the NVIC works](#5-how-the-nvic-works)
6. [What happens when an interrupt is triggered](#6-what-happens-when-an-interrupt-is-triggered)
7. [How the program counter moves between background and foreground](#7-how-the-program-counter-moves-between-background-and-foreground)
8. [Multiple interrupts at the same time](#8-multiple-interrupts-at-the-same-time)
9. [Interrupt latency](#9-interrupt-latency)
10. [Rules for writing an interrupt handler](#10-rules-for-writing-an-interrupt-handler)
11. [Extra topics worth knowing](#11-extra-topics-worth-knowing)
12. [Summary](#12-summary)

---

## 1. What is an interrupt?

> **Definition:** An interrupt is a hardware signal that makes the CPU **pause the code it is running**, execute a special function called an **Interrupt Service Routine (ISR)**, and then **resume the paused code from the exact point where it stopped**.

**Analogy:** You are reading a book (main program). The doorbell rings (interrupt). You put in a bookmark (save context), answer the door (ISR), then return to the same page (restore context).

### Key terms

| Term | Precise meaning |
|---|---|
| **Interrupt request (IRQ)** | The signal a peripheral raises to ask for the CPU's attention |
| **ISR / Interrupt handler** | The function the CPU runs in response to an interrupt |
| **Exception** | Any event that diverts the CPU from normal flow: reset, faults, SVC, PendSV, SysTick, and interrupts |
| **Interrupt** | An exception caused by an **external** source (a peripheral or pin). IRQ numbers start from 0 |
| **Context** | The CPU register values needed to resume the interrupted code |
| **Pending** | The request has arrived but the CPU has not started serving it yet |
| **Active** | The CPU is currently executing its ISR |

### Polling vs. interrupt

| | Polling | Interrupt |
|---|---|---|
| How it works | CPU repeatedly checks a status flag | Hardware notifies the CPU when the event occurs |
| CPU usage | Wastes cycles while waiting | CPU is free (or asleep) until needed |
| Response time | Depends on how often the loop checks | Fast and bounded |
| Power | CPU must stay awake | CPU can sleep with `WFI` |
| Complexity | Simple | Needs care with shared data and priorities |

### Common interrupt sources
Timer overflow or compare, UART byte received, GPIO pin edge (button), ADC conversion done, DMA transfer complete, SysTick, and fault conditions.

---

## 2. Foreground and background architecture

> **Definition:** A foreground/background system (also called a *super-loop*) splits firmware into two layers. The **background** is the infinite `main()` loop doing non-urgent work. The **foreground** is the set of ISRs doing urgent, short work whenever events occur.

```
        BACKGROUND                       FOREGROUND
   (main loop, Thread mode)         (ISRs, Handler mode)

   while (1) {                       Timer ISR   ──┐
      read_sensors();                UART ISR    ──┼── interrupt the loop
      process_data();                Button ISR  ──┘   whenever needed
      update_display();
   }
```

| | Background | Foreground |
|---|---|---|
| Code | `main()` loop | ISRs |
| Triggered by | Runs continuously | Hardware events |
| Priority | Lowest (can be interrupted) | Higher (can interrupt background) |
| Work type | Long, heavy, non-urgent | Short, urgent |
| CPU mode | Thread mode | Handler mode |

**How they cooperate:** the ISR does the minimum (grab the data, clear the flag, set a flag), and the background loop does the heavy processing later.

```c
volatile uint8_t rx_byte;
volatile bool    rx_ready = false;

void USART1_IRQHandler(void)      // foreground
{
    rx_byte  = USART1->DR;        // read data (also clears the flag)
    rx_ready = true;              // notify background
}

int main(void)                    // background
{
    init();
    while (1) {
        if (rx_ready) {
            rx_ready = false;
            handle_byte(rx_byte); // heavy work happens here
        }
    }
}
```

---

## 3. The vector table

> **Definition:** The vector table is an array of 32-bit addresses stored in memory. Entry *N* holds the address of the handler for exception number *N*. It lets the CPU find the right ISR in hardware, without any software lookup.

Default location: address `0x00000000` (start of Flash).

| Offset | Content |
|---|---|
| `0x00` | Initial Main Stack Pointer (MSP) value |
| `0x04` | Reset handler |
| `0x08` | NMI handler |
| `0x0C` | HardFault handler |
| ... | Other system exceptions (SVC, PendSV, SysTick) |
| `0x40` onward | External interrupts IRQ0, IRQ1, IRQ2 ... |

For external interrupt *N*, the CPU reads the handler address from:

```
vector_table_base + 4 × (16 + N)
```

The table's location can be changed by writing the **VTOR** register. This is how bootloaders hand control to an application, and how handlers can be placed in fast RAM.

---

## 4. CPU modes, privilege and stacks

Understanding these first makes the interrupt sequence easy to follow.

> **Definition:** Cortex-M has two **modes** (Thread and Handler) that say *what kind of code is running*, and two **privilege levels** (Privileged and Unprivileged) that say *what the code is allowed to do*.

| | Privileged | Unprivileged |
|---|---|---|
| **Thread mode** | Normal code (default after reset) | Restricted application code, such as RTOS tasks |
| **Handler mode** | **Always privileged** | Not possible |

- **Thread mode:** runs `main()` and RTOS tasks.
- **Handler mode:** runs exceptions and ISRs. Always privileged and always uses MSP.

### Two stack pointers

| Stack pointer | Used by |
|---|---|
| **MSP** (Main Stack Pointer) | Reset code, the OS kernel, and **all handlers** |
| **PSP** (Process Stack Pointer) | Optional, for RTOS tasks in Thread mode (selected by bit 1 of the `CONTROL` register) |

With an RTOS, every task has its own stack (PSP), while all ISRs share the MSP.

### What changes when an interrupt is accepted

| Item | Before (Thread) | During ISR (Handler) |
|---|---|---|
| Mode | Thread | **Handler** |
| Privilege | Privileged or Unprivileged | **Privileged** |
| Stack pointer used | MSP or PSP | **MSP** |
| `IPSR` (active exception number) | 0 | **Exception number** (e.g. 37) |
| `LR` | Normal return address | **EXC_RETURN** special value |
| `PC` | Background code address | **ISR address** |

On return, the CPU goes back to the mode, privilege and stack that were in use before, using the EXC_RETURN value (explained in section 6).

### Interrupt masking registers

| Register | Effect |
|---|---|
| **PRIMASK** | Blocks **all** configurable interrupts (`__disable_irq()`) |
| **BASEPRI** | Blocks interrupts at or below a chosen priority level |
| **FAULTMASK** | Also blocks HardFault; only NMI still works |

> Cortex-M0/M0+ has only PRIMASK. Cortex-M3 and higher have all three.

---

## 5. How the NVIC works

> **Definition:** The **NVIC (Nested Vectored Interrupt Controller)** is a hardware block inside every Cortex-M core. It receives interrupt requests, tracks their state, compares their priorities, and tells the CPU which ISR to run next.

The name describes its job:
- **Vectored:** it fetches the handler address directly from the vector table.
- **Nested:** a higher-priority interrupt can interrupt a lower-priority ISR.
- **Interrupt Controller:** it enables, prioritises and tracks interrupts.

### State of each interrupt

```
 Inactive ──(event)──► Pending ──(CPU accepts)──► Active ──(ISR returns)──► Inactive
```

### NVIC registers

| Register | Purpose |
|---|---|
| **ISER** | Enable an interrupt |
| **ICER** | Disable an interrupt |
| **ISPR** | Set pending (can trigger an interrupt from software) |
| **ICPR** | Clear pending |
| **IABR** | Read-only: which interrupts are active |
| **IPR** | Priority of each interrupt |

### Conditions for the CPU to take an interrupt

All of these must be true:

1. The peripheral's own interrupt-enable bit is set, and it raised the request.
2. The interrupt is **enabled** in the NVIC.
3. The interrupt is **pending**.
4. Its priority is **higher** than the currently running priority.
5. It is not blocked by `PRIMASK`, `BASEPRI` or `FAULTMASK`.

### Priority

> **Lower number means higher priority.** Priority `0` is the most urgent among configurable interrupts. Reset (-3), NMI (-2) and HardFault (-1) have fixed priorities above all of them.

**Priority grouping:** the priority byte is split by `PRIGROUP` (in `AIRCR`) into two fields:

```
 | Preempt priority bits | Sub-priority bits |
```

- **Preempt priority:** decides whether an interrupt may **interrupt another ISR**.
- **Sub-priority:** only decides the **order** when interrupts are pending at the same time. It never causes preemption.

Chips usually implement only the upper bits of the byte (STM32 implements 4 bits, giving 16 levels).

### Basic setup (CMSIS)

```c
NVIC_SetPriority(USART1_IRQn, 2);   // set priority
NVIC_EnableIRQ(USART1_IRQn);        // enable in NVIC
// also enable the interrupt inside the UART peripheral itself
```

---

## 6. What happens when an interrupt is triggered

```
 Peripheral event
       │
 [1]   ▼  Peripheral sets its flag and raises the request line
       │
 [2]   ▼  NVIC marks it PENDING and checks enable, priority and masks
       │
 [3]   ▼  CPU completes (or abandons) the current instruction
       │
 [4]   ▼  STACKING: hardware pushes 8 registers onto the stack
       │
 [5]   ▼  VECTOR FETCH: handler address read from the vector table
       │
 [6]   ▼  Handler mode entered, LR = EXC_RETURN, IPSR = exception number
       │
 [7]   ▼  ISR runs
       │
 [8]   ▼  ISR ends with BX LR (LR holds EXC_RETURN)
       │
 [9]   ▼  UNSTACKING: hardware pops 8 registers, PC restored
       │
       ▼  Background resumes exactly where it stopped
```

### Step 4: automatic stacking

> **Definition:** Stacking is the hardware saving of the CPU context onto the stack, performed automatically with no software instructions. This is why Cortex-M interrupts are fast.

The saved block is the **exception frame**:

```
 Higher address
   ┌─────────┐
   │  xPSR   │  status register
   │   PC    │  return address (where to resume)
   │   LR    │
   │   R12   │
   │   R3    │
   │   R2    │
   │   R1    │
   │   R0    │  ← SP points here after stacking
   └─────────┘
 Lower address
```

These are the **caller-saved registers** of the ARM calling convention (AAPCS). Because of this, an ISR can be an ordinary C function: the compiler saves R4 to R11 itself if it needs them.

If an FPU is present and was in use, the frame also holds S0 to S15 and FPSCR (26 words instead of 8). With **lazy stacking**, space is reserved but the FPU registers are copied only if the ISR actually uses the FPU.

### Step 6: EXC_RETURN

> **Definition:** EXC_RETURN is a special value loaded into LR on exception entry. It is not a real address. It tells the CPU which mode and stack to return to.

| EXC_RETURN | Return to | Stack |
|---|---|---|
| `0xFFFFFFF1` | Handler mode (returning to a nested ISR) | MSP |
| `0xFFFFFFF9` | Thread mode | MSP |
| `0xFFFFFFFD` | Thread mode | PSP |
| `0xFFFFFFE1 / E9 / ED` | Same three, with an FPU frame | |

When the ISR executes `BX LR` and the CPU sees a value of the form `0xFFFFFFxx` going into the PC, it knows this is an exception return and starts unstacking.

---

## 7. How the program counter moves between background and foreground

> **Idea:** The PC is not remembered by magic. It is **pushed onto the stack on entry** and **popped from the stack on exit**.

Example:

```
Address        Code
0x0800_0100    LDR  R0, [R1]       ← background running
0x0800_0104    ADD  R0, R0, #1     ← interrupt arrives during this instruction
0x0800_0108    STR  R0, [R1]       ← next instruction (resume point)

0x0800_0500    USART1_IRQHandler:  ← ISR starts
 ...
0x0800_0520    BX   LR             ← ISR ends
```

| Step | What happens | PC |
|---|---|---|
| 1 | Background executes `ADD` | `0x0800_0104` |
| 2 | Interrupt accepted when `ADD` completes | `0x0800_0108` (next instruction) |
| 3 | Hardware **pushes this PC** on the stack as the return address | `0x0800_0108` |
| 4 | Hardware reads the USART1 vector table entry | PC becomes `0x0800_0500` |
| 5 | ISR executes (foreground) | `0x0800_0500` to `0x0800_0520` |
| 6 | `BX LR` with LR = `0xFFFFFFF9` is seen as exception return | |
| 7 | Hardware **pops the saved PC** from the stack | PC becomes `0x0800_0108` |
| 8 | Background continues | `0x0800_0108` |

The background code never knows it was paused.

**Long instructions:** multi-cycle instructions such as `LDM`, `STM`, `PUSH`, `POP` and divide can be abandoned and restarted if an interrupt arrives mid-way. This keeps latency low.

---

## 8. Multiple interrupts at the same time

The NVIC resolves this with fixed rules.

### Case 1: pending together, different priority
The **higher priority** (lower number) is served first. The other stays pending.

### Case 2: same preempt priority
1. The lower **sub-priority** number goes first.
2. If still tied, the **lower IRQ number** goes first.

### Case 3: new interrupt arrives while an ISR is running

| New interrupt | Result |
|---|---|
| **Higher** priority than the running ISR | **Preempts** it (nesting) |
| **Equal or lower** priority | Waits as pending until the ISR finishes |

```
Priority 1  ─────────────────[ Timer ISR ]──────────────────
                              ▲          │
Priority 3  ──[ UART ISR ]────┘ preempt  └─ resume ─[ UART ISR cont. ]──
Background  ─[ main ]─────────────────────────────────────────────[ main ]
```

### Case 4: the same interrupt fires again during its own ISR
The pending bit is a **single bit**, not a counter. If the event happens 5 times while the ISR runs, you get only **one** more call. Your ISR must therefore process everything available (for example drain the UART FIFO) and must not assume one interrupt equals one event.

### NVIC optimisations

| Feature | What it does | Benefit |
|---|---|---|
| **Tail-chaining** | If another interrupt is pending when an ISR ends, the CPU skips unstack and stack and jumps straight to the next ISR | About 6 cycles instead of about 24 |
| **Late arrival** | A higher-priority interrupt arriving during stacking of a lower one takes over, reusing the stacking already done | No wasted work |
| **Pop preemption** | A new interrupt arriving during unstacking cancels the unstack and goes straight to the new ISR | Faster response |

---

## 9. Interrupt latency

> **Definition:** Interrupt latency is the time from the **interrupt request** until the **first instruction of the ISR** executes.

```
 Event ──► NVIC pending ──► stacking ──► vector fetch ──► ISR first instruction
           └──────────────────── LATENCY ──────────────────┘
```

### Hardware latency

| Core | Minimum latency (zero-wait-state memory) |
|---|---|
| Cortex-M3 / M4 / M7 | **12 cycles** (about 167 ns at 72 MHz) |
| Cortex-M0 / M0+ | 15 to 16 cycles |
| Tail-chained | About 6 cycles |

### What increases latency

| Cause | Effect |
|---|---|
| A higher or equal priority ISR already running | You wait until it finishes |
| Critical sections (`__disable_irq()`) | Request is held until interrupts are re-enabled |
| Flash wait states or slow bus | Stacking and vector fetch take longer |
| Long non-abandonable operations | Small extra delay |
| FPU context saving (non-lazy) | About 17 extra cycles |

### Worst-case latency

```
Worst latency = hardware latency
              + longest critical section
              + longest higher/equal priority ISR that may run first
```

### Related terms

| Term | Meaning |
|---|---|
| **Latency** | Request to first ISR instruction |
| **Response time** | Request to the work being fully done |
| **Jitter** | Variation of latency from one event to the next. Critical in motor control and audio |

### How to reduce it
- Give urgent interrupts the highest priority.
- Keep every ISR short.
- Keep critical sections as short as possible.
- Place the vector table and ISR code in RAM or TCM (zero wait states).
- Use DMA for bulk data instead of one interrupt per byte.

---

## 10. Rules for writing an interrupt handler

1. **Keep it short and fast.** Do the minimum, defer the rest to the background.
2. **Clear the interrupt flag.** Forgetting it makes the ISR re-trigger forever (a very common bug).
3. **Never block or wait.** No delays, no busy-wait loops, no waiting on other events.
4. **Use `volatile`** for variables shared with background code.
5. **No `printf`, `malloc` or file I/O.** They are slow and not safe to call from an ISR.
6. **Avoid non-re-entrant functions.** If the background was inside one, you corrupt its state.
7. **Signature is `void IRQHandler(void)`.** No arguments and no return value.
8. **Protect shared data** with a critical section when it cannot be read or written in one atomic access.
9. **Watch stack usage.** ISRs run on MSP, and nested ISRs add up.
10. **Use ISR-safe RTOS calls only**, for example `xSemaphoreGiveFromISR()`.
11. **Use the exact handler name** from the startup file, or the default handler will run instead.
12. **Choose priorities deliberately.** If everything is top priority, nothing is.

### The classic shared-data bug

```c
volatile uint32_t counter;       // 32-bit read or write is atomic on Cortex-M
volatile uint64_t big_counter;   // 64-bit access is NOT atomic, it can be torn

__disable_irq();
uint64_t copy = big_counter;     // safe copy
__enable_irq();
```

Also, `counter++` is a **read-modify-write** (three steps). If an ISR modifies the same variable, an update can be lost.

### Bad vs. good

```c
// BAD
void TIM2_IRQHandler(void)
{
    printf("tick\n");           // slow, unsafe
    HAL_Delay(100);             // blocks everything
    process_big_buffer();       // too long
    // flag never cleared!
}

// GOOD
void TIM2_IRQHandler(void)
{
    TIM2->SR &= ~TIM_SR_UIF;    // clear flag first
    tick_flag = true;           // background does the work
}
```

### Useful pattern: ring buffer
The ISR writes into a circular buffer and the background reads from it. Each side changes only its own index, so a single producer and single consumer need no locking.

---

## 11. Extra topics worth knowing

### Special system exceptions

| Exception | Purpose |
|---|---|
| **Reset** | Startup |
| **NMI** | Non-maskable, for critical failures such as clock failure |
| **HardFault** | Catch-all for serious errors |
| **MemManage / BusFault / UsageFault** | Specific fault types (M3 and above) |
| **SVC** | Software call into the OS (supervisor call) |
| **PendSV** | Lowest priority, used by RTOS for **context switching** |
| **SysTick** | System timer, the RTOS heartbeat |

**Why PendSV is lowest priority:** the RTOS requests a task switch through PendSV. Since it runs only after all other ISRs finish, a context switch never happens in the middle of an ISR.

### Power saving
`WFI` (Wait For Interrupt) sleeps the CPU until an interrupt arrives:

```c
while (1) {
    if (!work_pending) __WFI();   // sleep until next interrupt
    else do_work();
}
```

### Debugging checklist

| Symptom | Likely cause |
|---|---|
| Stuck in the default handler | Wrong handler name, or an interrupt enabled with no handler |
| ISR fires endlessly | Flag not cleared |
| ISR never fires | Peripheral enable, NVIC enable or global mask missing |
| Random crashes | Stack overflow, or a non-`volatile` shared variable optimised away |
| HardFault | Inspect the stacked PC to find the faulting instruction |

### Setup checklist
1. Configure the peripheral and enable **its** interrupt.
2. Set the priority with `NVIC_SetPriority()`.
3. Enable it in the NVIC with `NVIC_EnableIRQ()`.
4. Make sure interrupts are not globally masked.
5. Write `IRQHandler` with the exact vector-table name.
6. **Clear the flag** inside the ISR.

---

## 12. Summary

| Question | Short answer |
|---|---|
| What is an interrupt? | A hardware signal that pauses normal code to run an ISR, then resumes |
| Foreground vs. background | ISRs (urgent, short) vs. the `main` loop (long, non-urgent) |
| Vector table | Array of handler addresses indexed by exception number |
| CPU mode change | Thread to Handler, privileged, MSP, IPSR set to exception number |
| What does the NVIC do? | Enables, prioritises, nests and dispatches interrupts |
| What happens on trigger? | Pending, stack 8 registers, vector fetch, Handler mode, ISR, unstack |
| PC change | Pushed on entry, popped on exit through EXC_RETURN |
| Multiple at once | Highest priority first; ties go to sub-priority, then lowest IRQ number |
| Latency | 12 cycles minimum on M3/M4, plus masking and higher-priority ISRs |
| ISR rules | Short, clear flag, no blocking, `volatile`, ISR-safe calls only |

---

*Next week, topic ideas: DMA, RTOS context switching with PendSV, Cortex-M startup sequence, or HardFault debugging.*
