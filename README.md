# Gerenciador_de_Estoque_atualizado
Atualizado com cadastrar, consultar, excluir e atualizar.
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <locale.h>
using namespace std;
/* ---------------------------------------------------------------------------
   ESTRUTURA DE DADOS — PRODUTO
--------------------------------------------------------------------------- */
typedef struct {
    char nome[50];
    int codigo;
    float preco;
    char status;   /* ' ' = ativo | '*' = excluído */
} Produto;

/* PROTÓTIPOS */
void configurarLocale(void);
void limpaBuffer(void);
void lerString(char *s, int tam);
int tamanho(FILE *arq);
void cadastrar(FILE *arq);
void consultar(FILE *arq);
void excluir(FILE *arq);
void gerarRelatorioTxt(FILE *arq);

/* ---------------------------------------------------------------------------
   limpaBuffer — limpa lixo deixado pelo scanf
--------------------------------------------------------------------------- */
void limpaBuffer(void) {
    int c;
    while ((c = getchar()) != '\n' && c != EOF);
}

/* ---------------------------------------------------------------------------
   lerString — leitura segura com fgets + remoção do '\n'
--------------------------------------------------------------------------- */
void lerString(char *s, int tam) {
    fgets(s, tam, stdin);
    s[strcspn(s, "\n")] = '\0';
}

/* ---------------------------------------------------------------------------
   tamanho — quantidade total de registros no arquivo
--------------------------------------------------------------------------- */
int tamanho(FILE *arq) {
    long pos = ftell(arq);
    fseek(arq, 0, SEEK_END);
    long fim = ftell(arq);
    fseek(arq, pos, SEEK_SET);

    return (int)(fim / sizeof(Produto));
}

/* ---------------------------------------------------------------------------
   cadastrar — salva um novo registro no final do arquivo
--------------------------------------------------------------------------- */
void cadastrar(FILE *arq) {
    Produto p;
    char confirma;

    p.status = ' ';

    printf("\n=== CADASTRAR PRODUTO ===\n");
    printf("Registro número: %d\n", tamanho(arq) + 1);

    printf("Nome do produto: ");
    lerString(p.nome, sizeof(p.nome));

    printf("Código numérico: ");
    scanf("%d", &p.codigo);
    limpaBuffer();

    printf("Preço: ");
    scanf("%f", &p.preco);
    limpaBuffer();

    printf("Confirmar cadastro (s/n)? ");
    scanf("%c", &confirma);
    limpaBuffer();

    if (toupper(confirma) == 'S') {
        fseek(arq, 0, SEEK_END);
        fwrite(&p, sizeof(Produto), 1, arq);
        fflush(arq);

        printf("Produto cadastrado com sucesso!\n");
    } else {
        printf("Cadastro cancelado.\n");
    }
}

/* ---------------------------------------------------------------------------
   consultar — acesso direto via fseek
--------------------------------------------------------------------------- */
void consultar(FILE *arq) {
    int nr;
    Produto p;

    printf("\nInforme o código do registro: ");
    if (scanf("%d", &nr) != 1) {
        printf("Entrada inválida!\n");
        limpaBuffer();
        return;
    }
    limpaBuffer();

    if (nr <= 0 || nr > tamanho(arq)) {
        printf("Código inválido!\n");
        return;
    }

    fseek(arq, (long)(nr - 1) * sizeof(Produto), SEEK_SET);

    if (fread(&p, sizeof(Produto), 1, arq) != 1) {
        printf("Erro ao ler registro!\n");
        return;
    }

    printf("\n=== PRODUTO %d ===\n", nr);
    if (p.status == '*') printf("STATUS: EXCLUÍDO\n");

    printf("Nome....: %s\n", p.nome);
    printf("Código..: %d\n", p.codigo);
    printf("Preço...: R$ %.2f\n", p.preco);
}

/* ---------------------------------------------------------------------------
   excluir — exclusão lógica
--------------------------------------------------------------------------- */
void excluir(FILE *arq) {
    int nr;
    char confirma;
    Produto p;

    printf("\nInforme o código do registro para excluir: ");
    if (scanf("%d", &nr) != 1) {
        printf("Entrada inválida!\n");
        limpaBuffer();
        return;
    }
    limpaBuffer();

    if (nr <= 0 || nr > tamanho(arq)) {
        printf("Código inválido!\n");
        return;
    }

    fseek(arq, (long)(nr - 1) * sizeof(Produto), SEEK_SET);
    fread(&p, sizeof(Produto), 1, arq);

    if (p.status == '*') {
        printf("Registro já estava excluído.\n");
        return;
    }

    printf("Confirmar exclusão (s/n)? ");
    scanf("%c", &confirma);
    limpaBuffer();

    if (toupper(confirma) == 'S') {
        p.status = '*';
        fseek(arq, (long)(nr - 1) * sizeof(Produto), SEEK_SET);
        fwrite(&p, sizeof(Produto), 1, arq);

        printf("Registro excluído com sucesso!\n");
    } else {
        printf("Exclusão cancelada.\n");
    }
}

/* ---------------------------------------------------------------------------
   gerarRelatorioTxt — gera um arquivo texto com todos os produtos
--------------------------------------------------------------------------- */
void gerarRelatorioTxt(FILE *arq) {
    FILE *txt;
    char nomeArquivo[80];
    Produto p;
    int total;
    int i;

    printf("\nNome do arquivo (sem extensão): ");
    lerString(nomeArquivo, sizeof(nomeArquivo));
    strcat(nomeArquivo, ".txt");

    txt = fopen(nomeArquivo, "w");
    if (!txt) {
        printf("Erro ao criar arquivo de relatório.\n");
        return;
    }

    fprintf(txt, "RELATÓRIO DE PRODUTOS\n");
    fprintf(txt, "--------------------------------------------------------------\n");
    fprintf(txt, "COD | %-20s | PREÇO      | STATUS\n", "NOME");
    fprintf(txt, "--------------------------------------------------------------\n");

    total = tamanho(arq);

    for (i = 0; i < total; i++) {
        fseek(arq, i * sizeof(Produto), SEEK_SET);
        fread(&p, sizeof(Produto), 1, arq);

        fprintf(txt, "%03d | %-20s | R$ %-8.2f | %s\n",
                i + 1,
                p.nome,
                p.preco,
                (p.status == '*') ? "EXCLUIDO" : "ATIVO");
    }

    fclose(txt);
    printf("\nArquivo '%s' gerado com sucesso!\n", nomeArquivo);
}

/* ---------------------------------------------------------------------------
   configurarLocale — tenta ativar UTF-8
--------------------------------------------------------------------------- */
void configurarLocale(void) {
#if defined(_WIN32)
    system("chcp 65001 > nul");
#endif
    setlocale(LC_ALL, "pt_BR.UTF-8");
}

/* ---------------------------------------------------------------------------
   main — menu principal
--------------------------------------------------------------------------- */
int main(void) {

    configurarLocale();

    FILE *arq = fopen("estoque.dat", "r+b");
    if (!arq) {
        arq = fopen("estoque.dat", "w+b");
        if (!arq) {
            printf("Erro ao abrir/criar arquivo!\n");
            return 1;
        }
    }

    {
        int op;

        do {
            printf("\n=========== SISTEMA DE ESTOQUE ===========\n");
            printf("1. Cadastrar produto\n");
            printf("2. Consultar produto\n");
            printf("3. Excluir produto\n");
            printf("4. Gerar relatório (.txt)\n");
            printf("5. Sair\n");
            printf("-------------------------------------------\n");
            printf("Total de registros: %d\n", tamanho(arq));
            printf("Opção: ");

            if (scanf("%d", &op) != 1) {
                printf("Digite um número válido.\n");
                limpaBuffer();
                continue;
            }
            limpaBuffer();

            switch (op) {
                case 1: cadastrar(arq); break;
                case 2: consultar(arq); break;
                case 3: excluir(arq); break;
                case 4: gerarRelatorioTxt(arq); break;
                case 5: printf("Encerrando...\n"); break;
                default: printf("Opção inválida!\n");
            }

        } while (op != 5);
    }

    fclose(arq);
    return 0;
}

