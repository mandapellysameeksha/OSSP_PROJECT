#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mman.h>
#include <sys/select.h>
#include <time.h>
#include <string.h>
#include <errno.h>

typedef struct {
    volatile unsigned long counter;
    volatile sig_atomic_t running;
} SharedData;

static SharedData *shared;
static pid_t worker_pid = -1;
static char message[160] = "Ready. Press 1 to start.";
static int paused = 0;

static void worker_term(int sig) {
    (void)sig;
    shared->running = 0;
}

static void worker_loop(void) {
    struct sigaction sa = {0};
    sa.sa_handler = worker_term;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGTERM, &sa, NULL);

    shared->running = 1;

    while (shared->running) {
        shared->counter++;

        struct timespec delay = {0, 200000000};
        nanosleep(&delay, NULL);
    }

    _exit(0);
}

static void check_worker(void) {
    if (worker_pid <= 0) return;

    int status;
    pid_t result = waitpid(worker_pid, &status, WNOHANG);

    if (result == worker_pid ||
        (result == -1 && errno == ECHILD)) {
        worker_pid = -1;
        paused = 0;
        snprintf(message, sizeof(message),
                 "Worker has exited.");
    }
}

static void start_process(void) {
    check_worker();

    if (worker_pid > 0) {
        snprintf(message, sizeof(message),
                 "Worker already exists.");
        return;
    }

    shared->counter = 0;
    shared->running = 1;

    pid_t pid = fork();

    if (pid < 0) {
        snprintf(message, sizeof(message),
                 "Fork failed: %s", strerror(errno));
        return;
    }

    if (pid == 0) {
        worker_loop();
    }

    worker_pid = pid;
    paused = 0;

    snprintf(message, sizeof(message),
             "Started worker PID %ld", (long)pid);
}

static void pause_process(void) {
    check_worker();

    if (worker_pid <= 0 || paused) {
        snprintf(message, sizeof(message),
                 "No running worker to pause.");
        return;
    }

    if (kill(worker_pid, SIGSTOP) == 0) {
        paused = 1;
        snprintf(message, sizeof(message),
                 "SIGSTOP sent. Worker suspended.");
    } else {
        snprintf(message, sizeof(message),
                 "Pause failed: %s", strerror(errno));
    }
}

static void resume_process(void) {
    check_worker();

    if (worker_pid <= 0 || !paused) {
        snprintf(message, sizeof(message),
                 "No paused worker to resume.");
        return;
    }

    if (kill(worker_pid, SIGCONT) == 0) {
        paused = 0;
        snprintf(message, sizeof(message),
                 "SIGCONT sent. Worker resumed.");
    } else {
        snprintf(message, sizeof(message),
                 "Resume failed: %s", strerror(errno));
    }
}

static void terminate_process(void) {
    check_worker();

    if (worker_pid <= 0) {
        snprintf(message, sizeof(message),
                 "No active worker.");
        return;
    }

    pid_t pid = worker_pid;

    if (paused) {
        kill(pid, SIGCONT);
    }

    if (kill(pid, SIGTERM) == -1 && errno != ESRCH) {
        snprintf(message, sizeof(message),
                 "Terminate failed: %s", strerror(errno));
        return;
    }

    int status;
    pid_t result;

    do {
        result = waitpid(pid, &status, 0);
    } while (result == -1 && errno == EINTR);

    worker_pid = -1;
    paused = 0;

    snprintf(message, sizeof(message),
             "Worker PID %ld terminated.", (long)pid);
}

/* Read actual Linux process information */
static void get_process_info(char *state,
                             size_t state_size,
                             double *memory_mb) {
    snprintf(state, state_size, "Not running");
    *memory_mb = 0;

    if (worker_pid <= 0) return;

    char path[80];
    snprintf(path, sizeof(path),
             "/proc/%ld/status", (long)worker_pid);

    FILE *fp = fopen(path, "r");
    if (!fp) {
        snprintf(state, state_size, "Unavailable");
        return;
    }

    char line[256];

    while (fgets(line, sizeof(line), fp)) {
        if (strncmp(line, "State:", 6) == 0) {
            char *p = line + 6;
            while (*p == ' ' || *p == '\t') p++;

            size_t len = strcspn(p, "\n");
            if (len >= state_size)
                len = state_size - 1;

            memcpy(state, p, len);
            state[len] = '\0';
        }

        if (strncmp(line, "VmRSS:", 6) == 0) {
            unsigned long kb = 0;
            if (sscanf(line + 6, "%lu", &kb) == 1)
                *memory_mb = kb / 1024.0;
        }
    }

    fclose(fp);
}

/* CPU ticks from /proc/PID/stat */
static unsigned long long get_cpu_ticks(void) {
    if (worker_pid <= 0) return 0;

    char path[80];
    snprintf(path, sizeof(path),
             "/proc/%ld/stat", (long)worker_pid);

    FILE *fp = fopen(path, "r");
    if (!fp) return 0;

    char line[4096];
    if (!fgets(line, sizeof(line), fp)) {
        fclose(fp);
        return 0;
    }
    fclose(fp);

    char *p = strrchr(line, ')');
    if (!p) return 0;
    p++;

    unsigned long long utime = 0, stime = 0;
    char *saveptr = NULL;
    char *token = strtok_r(p, " ", &saveptr);

    int field = 3;
    while (token) {
        if (field == 14)
            utime = strtoull(token, NULL, 10);
        if (field == 15) {
            stime = strtoull(token, NULL, 10);
            break;
        }
        field++;
        token = strtok_r(NULL, " ", &saveptr);
    }

    return utime + stime;
}

static void draw_dashboard(double cpu_percent) {
    check_worker();

    char state[100];
    double memory_mb;

    get_process_info(state, sizeof(state), &memory_mb);

    /* Clear screen and move cursor to top */
    printf("\033[2J\033[H");

    printf("\033[1;36m");
    printf("====================================================\n");
    printf("       LINUX SIGNAL PROCESS CONTROL DASHBOARD\n");
    printf("====================================================\n");
    printf("\033[0m");

    printf("\n  PROCESS INFORMATION\n");
    printf("  ------------------------------------------------\n");

    if (worker_pid > 0)
        printf("  PID             : %ld\n", (long)worker_pid);
    else
        printf("  PID             : None\n");

    printf("  Linux State     : %s\n", state);
    printf("  CPU Usage       : %.2f%% (one CPU core)\n",
           cpu_percent);
    printf("  Memory Usage    : %.2f MB\n", memory_mb);

    printf("\n  LIVE WORKER ACTIVITY\n");
    printf("  ------------------------------------------------\n");
    printf("  Iteration Count : %lu\n", shared->counter);

    if (worker_pid <= 0)
        printf("  Activity        : STOPPED\n");
    else if (paused)
        printf("  Activity        : PAUSED\n");
    else
        printf("  Activity        : RUNNING\n");

    printf("\n  PROCESS CONTROLS\n");
    printf("  ------------------------------------------------\n");
    printf("  [1] Start       [2] Pause       [3] Resume\n");
    printf("  [4] Terminate   [5] Restart     [6] Status\n");
    printf("  [0] Exit\n");

    printf("\n  LAST EVENT\n");
    printf("  ------------------------------------------------\n");
    printf("  %s\n", message);

    printf("\n  Enter choice (0-6): ");
    fflush(stdout);
}

int main(void) {
    shared = mmap(NULL, sizeof(SharedData),
                  PROT_READ | PROT_WRITE,
                  MAP_SHARED | MAP_ANONYMOUS,
                  -1, 0);

    if (shared == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    shared->counter = 0;
    shared->running = 0;

    long ticks_per_sec = sysconf(_SC_CLK_TCK);
    if (ticks_per_sec <= 0) ticks_per_sec = 100;

    unsigned long long last_ticks = 0;
    pid_t last_pid = -1;

    while (1) {
        check_worker();

        unsigned long long ticks = get_cpu_ticks();
        double cpu = 0;

        if (worker_pid > 0 && worker_pid == last_pid &&
            ticks >= last_ticks) {
            cpu = (double)(ticks - last_ticks) /
                  (double)ticks_per_sec * 100.0;
        }

        last_ticks = ticks;
        last_pid = worker_pid;

        draw_dashboard(cpu);

        fd_set readfds;
        FD_ZERO(&readfds);
        FD_SET(STDIN_FILENO, &readfds);

        struct timeval timeout;
        timeout.tv_sec = 1;
        timeout.tv_usec = 0;

        int ready = select(STDIN_FILENO + 1,
                           &readfds, NULL, NULL, &timeout);

        if (ready < 0) {
            if (errno == EINTR) continue;
            perror("select");
            break;
        }

        if (ready > 0) {
            char input[64];

            if (!fgets(input, sizeof(input), stdin))
                break;

            int choice = atoi(input);

            switch (choice) {
                case 1:
                    start_process();
                    break;
                case 2:
                    pause_process();
                    break;
                case 3:
                    resume_process();
                    break;
                case 4:
                    terminate_process();
                    break;
                case 5:
                    terminate_process();
                    start_process();
                    break;
                case 6:
                    snprintf(message, sizeof(message),
                             "Status refreshed.");
                    break;
                case 0:
                    terminate_process();
                    munmap(shared, sizeof(SharedData));
                    printf("\033[2J\033[H");
                    printf("Controller exited safely.\n");
                    return 0;
                default:
                    snprintf(message, sizeof(message),
                             "Invalid choice. Use 0-6.");
            }
        }
    }

    terminate_process();
    munmap(shared, sizeof(SharedData));
    return 0;
}
