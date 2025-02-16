#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <unistd.h>
#include <syslog.h>
#include <libconfig.h>

#define CONFIG_FILE "/etc/gcr/gcrserver.conf"

void read_config(char *address, int *port) {
    config_t cfg;
    config_init(&cfg);

    if (!config_read_file(&cfg, CONFIG_FILE)) {
        syslog(LOG_ERR, "Erreur lors de la lecture du fichier de configuration : %s:%d - %s",
               config_error_file(&cfg),
               config_error_line(&cfg),
               config_error_text(&cfg));
        config_destroy(&cfg);
        exit(1);
    }

    const char *addr;
    if (config_lookup_string(&cfg, "address", &addr)) {
        strcpy(address, addr);
    } else {
        syslog(LOG_ERR, "Erreur: Paramètre 'address' non trouvé.");
        exit(2);
    }

    if (!config_lookup_int(&cfg, "port", port)) {
        syslog(LOG_ERR, "Erreur: Paramètre 'port' non trouvé.");
        exit(3);
    }

    config_destroy(&cfg);
}

int main() {
    char address[16];
    int port;

    // Lire la configuration
    read_config(address, &port);

    // Transformer en daemon
    if (daemon(0, 0) < 0) {
        perror("Erreur lors du passage en daemon");
        exit(4);
    }

    // Initialisation du logging
    openlog("gcrsh_server", LOG_PID | LOG_CONS, LOG_DAEMON);
    syslog(LOG_NOTICE, "Serveur démarré en mode daemon. Écoute sur %s:%d", address, port);

    int server_sock, client_sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);

    // Création de la socket serveur
    server_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (server_sock < 0) {
        syslog(LOG_ERR, "Erreur lors de la création de la socket");
        exit(5);
    }

    // Configuration de l'adresse du serveur
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = inet_addr(address);
    server_addr.sin_port = htons(port);

    // Liaison de la socket au port et adresse spécifiés
    if (bind(server_sock, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        syslog(LOG_ERR, "Erreur lors de la liaison de la socket");
        exit(6);
    }

    // Mise en écoute des connexions entrantes
    if (listen(server_sock, 5) < 0) {
        syslog(LOG_ERR, "Erreur lors de la mise en écoute");
        exit(7);
    }

    syslog(LOG_NOTICE, "Serveur en attente de connexions...");

    while (1) {
        // Acceptation d'une connexion entrante
        client_sock = accept(server_sock, (struct sockaddr*)&client_addr, &client_len);
        if (client_sock < 0) {
            syslog(LOG_ERR, "Erreur lors de l'acceptation de la connexion");
            continue;
        }

        syslog(LOG_NOTICE, "Connexion reçue de %s:%d", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));

        // Création d'un processus fils pour gérer la connexion
        int pid = fork();

        if (pid < 0) {
            syslog(LOG_ERR, "Erreur lors de la création du processus fils");
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
            syslog(LOG_ERR, "Erreur lors de l'exécution du shell");
            exit(8);
        } else {  // Processus parent
            close(client_sock);  // Le parent ferme la socket client et continue à écouter
        }
    }

    closelog();
    return 0;
}
