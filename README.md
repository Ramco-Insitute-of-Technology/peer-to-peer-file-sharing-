# peer-to-peer-file-sharing-
#SERVER code:

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define SOCKET_PATH "p2p_socket"
#define BUFFER_SIZE 1024

void handle_client(int client_socket) {
    char buffer[BUFFER_SIZE];
    int bytes_read;
    FILE *file;

    // Read file name
    bytes_read = read(client_socket, buffer, BUFFER_SIZE);
    buffer[bytes_read] = '\0';

    if (strncmp(buffer, "upload ", 7) == 0) {
        char *file_name = buffer + 7;
        file = fopen(file_name, "wb");

        // Receive file content
        while ((bytes_read = read(client_socket, buffer, BUFFER_SIZE)) > 0) {
            fwrite(buffer, 1, bytes_read, file);
        }

        fclose(file);
        printf("File %s received.\n", file_name);

    } else if (strncmp(buffer, "download ", 9) == 0) {
        char *file_name = buffer + 9;
        file = fopen(file_name, "rb");

        if (file) {
            // Send file content
            while ((bytes_read = fread(buffer, 1, BUFFER_SIZE, file)) > 0) {
                write(client_socket, buffer, bytes_read);
            }
            fclose(file);
            printf("File %s sent.\n", file_name);
        } else {
            printf("File %s not found.\n", file_name);
        }
    }

    close(client_socket);
}

int main() {
    int server_socket, client_socket;
    struct sockaddr_un server_addr;

    // Create socket
    server_socket = socket(AF_UNIX, SOCK_STREAM, 0);
    if (server_socket == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // Bind socket
    memset(&server_addr, 0, sizeof(struct sockaddr_un));
    server_addr.sun_family = AF_UNIX;
    strncpy(server_addr.sun_path, SOCKET_PATH, sizeof(server_addr.sun_path) - 1);

    unlink(SOCKET_PATH);
    if (bind(server_socket, (struct sockaddr *)&server_addr, sizeof(struct sockaddr_un)) == -1) {
        perror("bind");
        exit(EXIT_FAILURE);
    }

    // Listen for connections
    if (listen(server_socket, 5) == -1) {
        perror("listen");
        exit(EXIT_FAILURE);
    }

    printf("Server is listening...\n");

    while (1) {
        // Accept connection
        client_socket = accept(server_socket, NULL, NULL);
        if (client_socket == -1) {
            perror("accept");
            exit(EXIT_FAILURE);
        }

        // Handle client in a new process
        if (fork() == 0) {
            close(server_socket);
            handle_client(client_socket);
            exit(EXIT_SUCCESS);
        } else {
            close(client_socket);
        }
    }

    close(server_socket);
    unlink(SOCKET_PATH);

    return 0;
}


#CLIENT code:

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define SOCKET_PATH "p2p_socket"
#define BUFFER_SIZE 1024

void upload_file(int socket, const char *file_name) {
    char buffer[BUFFER_SIZE];
    FILE *file;

    // Send command
    snprintf(buffer, sizeof(buffer), "upload %s", file_name);
    write(socket, buffer, strlen(buffer));

    // Send file content
    file = fopen(file_name, "rb");
    if (file) {
        while (!feof(file)) {
            int bytes_read = fread(buffer, 1, BUFFER_SIZE, file);
            write(socket, buffer, bytes_read);
        }
        fclose(file);
        printf("File %s uploaded.\n", file_name);
    } else {
        perror("fopen");
    }
}

void download_file(int socket, const char *file_name) {
    char buffer[BUFFER_SIZE];
    FILE *file;
    int bytes_read;

    // Send command
    snprintf(buffer, sizeof(buffer), "download %s", file_name);
    write(socket, buffer, strlen(buffer));

    // Receive file content
    file = fopen(file_name, "wb");
    if (file) {
        while ((bytes_read = read(socket, buffer, BUFFER_SIZE)) > 0) {
            fwrite(buffer, 1, bytes_read, file);
        }
        fclose(file);
        printf("File %s downloaded.\n", file_name);
    } else {
        perror("fopen");
    }
}

int main(int argc, char *argv[]) {
    int client_socket;
    struct sockaddr_un server_addr;

    if (argc < 3) {
        fprintf(stderr, "Usage: %s <upload|download> <file_name>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    // Create socket
    client_socket = socket(AF_UNIX, SOCK_STREAM, 0);
    if (client_socket == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // Connect to server
    memset(&server_addr, 0, sizeof(struct sockaddr_un));
    server_addr.sun_family = AF_UNIX;
    strncpy(server_addr.sun_path, SOCKET_PATH, sizeof(server_addr.sun_path) - 1);

    if (connect(client_socket, (struct sockaddr *)&server_addr, sizeof(struct sockaddr_un)) == -1) {
        perror("connect");
        exit(EXIT_FAILURE);
    }

    if (strcmp(argv[1], "upload") == 0) {
        upload_file(client_socket, argv[2]);
    } else if (strcmp(argv[1], "download") == 0) {
        download_file(client_socket, argv[2]);
    } else {
        fprintf(stderr, "Unknown command: %s\n", argv[1]);
    }

    close(client_socket);

    return 0;
}
