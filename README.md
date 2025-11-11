// Programa: Controle de Estoque (arquivo: estoque.txt)
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX_ITEMS 1000
#define MAX_NAME 200
#define STOCK_FILE "estoque.txt"

typedef struct {
    char name[MAX_NAME];
    int qty;
} Item;

static void trim(char *s) {
    // remove trailing newline and spaces
    size_t i, len = strlen(s);
    while (len > 0 && (s[len-1] == '\n' || s[len-1] == '\r' || isspace((unsigned char)s[len-1]))) {
        s[--len] = '\0';
    }
    // remove leading spaces
    i = 0;
    while (s[i] && isspace((unsigned char)s[i])) i++;
    if (i) memmove(s, s + i, strlen(s + i) + 1);
}

static int read_stock(Item items[], int max_items) {
    FILE *f = fopen(STOCK_FILE, "r");
    if (!f) return 0;
    char line[MAX_NAME];
    int count = 0;
    while (fgets(line, sizeof(line), f)) {
        if (count >= max_items) break;
        trim(line);
        if (line[0] == '\0') continue;
        strncpy(items[count].name, line, MAX_NAME-1);
        items[count].name[MAX_NAME-1] = '\0';
        if (!fgets(line, sizeof(line), f)) break;
        trim(line);
        items[count].qty = atoi(line);
        count++;
    }
    fclose(f);
    return count;
}

static int write_stock(Item items[], int count) {
    FILE *f = fopen(STOCK_FILE, "w");
    if (!f) return 0;
    for (int i = 0; i < count; ++i) {
        fprintf(f, "%s\n%d\n", items[i].name, items[i].qty);
    }
    fclose(f);
    return 1;
}

static void read_line_prompt(const char *prompt, char *buf, size_t sz) {
    printf("%s", prompt);
    if (!fgets(buf, (int)sz, stdin)) { buf[0] = '\0'; return; }
    trim(buf);
}

static int read_int_prompt(const char *prompt) {
    char buf[64];
    while (1) {
        printf("%s", prompt);
        if (!fgets(buf, sizeof(buf), stdin)) exit(0);
        trim(buf);
        if (buf[0] == '\0') { printf("Entrada inválida. Tente novamente.\n"); continue; }
        char *end;
        long v = strtol(buf, &end, 10);
        while (*end && isspace((unsigned char)*end)) end++;
        if (*end != '\0') { printf("Entrada inválida. Tente novamente.\n"); continue; }
        return (int)v;
    }
}

static int read_menu_option(void) {
    char buf[64];
    while (1) {
        printf("Opção: ");
        if (!fgets(buf, sizeof(buf), stdin)) exit(0);
        trim(buf);
        if (buf[0] == '\0') { printf("Opção inválida! Tente novamente.\n"); continue; }
        char *end;
        long v = strtol(buf, &end, 10);
        while (*end && isspace((unsigned char)*end)) end++;
        if (*end != '\0' || v < 1 || v > 4) { printf("Opção inválida! Tente novamente.\n"); continue; }
        return (int)v;
    }
}

int main(void) {
    Item items[MAX_ITEMS];
    int count;
    char resp[16];

    for (;;) {
#ifdef _WIN32
        system("cls");
#endif
        printf("================================\n");
        printf("   Controle de Estoque\n");
        printf("================================\n");
        printf("Selecione uma opção:\n");
        printf("1. Adicionar Item\n");
        printf("2. Remover Item\n");
        printf("3. Listar Estoque\n");
        printf("4. Sair\n");
        int opc = read_menu_option();

        if (opc == 4) {
            printf("Obrigado por usar o Controle de Estoques! Até a próxima.\n");
            break;
        }

        if (opc == 1) { /* Adicionar Item */
            char name[MAX_NAME];
            read_line_prompt("Digite o nome do item: ", name, sizeof(name));
            if (name[0] == '\0') { printf("Nome inválido.\n"); goto ask_again; }
            int q = read_int_prompt("Digite a quantidade: ");
            if (q < 0) { printf("Quantidade inválida.\n"); goto ask_again; }

            count = read_stock(items, MAX_ITEMS);
            int found = -1;
            for (int i = 0; i < count; ++i) {
                if (strcmp(items[i].name, name) == 0) { found = i; break; }
            }
            if (found >= 0) {
                items[found].qty += q;
            } else {
                if (count < MAX_ITEMS) {
                    strncpy(items[count].name, name, MAX_NAME-1);
                    items[count].name[MAX_NAME-1] = '\0';
                    items[count].qty = q;
                    count++;
                } else {
                    printf("Estoque cheio. Não é possível adicionar mais itens.\n");
                    goto ask_again;
                }
            }
            if (!write_stock(items, count)) {
                printf("Erro ao acessar o arquivo de estoque.\n");
            } else {
                printf("Item adicionado com sucesso!\n");
            }
        }
        else if (opc == 2) { /* Remover Item */
            char name[MAX_NAME];
            read_line_prompt("Digite o nome do item: ", name, sizeof(name));
            if (name[0] == '\0') { printf("Nome inválido.\n"); goto ask_again; }
            int q = read_int_prompt("Digite a quantidade a ser removida: ");
            if (q <= 0) { printf("Quantidade inválida.\n"); goto ask_again; }

            count = read_stock(items, MAX_ITEMS);
            int found = -1;
            for (int i = 0; i < count; ++i) {
                if (strcmp(items[i].name, name) == 0) { found = i; break; }
            }
            if (found < 0) {
                printf("Item não encontrado.\n");
            } else {
                if (items[found].qty < q) {
                    printf("Estoque insuficiente. Quantidade disponível: %d\n", items[found].qty);
                } else {
                    items[found].qty -= q;
                    if (items[found].qty == 0) {
                        // remove item by shifting
                        for (int j = found; j < count - 1; ++j) items[j] = items[j+1];
                        count--;
                        if (!write_stock(items, count)) printf("Erro ao acessar o arquivo de estoque.\n");
                        else printf("Item removido do estoque!\n");
                    } else {
                        if (!write_stock(items, count)) printf("Erro ao acessar o arquivo de estoque.\n");
                        else printf("Quantidade atualizada com sucesso!\n");
                    }
                }
            }
        }
        else if (opc == 3) { /* Listar Estoque */
            count = read_stock(items, MAX_ITEMS);
            if (count == 0) {
                printf("O estoque está vazio.\n");
            } else {
                printf("=============================\n");
                printf("        Estoque Atual\n");
                printf("=============================\n");
                for (int i = 0; i < count; ++i) {
                    printf("Nome: %s\n", items[i].name);
                    printf("Quantidade: %d\n\n", items[i].qty);
                }
            }
        }

    ask_again:
        while (1) {
            printf("Deseja realizar outra operação? (s/n): ");
            if (!fgets(resp, sizeof(resp), stdin)) exit(0);
            trim(resp);
            if (resp[0] == '\0') { printf("Resposta inválida. Por favor, digite 's' para sim ou 'n' para não.\n"); continue; }
            char c = tolower((unsigned char)resp[0]);
            if (c == 's') break; // volta ao menu
            else if (c == 'n') { printf("Obrigado por usar o Controle de Estoques! Até a próxima.\n"); return 0; }
            else { printf("Resposta inválida. Por favor, digite 's' para sim ou 'n' para não.\n"); }
        }
    }

    return 0;
}
