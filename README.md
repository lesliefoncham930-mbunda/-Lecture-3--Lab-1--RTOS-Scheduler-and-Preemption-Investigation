# -Lecture-3--Lab-1--RTOS-Scheduler-and-Preemption-Investigation
<img width="1920" height="1080" alt="Screenshot (176)" src="https://github.com/user-attachments/assets/61c1cb7a-576d-4980-aafe-b2098f42c4c3" />


FreeRTOS Scheduler Demonstration
1. Assignment Purpose
The purpose of this assignment is to investigate how the FreeRTOS scheduler selects tasks on an ESP32-S3. Two tasks were created and pinned to the same CPU core. The experiment demonstrates Running, Ready, and Blocked states, task priorities, preemption, notifications, and equal-priority behaviour.

2. Hardware and Software
Hardware
ESP32-S3-DevKitC-1
USB cable
Software
Arduino IDE 2.x
ESP32 board package
Serial Monitor
Serial communication: 115200 baud
3. Task Configuration
Task A
Priority: 1
Pinned to CPU core 0
Prints a message character by character
Sends a notification to Task B
Task B
Pinned to CPU core 0
Waits for a notification using ulTaskNotifyTake()
Runs when notified using xTaskNotifyGive()
4. Priority Experiments
Scenario	Task A Priority	Task B Priority	Actual Observation
A	1	2	Task B ran before Task A finished its message.
B	1	1	Task B ran when Task A later blocked with vTaskDelay(). The output became interleaved.
C	1	3	Task B ran before Task A finished its message.
5. Prediction and Observation
Scenario	Prediction	Actual Observation	Match? Why?
A: B = 2	Task B should preempt Task A because B has higher priority.	Task B ran after notification and before Task A finished.	Yes. B had higher priority.
B: B = 1	Task B should not preempt Task A based on priority because both have equal priority.	Task B ran after Task A's vTaskDelay(), and the output became interleaved.	Yes. Equal priority did not give B priority-based preemption.
C: B = 3	Task B should preempt Task A because B has higher priority.	Task B ran after notification and before Task A finished.	Yes. B had higher priority.
6. Scheduler Behaviour
In this experiment, two FreeRTOS tasks were created on the same ESP32-S3 CPU core. Task A had priority 1, while Task B was tested with priorities 2, 1, and 3. At the beginning, Task B was in the Blocked state because it was waiting for a notification using ulTaskNotifyTake(). A blocked task does not use the CPU because it is waiting for an event.

Task A was in the Running state while it printed its message character by character. When Task A called xTaskNotifyGive(), Task B received its notification and changed from Blocked to Ready. In Scenario A, Task B had priority 2, which was higher than Task A's priority 1. Therefore, Task B was selected by the scheduler and preempted Task A. The Serial Monitor showed Task B's message before Task A had finished printing.

In Scenario B, both tasks had priority 1. Task B did not have a higher priority than Task A, so the notification alone did not cause priority-based preemption. However, Task A later called vTaskDelay(), causing Task A to become Blocked temporarily. This allowed the Ready Task B to run. The Serial Monitor output became interleaved, showing that both tasks were executing.

In Scenario C, Task B had priority 3. It again preempted Task A after receiving the notification because it had the higher priority. After Task B finished, Task A continued from where it had stopped.

7. Screenshot
The Serial Monitor screenshot from the experiment is stored in:

images/serial-monitor.png

Scheduler Behaviour Explanation
In this experiment, two FreeRTOS tasks were created on the same ESP32-S3 CPU core. Task A had priority 1, while Task B was tested with priorities 2, 1, and 3. At the beginning, Task B was in the Blocked state because it was waiting for a notification using ulTaskNotifyTake(). A blocked task does not use the CPU because it is waiting for an event.

Task A was in the Running state while it printed its message character by character. When Task A called xTaskNotifyGive(), Task B received its notification and changed from Blocked to Ready. In Scenario A, Task B had priority 2, which was higher than Task A's priority 1. Therefore, Task B was selected by the scheduler and preempted Task A. The Serial Monitor showed Task B's message before Task A had finished printing.

In Scenario B, both tasks had priority 1. Task B did not have a higher priority than Task A, so the notification alone did not cause priority-based preemption. However, Task A later called vTaskDelay(), causing Task A to become Blocked temporarily. This allowed the Ready Task B to run. The Serial Monitor output became interleaved, showing that both tasks were executing.

In Scenario C, Task B had priority 3. It again preempted Task A after receiving the notification because it had the higher priority. After Task B finished, Task A continued from where it had stopped. This demonstrates how the FreeRTOS scheduler manages Running, Ready, and Blocked tasks.

// FreeRTOS Scheduler Demonstration
// ESP32-S3
// Task A: Priority 1
// Task B: Priority 2

TaskHandle_t taskAHandle;
TaskHandle_t taskBHandle;

// -------------------------
// Task A
// -------------------------
void taskA(void *parameter)
{
  while (true)
  {
    Serial.println("\nTask A is printing slowly...");

    Serial.print("Task A: ");
    Serial.flush();

    // Print character by character
    const char *message = "Hello from Task A";

    for (int i = 0; message[i] != '\0'; i++)
    {
      Serial.print(message[i]);
      Serial.flush();

      // Small delay so we can observe the scheduler
      vTaskDelay(pdMS_TO_TICKS(100));

      // Notify Task B during Task A's message
      if (i == 5)
      {
        Serial.println("\nTask A: notifying Task B");
        xTaskNotifyGive(taskBHandle);
      }
    }

    Serial.println();
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// -------------------------
// Task B
// -------------------------
void taskB(void *parameter)
{
  while (true)
  {
    // Wait for Task A notification
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

    Serial.println(">>> Task B is running!");
    Serial.println(">>> Task B received notification.");
  }
}

// -------------------------
// Setup
// -------------------------
void setup()
{
  Serial.begin(115200);

  // Give Serial time to start
  vTaskDelay(pdMS_TO_TICKS(1000));

  Serial.println("\n=== FreeRTOS Scheduler Demo ===");

  // Both tasks are pinned to the SAME core
  xTaskCreatePinnedToCore(
    taskA,          // Task function
    "Task A",       // Task name
    4096,           // Stack size
    NULL,           // Parameter
    1,              // Priority
    &taskAHandle,   // Task handle
    0               // Core 0
  );

  xTaskCreatePinnedToCore(
    taskB,          // Task function
    "Task B",       // Task name
    4096,           // Stack size
    NULL,           // Parameter
    3,              // Priority
    &taskBHandle,   // Task handle
    0               // Core 0
  );
}

void loop()
{
  // Nothing needed here.
  vTaskDelay(pdMS_TO_TICKS(1000));
}
