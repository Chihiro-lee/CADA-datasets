# Appendix

> This Markdown page contains Attack Development used in CADA. Because zenodo can not compile markdown, for a better experience please download all files from repo and then see this markdown file locally.

## Attack Development

**Applications Injected with Different Attacks**

| Application | Description |
|---|---|
| `syringePump_OAT_atk` | Syringe-pump program from the OAT test program, ported to MSP430F5529 and the ACFA board. A control-flow attack is injected through a buffer overflow. |
| `syringePump_ACFA_atk` | Syringe-pump program from the ACFA test program, ported to MSP430F5529. A control-flow attack is injected through a buffer overflow. |
| `ACFA_demo_atk` | The attack demo from ACFA, adapted to MSP430F5529. |
| `TOCTOU_atk1` | TOCTOU attack targeting `8bit2dimMatrix`, ported to ACFA and CADA. |
| `TOCTOU_atk2` | TOCTOU attack targeting `16bit2dimMatrix`, ported to ACFA and CADA. |
| `TOCTOU_atk3` | TOCTOU attack targeting `bitcount`, ported to ACFA and CADA. |
| `TOCTOU_atk4` | TOCTOU attack targeting `32bitMath`, ported to ACFA and CADA. |
| `TOCTOU_atk5` | TOCTOU attack targeting `matrixMultiplication`, ported to ACFA and CADA. |
| `DF_atk_global` | Data-only attack targeting a global buffer. |
| `DF_atk_local` | Data-only attack targeting a local buffer. |

No publicly available control-flow-hijacking, direct-data-manipulation, or TOCTOU-attack proof of concept suitable for MSP430F5529 applications was found in CVEs or the official TI-board community. We therefore construct four buffer-overflow control-flow attack programs and adapt one attack demo from ACFA, corresponding to the first five applications in the preceding table.

The control-flow attack is simulated by constructing a buffer overflow in the prover program. The following example includes a vulnerability through which crafted `user_input` can cause `waitForPassword` to return to an attacker-selected code location.

```c
#define TARGET_UPPER    0xfe
#define TARGET_LOWER    0xc2

char user_input[6] = {0x01, 0x02, 0x03,
                      TARGET_LOWER, TARGET_UPPER, '\r'};

void waitForPassword() {
    char entry[4] = {0, 0, 0, 0};
    read_data(entry);
    unsigned int i;
    for (i = 0; i < 4; i++) {
        total |= (pass[i] ^ entry[i]);
    }
}

void read_data(char *entry) {
    // Simulate UART receive.
    int i = 0;
    while (user_input[i] != '\r') {
        // Save read value.
        *(entry + i) = user_input[i];
        i++;
    }
}
```

We develop five TOCTOU attack programs, each targeting a benign application listed above. The benign application is initially deployed and attested as secure. In the attack scenario, the benign application directly initiates a code-update request without verifier supervision. An adversary can then inject a TOCTOU attack program into the prover, have the attack program issue a second code-update request at completion, restore the benign application, and run it again.

CADA's verifier detects the TOCTOU attack through the continual increase in the memory-write count. The adversary is assumed to use CADA's toolchain to prepare the TOCTOU attack program, avoiding rejection by static code validation. The prover's network environment is not assumed to be secure, so the update process remains exploitable by the adversary. ACFA provides no remote-code-update support; on the ACFA board, the TOCTOU attack programs are only adapted and executed.

We also construct two data-only attack programs that modify a global and a local buffer, respectively. The following example simulates a data-only attack by overflowing `user_input` into the prover's local `settings` buffer. The overflow does not change the function's return instruction pointer, but the runtime memory-read values differ from the correct measurements obtained with `user_input` of length at most five.

```c
char total = 0;
char user_input[11] = {0x01, 0x02, 0x03, 0x04, 0x05,
                       0x06, 0x07, 0x08, 0x09, 0x10, '\r'};

void injectMedicinePort1(char *input) {
    volatile char settings[5] = {1, 1, 1, 1, 1};
    volatile char local[8] = {2, 2, 2, 2, 2, 2, 2, 2};
    int i = 0;

    while (input[i] != '\r') {
        settings[i] = input[i];
        total = 'a' ^ settings[i];
        i++;
    }

    for (int j = 0; j < 8; j++) {
        total = local[j];
    }
}
```
