## RP2040 ResNet-8 Hardware Fault Injection Campaign – Development Log

### Objective

The goal of this work was to implement a hardware fault injection framework on an RP2040-based deployment of a quantized ResNet-8 neural network and classify injected faults into:

1) Masked Faults - The injected error doesn't change the inference output
2) Safe Silent Data Corruption (Safe SDC) - When the fault does not change the corruption
3) Critical Silent Data Corruption (Critical SDC) - When the fault changes the corruption
4) Detectable Unrecoverable Errors (DUE) - Fault causing a system hang or crash

Read it from a paper called [Evaluating Different Fault Injection Abstractions on
the Assessment of DNN SW Hardening Strategies](https://ieeexplore.ieee.org/document/10915284)


### Initial Setup

The ResNet-8 model was successfully deployed on the Raspberry Pi Pico (RP2040). The inference pipeline was:

```
  predicted = resnet_infer_full(test_image);
  print_results(g_logits, predicted);
```
where:

1) g_logits contains the final classification logits.
2) predicted contains the argmax output class.


To support fault campaign analysis, two global variables were introduced:
```
volatile int FI_PREDICTION;
volatile long FI_SUM;
```

These variables store:
```
FI_PREDICTION = predicted;
FI_SUM = 0;

for(int i = 0; i < NUM_CLASSES; i++)
{
    FI_SUM += g_logits[i];
}
```

The purpose was:

1) FI_PREDICTION → detect Critical SDC
2) FI_SUM → detect Safe SDC


### Initial Fault Injection Strategy

The first strategy used dedicated markers:

```
START_FI_INJECT();
...
...
END_FI_INJECT();
```
The GDB campaign would:

1) Break at START_FI_INJECT
2) Modify a register
3) Continue execution
4) Read the final output
5) Check the final output with golden output and determine the type of injected fault

Example:
```
GDB Commands

break START_FI_INJECT
continue

set $r0 = $r0 ^ (1<<5)

continue
```

Even after 100s of injections, no faults were visible, the injection implemented was not affective.

### Investigation of Register Usage

Registers R0-R12 were injected.

Observations:
1) R1-R6
2) R8-R10

showed some propagation while R7 caused frequent stalls or timeouts. Investigation revealed:
R7 = Frame Pointer (FP) Corrupting R7 damages stack addressing.

Consequences:
1) Invalid stack accesses
2) Infinite loops
3) Program hangs
4) This explained why R7 behaved differently from other registers.

### Migration to ResNet Logit Injection

This Idea was occured as Logits is the part where the prediction of the digits are decided. So the most affected part will be logits.

Additional markers were introduced:
```
LOGITS_START();
fc(...);
LOGITS_END();
```
```
The GDB sequence:

break LOGITS_END
continue
finish
p g_logits[0]@10
set g_logits[1] = 150
```

Result : 
\
Prediction = 1 
\
\
instead of :
\
Golden Prediction = 4
\
\
This produced the first confirmed Critical SDC because the classification output changed.


### Logit Corruption is Memory Corruption

Although effective, this method modifies g_logits[] which resides in SRAM. so its Memory Fault Injection rather than Register Fault Injection.

















