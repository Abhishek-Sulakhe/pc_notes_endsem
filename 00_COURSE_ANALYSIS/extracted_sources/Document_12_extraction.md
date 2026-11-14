# Source Extraction: Document 12.pdf

**Source File:** `input_notes/Document 12.pdf`  
**Total Pages:** 5  
**Topic:** MPI-2 Dynamic Process Management / Inter-Communicator Connection via Ports (Client-Server Architecture)  

---

## Page 01

### Topic: MPI (Message Passing Interface) Continued — How your sender works
**Context Header:** HTTP port? [P2]

### Conceptual Explanation:
The note outlines the step-by-step workflow of an MPI sender process acting as a server waiting for an independent receiver process to connect:
1. **Initialize MPI:**
   `MPI_Init(&argc, &argv)`
2. **Open a communication port:**
   `MPI_Open_port(MPI_INFO_NULL, port)`
3. **Save the port name:**
   Write the port string into `port.txt` (or publish it to an MPI name service).
4. **Wait for the receiver:**
   `MPI_Comm_accept(...)`
5. **Send the integer:**
   `MPI_Send(&data, 1, MPI_INT, 0, 0, client)`
   Value to send: `data = 100`
6. **Disconnect & finalize:**
   `MPI_Comm_disconnect() -> MPI_Close_port() -> MPI_Finalize()`

---

## Page 02

### Topic: ompi-server and Cross-Communicator Connections
**Top Notes:**
- "lookup service"
- "DNS: Domain name service in OS"
- "management services"

### Notes Content:
- `ompi-server`:
  - "Normally, MPI processes communicate using a communicator such as `MPI_COMM_WORLD`."
  - "But sometimes I have 2 MPI communications and want to connect to each other."
- Commands used to establish connection via Open MPI server daemon:
  - Starting the ompi-server:
    ```bash
    ompi-server --no-daemonize -report-uri /tmp/ompi-server.uri
    ```
  - Running sender with the server URI:
    ```bash
    mpirun -mca pmix_server_uri "$(cat /tmp/ompi-server.uri)" -np 1 ./sender
    ```
  - Verification:
    ```bash
    cat /tmp/ompi-server.uri
    ```

---

## Page 03

### Topic: MPI Dynamic Connection Architecture
**Content:**
- Sender process functions:
  - `MPI_Open_port()`
  - `MPI_Comm_accept()`
- Concept Header: **Dynamic Connection**
- **What these 2 programs do:**
  "2 independently launched MPI programs establish a connection & exchange an integer."
- **Key MPI Concepts:**
  - `MPI_Open_port()`
  - `MPI_Comm_accept()`
  - `MPI_Comm_connect()`

---

## Page 04

### Topic: Sender Implementation (`sender.c`)
**Full C Code Snippet:**
```c
int main(int argc, char *argv[]) {
    MPI_Comm client;
    MPI_Status status;
    int data = 100;
    char port[MPI_MAX_PORT_NAME];

    MPI_Init(&argc, &argv);

    // Open a port for communication
    MPI_Open_port(MPI_INFO_NULL, port);
    printf("port: %s\n", port);
    printf("waiting for receiver...\n");
    fflush(stdout);

    // Save the port name for the receiver
    FILE *fp = fopen("port.txt", "w");
    fprintf(fp, "%s\n", port);
    fclose(fp);

    // Wait for the receiver to connect
    MPI_Comm_accept(port, MPI_INFO_NULL, 0, MPI_COMM_SELF, &client);

    // Send data
    MPI_Send(&data, 1, MPI_INT, 0, 0, client);
    printf("Sender sent: %d\n", data);
```

---

## Page 05

### Topic: Sender Finalization & Receiver Implementation (`receiver.c`)
**Continuation of `sender.c`:**
```c
    MPI_Comm_disconnect(&client);
    MPI_Close_port(port);
    MPI_Finalize();

    return 0;
}
```

**Receiver Implementation (`receiver.c`):**
```c
int main(int argc, char *argv[]) {
    MPI_Comm server;
    int data;
    char port[MPI_MAX_PORT_NAME];

    MPI_Init(&argc, &argv);

    // Read the sender's port name
    FILE *fp = fopen("port.txt", "r");
    if (fp == NULL) {
        printf("run sender first!\n");
        MPI_Abort(MPI_COMM_WORLD, 1);
    }
    fscanf(fp, "%s", port);
    fclose(fp);

    // Connect to sender
    MPI_Comm_connect(port, MPI_INFO_NULL, 0, MPI_COMM_SELF, &server);
    MPI_Recv(&data, 1, MPI_INT, 0, 0, server, MPI_STATUS_IGNORE);
    printf("Receiver received: %d\n", data);

    MPI_Comm_disconnect(&server);
    MPI_Finalize();
    return 0;
}
```
