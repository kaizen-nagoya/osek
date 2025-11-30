# unified-autosar-os — Skeleton repository & PoC

This document provides a ready-to-use repository skeleton, Docker-based dev environment, and first proof-of-concept code for:

* ISR classification (With OS / Without OS) bridge
* POSIX PSE51 compatibility stubs mapped to OSEK-like primitives

Use this as the initial commit for `unified-autosar-os` on GitHub.

## Repo layout (suggested)

```
unified-autosar-os/
├─ README.md
├─ ROADMAP.md
├─ ARCHITECTURE.md
├─ LICENSE
├─ .github/
│  ├─ workflows/ci.yml
│  └─ ISSUE_TEMPLATE/
├─ docker-dev-env/
│  ├─ Dockerfile
│  ├─ build.sh
│  └─ run-qemu.sh
├─ osek-core/  (git submodule or fork)
├─ bsd-core-rt/ (git submodule or fork)
├─ src/
│  ├─ interrupt_bridge.c
│  ├─ interrupt_bridge.h
│  ├─ posix_compat.c
│  └─ posix_compat.h
├─ examples/
│  ├─ test_isr.c
│  └─ test_posix.c
└─ ci/
   └─ test-scripts/
       └─ run_unit_tests.sh
```


## README.md (starter)

````markdown
# unified-autosar-os

Skeleton repo to unify BSD-style POSIX environment with OSEK/VDX Classic RTOS features.

## Quick start (development Docker)

```bash
# build container
cd docker-dev-env
./build.sh

# start interactive dev shell
./run-qemu.sh  # or `docker run -it unified-autosar-os:dev` depending on script
````

## Goals

* Provide ISR classification "With OS" / "Without OS"
* Implement Priority Ceiling Protocol glue
* Provide POSIX PSE51 compatibility layer on top of OSEK primitives

## Contributing

Follow CONTRIBUTING.md and use Issues/PRs.

```
```

## docker-dev-env/Dockerfile

```dockerfile
# Simple development Dockerfile for building OSEK and running tests under QEMU
FROM ubuntu:24.04

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential git cmake pkg-config autoconf automake libtool \
    qemu-system-x86 qemu-system-arm qemu-user-static \
    gcc-arm-none-eabi gdb-multiarch \
    python3 python3-pip sudo curl wget vim \
  && apt-get clean && rm -rf /var/lib/apt/lists/*

# Create developer user
RUN useradd -ms /bin/bash dev && echo "dev ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/dev
USER dev
WORKDIR /home/dev

# Optional: clone osek and bsd forks (left to user)
# RUN git clone https://github.com/kaizen-nagoya/osek.git osek-core

ENV PATH="/home/dev/.local/bin:${PATH}"

CMD ["/bin/bash"]
```

## docker-dev-env/build.sh

```bash
#!/usr/bin/env bash
set -e
IMAGE_NAME=unified-autosar-os:dev
docker build -t $IMAGE_NAME .
echo "Built $IMAGE_NAME"
```

---

## docker-dev-env/run-qemu.sh

```bash
#!/usr/bin/env bash
# Run an interactive dev container (for QEMU, build tools)
IMAGE_NAME=unified-autosar-os:dev
docker run --rm -it --privileged -v "$PWD/..:/workspace" -w /workspace $IMAGE_NAME
```

## .github/workflows/ci.yml (basic)

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up QEMU
        run: sudo apt-get update && sudo apt-get install -y qemu-system-x86 build-essential
      - name: Build test binaries
        run: |
          cd src || exit 1
          gcc -std=c11 -Wall -Wextra -O2 -c interrupt_bridge.c posix_compat.c || true
      - name: Run unit tests (placeholder)
        run: echo "Add real tests"
```

## src/interrupt_bridge.h

```c
#ifndef INTERRUPT_BRIDGE_H
#define INTERRUPT_BRIDGE_H

#include <stdint.h>

typedef enum {
    ISR_WITH_OS = 0,
    ISR_WITHOUT_OS = 1
} isr_kind_t;

typedef void (*isr_handler_t)(void *arg);

// Register an ISR with classification. Returns 0 on success.
int register_isr(int irq, isr_kind_t kind, isr_handler_t handler, void *arg);

// Unregister
int unregister_isr(int irq);

// Simple priority ceiling resource API (PoC)
typedef int resource_t; // opaque in PoC

int resource_init(resource_t *r, int ceiling_priority);
int resource_get(resource_t *r);
int resource_release(resource_t *r);

#endif // INTERRUPT_BRIDGE_H
```

## src/interrupt_bridge.c

```c
#include "interrupt_bridge.h"
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

// Very small PoC mapping: we maintain a registration table in userland for tests only.
#define MAX_IRQ 256

struct isr_entry {
    int used;
    isr_kind_t kind;
    isr_handler_t handler;
    void *arg;
    int priority; // lower number == higher priority
};

static struct isr_entry table[MAX_IRQ];

int register_isr(int irq, isr_kind_t kind, isr_handler_t handler, void *arg) {
    if (irq < 0 || irq >= MAX_IRQ) return -1;
    table[irq].used = 1;
    table[irq].kind = kind;
    table[irq].handler = handler;
    table[irq].arg = arg;
    table[irq].priority = 255; // default
    printf("[isr] registered irq=%d kind=%d\n", irq, kind);
    return 0;
}

int unregister_isr(int irq) {
    if (irq < 0 || irq >= MAX_IRQ) return -1;
    table[irq].used = 0;
    return 0;
}

// Priority ceiling PoC
int resource_init(resource_t *r, int ceiling_priority) {
    if (!r) return -1;
    *r = ceiling_priority; // PoC: store as int
    return 0;
}

// In a full kernel integration this would raise the caller's priority to the ceiling
int resource_get(resource_t *r) {
    if (!r) return -1;
    // PoC: no-op
    return 0;
}

int resource_release(resource_t *r) {
    if (!r) return -1;
    // PoC: no-op
    return 0;
}

// Simulate an IRQ trigger from userspace test
void trigger_irq(int irq) {
    if (irq < 0 || irq >= MAX_IRQ) return;
    if (!table[irq].used) {
        printf("[isr] no handler for irq=%d\n", irq);
        return;
    }
    if (table[irq].kind == ISR_WITHOUT_OS) {
        // Call handler directly (simulates non-OS ISR)
        table[irq].handler(table[irq].arg);
    } else {
        // With OS: queue to worker thread (PoC uses pthread)
        // In kernel this would schedule a task; here we spawn a thread
        pthread_t t;
        struct isr_entry *e = &table[irq];
        pthread_create(&t, NULL, (void *(*)(void *))e->handler, e->arg);
        pthread_detach(t);
    }
}
```

---

## src/posix_compat.h

```c
#ifndef POSIX_COMPAT_H
#define POSIX_COMPAT_H

#include <pthread.h>

// Initialize compatibility layer (map configs)
int posix_compat_init(void);

// Create a POSIX-like thread that maps to OSEK task in full implementation.
int posix_thread_create(pthread_t *thread, const pthread_attr_t *attr,
                        void *(*start_routine)(void *), void *arg);

// Simple wrappers for mutex
int posix_mutex_init(pthread_mutex_t *m);
int posix_mutex_lock(pthread_mutex_t *m);
int posix_mutex_unlock(pthread_mutex_t *m);

#endif // POSIX_COMPAT_H
```

## src/posix_compat.c

```c
#include "posix_compat.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int posix_compat_init(void) {
    // PoC: read configuration or set up translation tables.
    printf("[posix_compat] init\n");
    return 0;
}

int posix_thread_create(pthread_t *thread, const pthread_attr_t *attr,
                        void *(*start_routine)(void *), void *arg) {
    // PoC: create a pthread. Real implementation would map to OSEK TaskType
    if (!thread || !start_routine) return -1;
    return pthread_create(thread, attr, start_routine, arg);
}

int posix_mutex_init(pthread_mutex_t *m) {
    if (!m) return -1;
    pthread_mutexattr_t a;
    pthread_mutexattr_init(&a);
    pthread_mutexattr_setprotocol(&a, PTHREAD_PRIO_INHERIT);
    int r = pthread_mutex_init(m, &a);
    pthread_mutexattr_destroy(&a);
    return r;
}

int posix_mutex_lock(pthread_mutex_t *m) {
    return pthread_mutex_lock(m);
}
int posix_mutex_unlock(pthread_mutex_t *m) {
    return pthread_mutex_unlock(m);
}
```

---

## examples/test_isr.c

```c
#include "../src/interrupt_bridge.h"
#include <stdio.h>

void fast_isr(void *arg) {
    printf("fast_isr: arg=%p\n", arg);
}

void threaded_isr(void *arg) {
    printf("threaded_isr start arg=%p\n", arg);
    // emulate work
    for (int i=0;i<3;i++) {
        printf("threaded_isr: step %d\n", i);
    }
}

int main(void) {
    register_isr(10, ISR_WITHOUT_OS, fast_isr, (void*)0xdeadbeef);
    register_isr(11, ISR_WITH_OS, threaded_isr, (void*)0xcafebabe);

    trigger_irq(10);
    trigger_irq(11);

    // wait a bit for threads
    sleep(1);
    return 0;
}
```

## examples/test_posix.c

```c
#include "../src/posix_compat.h"
#include <stdio.h>

void *worker(void *arg) {
    printf("worker running: %s\n", (char*)arg);
    return NULL;
}

int main(void) {
    posix_compat_init();
    pthread_t t;
    posix_thread_create(&t, NULL, worker, "hello");
    pthread_join(t, NULL);
    return 0;
}
```

## ci/test-scripts/run_unit_tests.sh

```bash
#!/usr/bin/env bash
set -e
cd $(dirname "$0")/../../src
gcc -std=c11 -Wall -O2 interrupt_bridge.c posix_compat.c -lpthread -o unit_tests
./unit_tests
```

## CONTRIBUTING / ISSUE templates (suggested)

Create `.github/ISSUE_TEMPLATE/bug_report.md` and `feature_request.md` with boilerplate.

## Next steps (suggested immediate PR contents)

1. Add this skeleton to a new GitHub repo.
2. Add `osek-core` and `bsd-core-rt` as submodules or document how to fork them.
3. Push Dockerfile + scripts and verify container builds.
4. Run `examples/test_isr.c` and `examples/test_posix.c` inside the container.
5. Start a design document in `ARCHITECTURE.md` describing the kernel-space vs user-space boundary.

Happy to:

* generate an initial `git` commit patch (diff) you can apply locally, or
* create a GitHub repo with these files if you provide the organization/name and permission (I cannot perform GitHub actions for you, but I can produce the files and instructions).

Which would you like next?

## Reference

