# Codes by :- [Suzen Malik Firasta](https://github.com/SuzenFirasta)
## 1. Write a MPI program using synchronous send. The sender process sends a word to the receiver. The second process receives the word, toggles each letter of the word and sends it back to the first process. Both processes use synchronous send operations.

```cpp
#include <stdio.h>
#include <string.h>
#include <ctype.h>
#include <mpi.h>

void toggle_case(char* word) {
    for (int i = 0; word[i] != '\0'; i++) {
        if (isupper(word[i]))
            word[i] = tolower(word[i]);
        else if (islower(word[i]))
            word[i] = toupper(word[i]);
    }
}

int main(int argc, char *argv[]) {
    MPI_Init(&argc, &argv);

    int rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    const int tag = 0;
    const int root = 0;
    
    if (rank == root) {
        // Sender process
        const char* word = "Hello";
        printf("Process %d sends word: %s\n", rank, word);
        MPI_Ssend(word, strlen(word) + 1, MPI_CHAR, 1, tag, MPI_COMM_WORLD);
        
        // Receive the toggled word back from process 1
        char received_word[100];
        MPI_Recv(received_word, 100, MPI_CHAR, 1, tag, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        printf("Process %d received toggled word: %s\n", rank, received_word);
        
    } else if (rank == 1) {
        // Receiver process
        char word[100];
        MPI_Recv(word, 100, MPI_CHAR, 0, tag, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        printf("Process %d received word: %s\n", rank, word);
        
        // Toggle the case of each letter in the word
        toggle_case(word);
        
        // Send it back to the sender process
        MPI_Ssend(word, strlen(word) + 1, MPI_CHAR, 0, tag, MPI_COMM_WORLD);
    }

    MPI_Finalize();
    return 0;
}
```
## Output
```plaintext
C:\Users\suzen>cd C:\Users\suzen\source\repos\week7\Q1\x64\Debug

C:\Users\suzen\source\repos\week7\Q1\x64\Debug>mpiexec -n 2 Q1.exe
Process 1 received word: Hello
Process 0 sends word: Hello
Process 0 received toggled word: hELLO
```
## Explanatioin
- The program initializes MPI environment and obtains the rank of the process using `MPI_Comm_rank()`.

- It defines a tag for message identification and specifies the root process as process 0.

- If the process rank is the root (rank 0), it acts as the sender process:
  - It defines the word to be sent as "Hello".
  - It sends the word to process 1 using synchronous send (`MPI_Ssend()`).
  - It receives the toggled word back from process 1 using blocking receive (`MPI_Recv()`).
  - It prints the received toggled word.

- If the process rank is 1, it acts as the receiver process:
  - It receives the word from process 0 using blocking receive (`MPI_Recv()`).
  - It prints the received word.
  - It toggles the case of each letter in the received word using the `toggle_case()` function.
  - It sends the toggled word back to process 0 using synchronous send (`MPI_Ssend()`).

- Finally, MPI environment is finalized using `MPI_Finalize()`.

---
## 2. Write a MPI program where the master process (process 0) sends a number to each of the slaves and the slave processes receive the number and prints it. Use standard send.

```cpp
#include <mpi.h>
#include <stdio.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);
    
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

    int number;
    if (world_rank == 0) {
        // Master process
        // Choose a number to send to all slave processes
        number = 777;  // You can change this number to any number you want to send

        // Use MPI_Send to send it to all the other processes
        for (int i = 1; i < world_size; i++) {
            MPI_Send(&number, 1, MPI_INT, i, 0, MPI_COMM_WORLD);
            printf("Master process sending number %d to process %d\n", number, i);
        }
    } else {
        // Slave processes
        MPI_Recv(&number, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        printf("Slave process %d received number %d from master process\n", world_rank, number);
    }

    MPI_Finalize();
    return 0;
}
```
## Output
```plaintext
C:\Users\suzen\source\repos\week7\Q1\x64\Debug>cd C:\Users\suzen\source\repos\week7\Q2\x64\Debug\

C:\Users\suzen\source\repos\week7\Q2\x64\Debug>mpiexec -n 4 Q2.exe
Slave process 3 received number 777 from master process
Slave process 2 received number 777 from master process
Slave process 1 received number 777 from master process
Master process sending number 777 to process 1
Master process sending number 777 to process 2
Master process sending number 777 to process 3
```
Explanation : 

- The program initializes MPI environment and obtains the size and rank of the MPI communicator `MPI_COMM_WORLD`.

- The master process (rank 0) chooses a number to send to all slave processes. In this case, the number chosen is 777.

- The master process uses `MPI_Send()` in a loop to send the chosen number to all the other processes except itself.

- Each slave process (with rank > 0) receives the number sent by the master process using `MPI_Recv()`. It specifies the source rank as 0 (the master process).

- Each slave process prints the received number along with its own rank.

- Finally, MPI environment is finalized using `MPI_Finalize()`.
---

## 3. Write a MPI program to read N elements of the array in the root process (process 0) where N is equal to the total number of process. The root process sends one value to each of the slaves. Let even ranked process finds square of the received element and odd ranked process finds cube of received element. Use Buffered send.

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>

int main(int argc, char *argv[]) {
    MPI_Init(&argc, &argv);

    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

    // The root process (process 0) will read N elements where N is equal to world_size
    int *numbers = NULL;
    if (world_rank == 0) {
         numbers = (int*)malloc(sizeof(int) * world_size);

        // Initialize the array with some values
        for (int i = 0; i < world_size; i++) {
            numbers[i] = i + 1; // Or any other logic to initialize the array
        }
    }

    // Buffer for buffered send
    int buffer_size = world_size * sizeof(int) + MPI_BSEND_OVERHEAD;

    void *buffer = malloc(buffer_size);
    MPI_Buffer_attach(buffer, buffer_size);

    // Distribute one number to each slave process
    if (world_rank == 0) {
        for (int i = 1; i < world_size; i++) {
            MPI_Bsend(&numbers[i], 1, MPI_INT, i, 0, MPI_COMM_WORLD);
        }
    }

    // Each slave process receives a number and performs its operation
    int received_number;
    if (world_rank != 0) {
        MPI_Recv(&received_number, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        if (world_rank % 2 == 0) {
            // Even rank: square the number
            printf("Process %d received %d, squaring it to %d\n", world_rank, received_number, received_number * received_number);
        } else {
            // Odd rank: cube the number
            printf("Process %d received %d, cubing it to %d\n", world_rank, received_number, received_number * received_number * received_number);
        }
    }

    // Detach and free the buffer
    MPI_Buffer_detach(&buffer, &buffer_size);
    free(buffer);
    
    // Free numbers array on root process
    if(world_rank == 0) {
        free(numbers);
    }

    MPI_Finalize();
    return 0;
}
```
## Output
```plaintext
C:\Users\suzen\source\repos\week7\Q2\x64\Debug>cd C:\Users\suzen\source\repos\week7\Q3\x64\Debug\

C:\Users\suzen\source\repos\week7\Q3\x64\Debug>mpiexec -n 4 Q3.exe
Process 1 received 2, cubing it to 8
Process 3 received 4, cubing it to 64
Process 2 received 3, squaring it to 9
```
## Explanation

- The program initializes MPI environment, obtains the size and rank of the MPI communicator `MPI_COMM_WORLD`.

- The root process (process 0) allocates memory to store an array of integers named `numbers`, where the size of the array is equal to the total number of processes.

- The root process initializes the array with some values. In this case, the array is initialized with numbers from 1 to `world_size`.

- A buffer is allocated for buffered send operations. The buffer size is calculated based on the number of elements to be sent (`world_size`) and the MPI overhead.

- Buffered send is used to distribute one number to each slave process (processes with rank > 0). The root process sends each number to the respective process using `MPI_Bsend()`.

- Each slave process receives the number sent by the root process using `MPI_Recv()`. Based on the rank of the process, even-ranked processes square the received number, while odd-ranked processes cube the received number. The result is printed accordingly.

- The buffer is detached and freed after sending/receiving operations.

- Memory allocated for the `numbers` array is freed on the root process.

- Finally, MPI environment is finalized using `MPI_Finalize()`.
---
## 4. Write a MPI program to read an integer value in the root process. Root process sends this value to Process1, Process1 sends this value to Process2 and so on. Last process sends the value back to root process. When sending the value each process will first increment the received value by one. Write the program using point to point communication routines. 

```cpp
#include <stdio.h>
#include <mpi.h>

int main(int argc, char** argv) {
    int value;
    int world_rank, world_size;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    if (world_rank == 0) {
        // Root process reads the integer value
        value = 10; // Example value, can be read from user input
        printf("Root process starts with value: %d\n", value);
        MPI_Send(&value, 1, MPI_INT, world_rank + 1, 0, MPI_COMM_WORLD);
    } else {
        MPI_Recv(&value, 1, MPI_INT, world_rank - 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        value++; // Increment the value by one
        printf("Process %d incremented value to: %d\n", world_rank, value);

        if (world_rank < world_size - 1) {
            MPI_Send(&value, 1, MPI_INT, world_rank + 1, 0, MPI_COMM_WORLD);
        } else {
            // Last process sends it back to the root process
            MPI_Send(&value, 1, MPI_INT, 0, 0, MPI_COMM_WORLD);
        }
    }

    // Root process receives the value from the last process
    if (world_rank == 0) {
        MPI_Recv(&value, 1, MPI_INT, world_size - 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        printf("Root process received final value: %d\n", value);
    }

    MPI_Finalize();
    return 0;
}
```
## Output
```plaintext
C:\Users\suzen\source\repos\week7\Q3\x64\Debug>cd C:\Users\suzen\source\repos\week7\Q4\x64\Debug\

C:\Users\suzen\source\repos\week7\Q4\x64\Debug>mpiexec -n 4 Q4.exe
Process 1 incremented value to: 11
Root process starts with value: 10
Root process received final value: 13
Process 3 incremented value to: 13
Process 2 incremented value to: 12
```
## Explanation 

- The program initializes MPI environment and obtains the rank and size of the MPI communicator `MPI_COMM_WORLD`.

- If the process rank is 0 (root process), it initializes the value to be sent. In this example, the value is set to 10. The root process then sends this value to process 1 using `MPI_Send()`.

- If the process rank is not 0 (i.e., it's a non-root process), it receives the value from the previous process using `MPI_Recv()`. Each process increments the received value by one.

- Each non-root process sends the updated value to the next process using `MPI_Send()`. The last process sends the updated value back to the root process.

- The root process receives the final value from the last process using `MPI_Recv()`.

- All processes print the received or final value along with their rank.

- Finally, MPI environment is finalized using `MPI_Finalize()`.

---
## 5. Write a MPI program to read N elements of an array in the master process. Let N processes including master process check the array values are prime or not.

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>

// Function to check if a number is prime
int is_prime(int number) {
    if (number <= 1) return 0;
    for (int i = 2; i * i <= number; i++) {
        if (number % i == 0) return 0;
    }
    return 1;
}

int main(int argc, char** argv) {
    int world_rank, world_size, number_to_check;
    int* array = NULL; // Initialize the pointer to NULL

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    if (world_rank == 0) {
        array = (int*)malloc(sizeof(int) * world_size); // Allocate memory only on the root process
        for (int i = 0; i < world_size; i++) {
            array[i] = i + 2; // Example values, start from 2 (first prime number)
        }
    }

    // Use MPI_Scatter to distribute the values
    MPI_Scatter(array, 1, MPI_INT, &number_to_check, 1, MPI_INT, 0, MPI_COMM_WORLD);

    // Each process checks if the number received is prime
    int result = is_prime(number_to_check);
    printf("Process %d received %d, prime? %s\n", world_rank, number_to_check, result ? "Yes" : "No");

    // Free the allocated memory on the root process
    if (world_rank == 0) {
        free(array);
    }

    MPI_Finalize();
    return 0;
}
```
## Output
```plaintext
C:\Users\suzen\source\repos\week7\Q5\x64\Debug>mpiexec -n 4 Q5.exe
Process 1 received 3, prime? Yes
Process 2 received 4, prime? No
Process 0 received 2, prime? Yes
Process 3 received 5, prime? Yes
```
## Explanation
- The program initializes MPI environment and obtains the rank and size of the MPI communicator `MPI_COMM_WORLD`.

- In the master process (process with rank 0), memory is allocated dynamically to store an array of integers named `array`. Each element of the array represents a number to be checked for primality. In this example, the array is initialized with values starting from 2 (the first prime number) up to the size of the MPI communicator `world_size`.

- The `MPI_Scatter()` function is used to distribute the values of the array to all processes. Each process receives one element of the array (`number_to_check`) from the master process.

- Each process then checks whether the received number is prime or not using the `is_prime()` function.

- The result of the primality check is printed along with the rank of the process.

- Finally, memory allocated for the array is freed in the master process, and MPI environment is finalized using `MPI_Finalize()`.
---
