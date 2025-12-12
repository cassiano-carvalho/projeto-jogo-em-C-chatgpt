# projeto-jogo-em-C-chatgpt
Projeto da DIO.me


Ótimo — aqui está um jogo de damas (checkers) em C puro (C99), para jogar no terminal entre dois jogadores humanos. O programa implementa:

Tabuleiro 8x8 com casas escuras jogáveis.

Movimentação de peças normais e coroação (king).

Capturas obrigatórias e múltiplas capturas (sequência).

Validação de movimentos e mensagens de erro para entrada inválida.

Interface simples por texto (entradas com coordenadas numéricas 1..8).

Compile com gcc -std=c99 -O2 -o damas damas.c e execute ./damas.

Código (arquivo: damas.c)
// damas.c
// Jogo de damas (checkers) em C puro - dois jogadores humanos no terminal.
// Compile: gcc -std=c99 -O2 -o damas damas.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define SIZE 8

// Representação das peças:
// '.' casa vazia (ou '_' para casas claras não jogáveis quando imprimindo)
// 'r' peça do jogador Red (jogador 1)
// 'R' dama (king) do Red
// 'b' peça do jogador Black (jogador 2)
// 'B' dama do Black

char board[SIZE][SIZE];

void init_board() {
    for (int r = 0; r < SIZE; ++r) {
        for (int c = 0; c < SIZE; ++c) {
            board[r][c] = '.';
        }
    }
    // Posições iniciais tradicionais: peças em (r+c)%2==1 nas três primeiras linhas de cada lado
    for (int r = 0; r < 3; ++r) {
        for (int c = 0; c < SIZE; ++c) {
            if ((r + c) % 2 == 1) board[r][c] = 'b'; // Black no topo
        }
    }
    for (int r = SIZE-3; r < SIZE; ++r) {
        for (int c = 0; c < SIZE; ++c) {
            if ((r + c) % 2 == 1) board[r][c] = 'r'; // Red em baixo
        }
    }
}

void print_board() {
    printf("    1 2 3 4 5 6 7 8\n");
    printf("   -----------------\n");
    for (int r = 0; r < SIZE; ++r) {
        printf("%d | ", r+1);
        for (int c = 0; c < SIZE; ++c) {
            if ((r + c) % 2 == 0) {
                // casa clara (não jogavel)
                putchar('_');
            } else {
                char ch = board[r][c];
                if (ch == '.') ch = ' ';
                putchar(ch);
            }
            if (c < SIZE-1) putchar(' ');
        }
        printf(" |\n");
    }
    printf("   -----------------\n");
}

int in_bounds(int r, int c) {
    return r >= 0 && r < SIZE && c >= 0 && c < SIZE;
}

int is_enemy(char piece, char other) {
    if (piece == 'r' || piece == 'R') return (other == 'b' || other == 'B');
    if (piece == 'b' || piece == 'B') return (other == 'r' || other == 'R');
    return 0;
}

int is_king(char p) {
    return p == 'R' || p == 'B';
}

// Verifica se existe qualquer captura possível para o jogador 'side' ('r' ou 'b')
int capture_exists_for_side(char side) {
    for (int r = 0; r < SIZE; ++r) {
        for (int c = 0; c < SIZE; ++c) {
            char p = board[r][c];
            if (p == '.' || p == ' ') continue;
            if (tolower(p) != side) continue;
            // para cada direção diagonal, verifique salto
            int drs[4] = {-1,-1,1,1};
            int dcs[4] = {-1,1,-1,1};
            for (int d = 0; d < 4; ++d) {
                int nr = r + drs[d];
                int nc = c + dcs[d];
                int jr = r + 2*drs[d];
                int jc = c + 2*dcs[d];
                if (!in_bounds(jr,jc)) continue;
                char mid = board[nr][nc];
                if (mid == '.' || mid == ' ') continue;
                if (is_enemy(p, mid) && board[jr][jc] == '.') {
                    // para peças normais, checar direção permitida
                    if (!is_king(p)) {
                        if (tolower(p) == 'r' && drs[d] != -1) continue; // r se move para cima (dr = -1)
                        if (tolower(p) == 'b' && drs[d] != 1) continue;  // b se move para baixo (dr = 1)
                    }
                    return 1;
                }
            }
        }
    }
    return 0;
}

// Aplica um movimento simples (sem validar previamente) e retorna se uma captura foi feita.
// from/to são índices 0-based.
// Se houver captura, remove a peça capturada e retorna 1.
int apply_move(int fr, int fc, int tr, int tc) {
    char p = board[fr][fc];
    board[fr][fc] = '.';
    int dr = tr - fr;
    int dc = tc - fc;
    if (abs(dr) == 2 && abs(dc) == 2) {
        int mr = fr + dr/2;
        int mc = fc + dc/2;
        // captura
        board[mr][mc] = '.';
        board[tr][tc] = p;
        // promoção
        if (tolower(p) == 'r' && tr == 0) board[tr][tc] = 'R';
        if (tolower(p) == 'b' && tr == SIZE-1) board[tr][tc] = 'B';
        return 1;
    } else {
        board[tr][tc] = p;
        if (tolower(p) == 'r' && tr == 0) board[tr][tc] = 'R';
        if (tolower(p) == 'b' && tr == SIZE-1) board[tr][tc] = 'B';
        return 0;
    }
}

// Verifica se um movimento (fr,fc)->(tr,tc) é válido para jogador side ('r' ou 'b') considerando obrigatoriedade de captura caso exista.
// Se for captura válida, retorna 2. Se for movimento simples válido, retorna 1. Se inválido, retorna 0.
int validate_move(int fr, int fc, int tr, int tc, char side) {
    if (!in_bounds(fr,fc) || !in_bounds(tr,tc)) return 0;
    char p = board[fr][fc];
    if (p == '.' || p == ' ') return 0;
    if (tolower(p) != side) return 0;
    if (board[tr][tc] != '.') return 0;
    int dr = tr - fr;
    int dc = tc - fc;
    if (abs(dc) != abs(dr)) return 0;
    if (abs(dr) == 1) {
        // movimento simples
        if (capture_exists_for_side(side)) return 0; // captura obrigatória
        if (!is_king(p)) {
            if (tolower(p) == 'r' && dr != -1) return 0;
            if (tolower(p) == 'b' && dr != 1) return 0;
        }
        return 1;
    } else if (abs(dr) == 2) {
        // salto (captura)
        int mr = fr + dr/2;
        int mc = fc + dc/2;
        if (!in_bounds(mr,mc)) return 0;
        char mid = board[mr][mc];
        if (mid == '.' || mid == ' ') return 0;
        if (!is_enemy(p, mid)) return 0;
        if (!is_king(p)) {
            if (tolower(p) == 'r' && dr != -2) return 0;
            if (tolower(p) == 'b' && dr != 2) return 0;
        }
        return 2;
    }
    return 0;
}

int count_pieces(char side) {
    int cnt = 0;
    for (int r = 0; r < SIZE; ++r) for (int c = 0; c < SIZE; ++c)
        if (tolower(board[r][c]) == side) ++cnt;
    return cnt;
}

// Checa se a peça (r,c) depois de uma captura pode capturar novamente (para obrigar múltiplas capturas)
int piece_can_capture(int r, int c) {
    char p = board[r][c];
    if (p == '.' || p == ' ') return 0;
    int drs[4] = {-1,-1,1,1};
    int dcs[4] = {-1,1,-1,1};
    for (int d = 0; d < 4; ++d) {
        int nr = r + drs[d];
        int nc = c + dcs[d];
        int jr = r + 2*drs[d];
        int jc = c + 2*dcs[d];
        if (!in_bounds(jr,jc) || !in_bounds(nr,nc)) continue;
        char mid = board[nr][nc];
        if (mid == '.' || mid == ' ') continue;
        if (is_enemy(p, mid) && board[jr][jc] == '.') {
            if (!is_king(p)) {
                if (tolower(p) == 'r' && drs[d] != -1) continue;
                if (tolower(p) == 'b' && drs[d] != 1) continue;
            }
            return 1;
        }
    }
    return 0;
}

void flush_stdin() {
    int c;
    while ((c = getchar()) != '\n' && c != EOF) {}
}

int main() {
    init_board();
    char current = 'r'; // 'r' começa (jogador 1)
    int move_number = 1;
    printf("Jogo de Damas - Dois jogadores (terminal)\n");
    printf("Jogador 'r' (baixas) vs Jogador 'b' (altas). 'r' comeca.\n");
    printf("Entradas: linha coluna linha coluna (ex: 6 1 5 2) - coordenadas 1..8\n");
    printf("Casas claras nao sao jogaveis (sao mostradas como '_').\n");
    printf("Captura obrigatoria. Boa sorte!\n\n");

    while (1) {
        print_board();
        printf("Turno %d - Jogador '%c'\n", move_number, current);
        if (count_pieces(current) == 0) {
            printf("Jogador '%c' nao tem pecas. Jogador '%c' vence!\n", current, (current=='r'?'b':'r'));
            break;
        }
        if (!capture_exists_for_side(current)) {
            // ainda pode haver movimentos simples; se nenhum movimento possível -> perde
            int any = 0;
            for (int r = 0; r < SIZE && !any; ++r) for (int c = 0; c < SIZE && !any; ++c) {
                if (tolower(board[r][c]) != current) continue;
                int drs[4] = {-1,-1,1,1};
                int dcs[4] = {-1,1,-1,1};
                for (int d = 0; d < 4 && !any; ++d) {
                    int nr = r + drs[d];
                    int nc = c + dcs[d];
                    if (!in_bounds(nr,nc)) continue;
                    if (board[nr][nc] == '.') {
                        if (!is_king(board[r][c])) {
                            if (tolower(board[r][c])=='r' && drs[d] != -1) continue;
                            if (tolower(board[r][c])=='b' && drs[d] != 1) continue;
                        }
                        any = 1;
                    }
                }
            }
            if (!any) {
                printf("Jogador '%c' sem movimentos legais. Jogador '%c' vence!\n", current, (current=='r'?'b':'r'));
                break;
            }
        }

        printf("Digite seu movimento: ");
        int fr, fc, tr, tc;
        int res = scanf("%d %d %d %d", &fr, &fc, &tr, &tc);
        if (res != 4) {
            printf("Entrada invalida. Para sair, digite 'q' e enter. Para continuar, entre 4 numeros.\n");
            flush_stdin();
            char s[8];
            if (fgets(s, sizeof(s), stdin)) {
                if (s[0] == 'q' || s[0] == 'Q') {
                    printf("Jogo encerrado pelo usuario.\n");
                    break;
                }
            }
            continue;
        }
        // converter para 0-based
        fr--; fc--; tr--; tc--;
        if (!in_bounds(fr,fc) || !in_bounds(tr,tc)) {
            printf("Coordenadas fora do tabuleiro. Tente novamente.\n");
            continue;
        }
        int val = validate_move(fr,fc,tr,tc,current);
        if (val == 0) {
            printf("Movimento invalido. Tente novamente.\n");
            continue;
        }
        // aplicar movimento
        int captured = apply_move(fr,fc,tr,tc);
        // se foi captura, verificar possibilidade de multi-capture com a mesma peça
        if (captured) {
            int rcur = tr, ccur = tc;
            while (piece_can_capture(rcur, ccur)) {
                print_board();
                printf("Captura adicional possivel com a mesma peca (%d,%d). Deve continuar.\n", rcur+1, ccur+1);
                printf("Digite o proximo salto (linha coluna) destino final da peca: ");
                int nr, nc;
                int rr = scanf("%d %d", &nr, &nc);
                if (rr != 2) {
                    printf("Entrada invalida. Abortando cadeia de captura.\n");
                    flush_stdin();
                    break;
                }
                nr--; nc--;
                int v2 = validate_move(rcur, ccur, nr, nc, current);
                if (v2 != 2) {
                    printf("Esse salto nao e valido. Tente novamente.\n");
                    continue;
                }
                apply_move(rcur, ccur, nr, nc);
                rcur = nr; ccur = nc;
            }
        }
        // alterna jogador
        current = (current == 'r') ? 'b' : 'r';
        move_number++;
    }

    printf("Fim do jogo.\n");
    return 0;
}

Observações e melhorias possíveis

Atualmente o jogo é para dois humanos no mesmo terminal. Posso:

Adicionar IA (níveis simples, minimax, heurística).

Melhorar a interface (usar notação algébrica a3-b4, histórico de jogadas).

Salvar/carregar partidas.

Suportar variantes de regras (damas brasileiras, americanas, internacionais têm diferenças).

Se quiser, implemento IA (mais fácil: heurística e profundidade limitada) ou adapto o código para aceitar entrada no formato a3 b4.

Quer que eu:

Explique o código peça a peça?

Adicione IA (CPU) — nível fácil/médio?

Mude notação de entrada para letras (a-h) e número?

Diga só qual melhoria prefere e eu já implemento.
