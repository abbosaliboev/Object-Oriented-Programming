
# OS_WEB_PROJECT
Open Source Web Software Project

Albatta, Abbos! Quyida senga bu C dasturining **to‘liq va bosqichma-bosqich tushuntirishi**ni beraman. Bu kod asosan **“X” va “Y” agentlar o‘rtasida Connect Four (4ta qator qilish o‘yini)** o‘ynashini avtomatik boshqaradi.

---

## 🔹 1. KODNING BOSH QISMI — KUTUBXONALAR VA GLOBAL O‘ZGARUVCHILAR

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <signal.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <ctype.h>
```

Bu yerda kerakli kutubxonalar chaqirilgan:
- `stdio.h` – print va input funksiyalari uchun.
- `stdlib.h` – umumiy funksiyalar (masalan, `exit()`).
- `unistd.h` – `pipe()`, `fork()`, `dup2()` va `execl()` kabi tizim chaqiruvlari uchun.
- `string.h`, `ctype.h` – stringlar bilan ishlash.
- `signal.h` – signallarni ushlash uchun (masalan, `SIGINT`, `SIGALRM`).
- `sys/wait.h` – `wait()` funksiyasi uchun.
- `fcntl.h` – fayl descriptorlar uchun.

```c
pid_t pidX = -1;
pid_t pidY = -1;
int board[6][7]; // 6 rows and 7 columns game board
int turn = 1;    // 1 for X's turn, 2 for Y's turn
```

- `pidX` va `pidY` – X va Y agentlarining jarayon ID'lari
- `board` – 6x7 o‘lchamdagi Connect Four o‘yin taxtasi
- `turn` – hozirgi yurish kimga tegishli: 1 (X) yoki 2 (Y)

---

## 🔹 2. SIGNAL HANDLER FUNKSIYALARI

### `handle_sigint()` – Ctrl+C bosilganda bajariladi
```c
void handle_sigint(int sig) {
    printf("\n Ctrl+C pressed. Exiting cleanly...\n");
    if (pidX > 0) kill(pidX, SIGKILL);
    if (pidY > 0) kill(pidY, SIGKILL);
    exit(1);
}
```

Agar foydalanuvchi Ctrl+C bossa, X va Y agentlar to‘xtatiladi.

---

### `handle_sigchld()` – agent tugaganda chaqiriladi
```c
void handle_sigchld(int sig) {
    wait(NULL);
}
```

Agent (child process) tugagach, `wait()` chaqiriladi, bu zombiy processni oldini oladi.

---

### `handle_sigalrm()` – timeout holati
```c
void handle_sigalrm(int sig) {
    printf("\n Agent did not respond in time. Opponent wins.\n");
    if (turn == 1 && pidX > 0) kill(pidX, SIGKILL);
    if (turn == 2 && pidY > 0) kill(pidY, SIGKILL);
    exit(0);
}
```

Agar agent 3 soniyada javob bermasa – o‘yin tugaydi va raqib yutadi.

---

### `handle_sigterm()` – boshqa signal orqali to‘xtatilganda
```c
void handle_sigterm(int sig) {
    printf("\n[!] SIGTERM received. Program terminated.\n");
    exit(0);
}
```

---

## 🔹 3. YORDAMCHI FUNKSIYALAR

### `print_board()` – o‘yin taxtasini chiqaradi
```c
void print_board() {
    printf("\nGame Board:\n");
    for (int i = 0; i < 6; ++i) {
        for (int j = 0; j < 7; ++j) {
            printf("%d ", board[i][j]);
        }
        printf("\n");
    }
}
```

---

### `col_index()` – 'A'-'G' harflarini ustun raqamiga aylantiradi
```c
int col_index(char c) {
    return toupper(c) - 'A';
}
```

Masalan, `'A' → 0`, `'B' → 1`, ... `'G' → 6`.

---

### `place_stone()` – toshni pastdan yuqoriga joylaydi
```c
int place_stone(int col, int player) {
    if (col < 0 || col >= 7) return 0; // noto‘g‘ri ustun
    for (int i = 5; i >= 0; --i) {
        if (board[i][col] == 0) {
            board[i][col] = player;
            return 1;
        }
    }
    return 0; // ustun to‘la
}
```

---

### `check_winner()` – yutgan holatni tekshiradi (4ta ketma-ket)
```c
int check_winner(int player) {
    // gorizontal, vertikal, diagonal (ikki yo'nalishda)
    ...
}
```

Agar 4ta bir xil tosh ketma-ket joylashgan bo‘lsa – g‘alaba.

---

### `is_draw()` – taxta to‘la bo‘lsa, durang
```c
int is_draw() {
    for (int j = 0; j < 7; ++j)
        if (board[0][j] == 0)
            return 0;
    return 1;
}
```

---

## 🔹 4. `main()` — ASOSIY O‘YIN LOGIKASI

### a) Signal handler’larni ro‘yxatga olish
```c
signal(SIGINT, handle_sigint);
signal(SIGCHLD, handle_sigchld);
signal(SIGALRM, handle_sigalrm);
signal(SIGTERM, handle_sigterm);
```

---

### b) Buyruq qatorini tekshirish
```c
if (argc != 5 || strcmp(argv[1], "-X") != 0 || strcmp(argv[3], "-Y") != 0) {
    fprintf(stderr, "Usage: ./gamatch -X <agentX> -Y <agentY>\n");
    return 1;
}
```

Masalan:
```
./gamatch -X ./agentX -Y ./agentY
```

---

### c) O‘yin bosqichi: har bir yurish uchun
```c
while (1) {
    // pipe yaratish
    pipe(pipe_in); pipe(pipe_out);

    // agent tanlash (X yoki Y)
    char* agent = (turn == 1) ? agentX : agentY;

    pid_t pid = fork(); // yangi process yaratish

    if (pid == 0) {
        // Bola process: agentni ishga tushurish
        dup2(pipe_in[0], STDIN_FILENO);
        dup2(pipe_out[1], STDOUT_FILENO);
        close(pipe_in[1]); close(pipe_out[0]);
        execl(agent, agent, NULL);
    } else {
        if (turn == 1) pidX = pid;
        else pidY = pid;
    }

    // Ota process: kerakli joylarni yopish
    close(pipe_in[0]);
    close(pipe_out[1]);

    // Agentga input berish
    char input[512];
    snprintf(input, sizeof(input), "%d\n", turn);
    for (int i = 0; i < 6; ++i) {
        for (int j = 0; j < 7; ++j)
            snprintf(input + strlen(input), sizeof(input) - strlen(input),
                     "%d%c", board[i][j], j < 6 ? ' ' : '\n');
    }
    write(pipe_in[1], input, strlen(input));

    // Agentdan javob kutish (3 soniya)
    alarm(3);
    char buf[100];
    int n = read(pipe_out[0], buf, sizeof(buf) - 1);
    alarm(0);

    buf[n] = '\0';
    printf("%s agent move: %s", turn == 1 ? "X" : "Y", buf);

    // Javobni tahlil qilish va tosh qo‘yish
    int col = col_index(buf[0]);
    if (!place_stone(col, turn)) {
        printf("\n[!] Invalid column. You lose.\n");
        exit(0);
    }

    print_board();
    wait(NULL); // bola processni kutish

    // Yutgan yoki durang bo‘lsa – o‘yin tugaydi
    if (check_winner(turn)) {
        printf("%s wins the game!\n", turn == 1 ? "X" : "Y");
        break;
    }

    if (is_draw()) {
        printf("Game is a draw.\n");
        break;
    }

    sleep(1);
    turn = (turn == 1) ? 2 : 1; // yurishni almashtirish
}
```

---

## 🟢 XULOSA

Bu dastur `./gamatch -X ./agentX -Y ./agentY` formatida ishga tushiriladi. Har bir agent – alohida process sifatida ishlaydi va ular o‘yin holatini o‘qib, qaysi ustunga yurishni stdout orqali qaytaradi. Har yurish:
1. Agent process chaqiriladi.
2. O‘yin holati `stdin` orqali agentga uzatiladi.
3. Agent javob beradi: `A`, `B`, ... `G`
4. Toshiq joylanadi.
5. G‘alaba yoki durang bo‘lsa – o‘yin tugaydi.

---

Agar hohlasang, men senga `agentX.c` yoki `agentY.c` soddalashtirilgan variantini ham yozib beraman — sinab ko‘rish uchun. Yoki boshqa qismni chuqurroq tushuntirishim mumkin. Qanday davom etamiz? 😎
