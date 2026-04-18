#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <stdbool.h>
#include <unistd.h>
#include <time.h>

#define MAX_SENSORS 3
#define FAULT_LIMIT 5
#define RECOVERY_TIME_SEC 10

// THRESHOLDS
#define TEMP_MIN 20
#define MOISTURE_MIN 30        // Lowered to allow "Dry" readings to pass validation
#define MOISTURE_MAX 70
#define MOISTURE_DELTA 1       // Report if moisture changes by 1%

typedef struct {
    char name[20];
    int id;
    int last_reported_val;
    int fault_count;
    bool is_blocked;
    time_t blocked_until;
    pthread_mutex_t lock;
} SensorNode;

SensorNode sensors[MAX_SENSORS] = {
    {"TEMPERATURE", 101, -999, 0, false, 0, PTHREAD_MUTEX_INITIALIZER},
    {"SOIL_MOISTURE", 202, -999, 0, false, 0, PTHREAD_MUTEX_INITIALIZER},
    {"RESERVED", 303, -999, 0, false, 0, PTHREAD_MUTEX_INITIALIZER}
};

void print_time() {
    time_t now = time(NULL);
    struct tm *t = localtime(&now);
    printf("[%02d:%02d:%02d] ", t->tm_hour, t->tm_min, t->tm_sec);
}

// 1. VALIDATION LOGIC
bool is_data_valid(int idx, int val) {
    if (idx == 0) return (val >= TEMP_MIN);
    if (idx == 1) return (val >= MOISTURE_MIN && val <= MOISTURE_MAX);
    return (val >= 0 && val <= 100);
}

// 2. REPORTING LOGIC (The Fix)
bool should_report(int idx, int current_val) {
    SensorNode *s = &sensors[idx];
    if (s->last_reported_val == -999) return true; // Always report first reading

    if (idx == 0 || idx == 1) {
        // Report if the value has changed significantly (Delta)
        return (abs(current_val - s->last_reported_val) >= 1);
    }
    return true;
}

// 3. CORE PROCESSING ENGINE
void process_sensor_data(int idx, int val) {
    SensorNode *s = &sensors[idx];
    pthread_mutex_lock(&s->lock);

    time_t now = time(NULL);

    if (s->is_blocked) {
        if (now >= s->blocked_until) {
            print_time();
            printf("[RECOVERY] %s: Resuming service...\n", s->name);
            s->is_blocked = false;
            s->fault_count = 0;
        } else {
            pthread_mutex_unlock(&s->lock);
            return;
        }
    }

    if (!is_data_valid(idx, val)) {
        if (++s->fault_count >= FAULT_LIMIT) {
            s->is_blocked = true;
            s->blocked_until = now + RECOVERY_TIME_SEC;
            print_time();
            printf("[CRITICAL] %s SHUTDOWN: Invalid Range (%d). Blocked for %ds.\n",
                    s->name, val, RECOVERY_TIME_SEC);
        }
        pthread_mutex_unlock(&s->lock);
        return;
    }

    s->fault_count = 0;

    if (should_report(idx, val)) {
        s->last_reported_val = val;
        print_time();
        printf("[UPLINK] %s: Value %d\n", s->name, val);
    }

    pthread_mutex_unlock(&s->lock);
}

// 4. SIMULATION
void* sensor_thread(void* arg) {
    int idx = *((int*)arg);
    unsigned int seed = time(NULL) + idx;

    while (1) {
        int simulated_val;
        if (idx == 0) {
            simulated_val = 18 + (rand_r(&seed) % 10); // 18-28
        } else if (idx == 1) {
            simulated_val = 25 + (rand_r(&seed) % 40); // 25-65
        } else {
            simulated_val = 50;
        }

        process_sensor_data(idx, simulated_val);
        usleep(1000000);
    }
    return NULL;
}

int main() {
    pthread_t threads[MAX_SENSORS];
    int indices[MAX_SENSORS];

    printf("Gateway Active.\nBoundaries: Temp >= %d, Moisture %d-%d\n",
            TEMP_MIN, MOISTURE_MIN, MOISTURE_MAX);

    for (int i = 0; i < MAX_SENSORS; i++) {
        indices[i] = i;
        pthread_create(&threads[i], NULL, sensor_thread, &indices[i]);
    }

    for (int i = 0; i < MAX_SENSORS; i++) pthread_join(threads[i], NULL);
    return 0;
}
