# Encriptor – Operating Systems Project

## Overview

Encriptor is a command-line tool designed as an operating systems project that can both encrypt and decrypt words. The program reads an input file containing words, maps the file into shared memory, and spawns multiple processes to work on different subsets of the data concurrently. Each process applies a random permutation to each word using the Fisher–Yates shuffle algorithm for encryption, or reverses the permutation for decryption. The results are then written to an output file.

The tool supports two modes:
- **Encryption Mode:** Processes an input file containing plain words (one per line), encrypts each word by applying a random permutation, and writes the encrypted word along with its permutation to the output file.
- **Decryption Mode:** Processes an input file containing pairs of lines (an encrypted word followed by its permutation), decrypts the word by inverting the permutation, and writes the original word to the output file.

## Features

- **Parallel Processing:** Utilizes multiple processes (via `fork()`) to handle subsets of the input simultaneously.
- **Shared Memory:** Uses `mmap` to allocate a shared memory segment that holds the data structures accessed by all processes.
- **Random Permutations:** Implements the Fisher–Yates shuffle to generate random permutations for encrypting words.
- **Dual Mode Operation:** Supports both encryption and decryption based on command-line arguments.

## Requirements

- A POSIX-compliant operating system (e.g., Linux).
- A C compiler (e.g., `gcc`).
- Standard C libraries:
  - `stdio.h`
  - `stdlib.h`
  - `string.h`
  - `unistd.h`
  - `sys/wait.h`
  - `sys/mman.h`
  - `fcntl.h`
  - `time.h`

## Building the Project

1. **Clone or Download the Repository**  
   Ensure that you have the source file (e.g., `encriptor.c`) in your working directory.

2. **Compile the Code**  
   Use a C compiler to build the project. For example:
   ```bash
   gcc -o encriptor encriptor.c
   ```
## Usage

The program requires three command-line arguments:

- The mode of operation: either encrypt or decrypt.
- The input file path.
- The output file path.
  
### Encryption Mode
When run in encryption mode, the input file should contain one plain word per line. The program will output the encrypted word followed by a line containing the permutation indices used.

Command:

```bash
./encriptor encrypt input.txt output.txt
```

Example Input File (input.txt):

```bash
apple
banana
cherry
```
Example Output File (output.txt):

```bash
lpeap
3 4 2 0 1 
aaabnn
2 3 4 1 5 0 
rrcehy
4 5 3 0 1 2 
```

### Decryption Mode
In decryption mode, the input file should contain pairs of lines:

- The first line is the encrypted word.
- The second line contains the permutation indices used during encryption.

Command:

```bash
./encriptor decrypt encrypted_input.txt decrypted_output.txt
```

Example Input File (input.txt):

```bash
lpeap
3 4 2 0 1 
aaabnn
2 3 4 1 5 0 
rrcehy
4 5 3 0 1 2 
```
Example Output File (output.txt):

```bash
apple
banana
cherry
```
  
## Implementation

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <time.h>

#define MAX_WORDS       1000
#define MAX_WORD_LENGTH 256
#define NUM_PROCESSES   4     // e un numar aproximativ

typedef struct {
    char original[MAX_WORD_LENGTH];     // cuvantul initial 
    char result[MAX_WORD_LENGTH];       // cuvantul rezultat (criptat/decriptat)
    int  perm[MAX_WORD_LENGTH];         // permutarea
} WordData;

typedef struct {
    WordData words[MAX_WORDS];
    int word_count;    // nr de cuvinte de procesat
} SharedData;    // blocul de memorie partajata dintre procese



void generate_permutation(const char *word, int *permutation, char *permuted_word) {
    int len = strlen(word);
    for (int i = 0; i < len; i++) {
        permutation[i] = i;
    }
    // amestecare random (Fisher–Yates)
    for (int i = len - 1; i > 0; i--) {
        int j = rand() % (i + 1);  // alege un index random intre 0 si i
        // interschimbam permutation[i] cu permutation[j]
        int temp = permutation[i];
        permutation[i] = permutation[j];
        permutation[j] = temp;
    }
    // construim cuvantul criptat
    for (int i = 0; i < len; i++) {
        permuted_word[i] = word[ permutation[i] ];
    }
    permuted_word[len] = '\0';
}

// decriptare folosind cuvantul criptat si permutarea
void invert_permutation(const char *encrypted_word, const int *permutation, char *decrypted_word) {
    int len = strlen(encrypted_word);
    for (int i = 0; i < len; i++) {
        decrypted_word[ permutation[i] ] = encrypted_word[i];
    }
    decrypted_word[len] = '\0';
}

// functie apelata de fiecare proces fiu pentru a cripta/decripta subsetul [start, end) 
void process_chunk(SharedData *shm_data, int start, int end, int encrypt_mode) {
    for (int i = start; i < end; i++) {
        if (encrypt_mode) {
            // encrypt_mode = 1 => criptam
            generate_permutation(shm_data->words[i].original,
                                 shm_data->words[i].perm,
                                 shm_data->words[i].result);
        } else {
            // encrypt_mode = 0 => decriptam
            invert_permutation(shm_data->words[i].original,
                               shm_data->words[i].perm,
                               shm_data->words[i].result);
        }
    }
}

// citeste datele din fisier, in functie de mod:
// - encrypt_mode = 1 -> fisierul contine doar cuvinte, cate unul pe linie
// - encrypt_mode = 0 -> fisierul contine perechi (cuvant criptat + permutare)
int read_input_file(const char *filename, SharedData *shm_data, int encrypt_mode) {
    FILE *f = fopen(filename, "r");
    if (!f) {
        perror("Eroare la deschiderea fisierului de intrare");
        return 0;
    }
    shm_data->word_count = 0;  // initial nu avem cuvinte

    char line_buffer[1024];
    while (!feof(f) && shm_data->word_count < MAX_WORDS) {
		// citim o linie din fisier
        if (!fgets(line_buffer, sizeof(line_buffer), f)) break;
        
        // eliminam \n de la final (daca exista)
        char *p = strchr(line_buffer, '\n');
        if (p) *p = '\0';

        if (encrypt_mode) {
            // linie = cuvant simplu
            strcpy(shm_data->words[shm_data->word_count].original, line_buffer);
            shm_data->word_count++; // adunam 1 la nr de cuvinte
        } else {
            // decrypt_mode -> line_buffer e cuvantul criptat
            strcpy(shm_data->words[shm_data->word_count].original, line_buffer);

            // citim inca o linie cu permutarile
            if (!fgets(line_buffer, sizeof(line_buffer), f)) {
                fprintf(stderr, "Fisier incorect (lipsa linie permutare)!\n");
                break;
            }
            p = strchr(line_buffer, '\n');
            if (p) *p = '\0';

            // Spargem linia in token-uri si convertim fiecare token in int
            int idx = 0;
            char *tok = strtok(line_buffer, " \t");
            while (tok && idx < MAX_WORD_LENGTH) {
                shm_data->words[shm_data->word_count].perm[idx++] = atoi(tok);
                tok = strtok(NULL, " \t");
            }
            shm_data->word_count++; // adunam 1 la nr de cuvinte
        }
    }

    fclose(f);
    return 1;
}

// scrie rezultatele in fisier, in functie de mod:
// - encrypt_mode = 1 -> cuvant_criptat, permutare
// - encrypt_mode = 0 -> cuvant_decriptat
int write_output_file(const char *filename, SharedData *shm_data, int encrypt_mode) {
    FILE *out = fopen(filename, "w");
    if (!out) {
        perror("Eroare la deschiderea fisierului de iesire");
        return 0;
    }
    for (int i = 0; i < shm_data->word_count; i++) {
        if (encrypt_mode) {
            // 1) cuvantul criptat
            fprintf(out, "%s\n", shm_data->words[i].result);
            // 2) linia cu permutarea
            int len = strlen(shm_data->words[i].original);
            for (int j = 0; j < len; j++) {
                fprintf(out, "%d ", shm_data->words[i].perm[j]);
            }
            fprintf(out, "\n");
        } else {
            // doar cuvantul decriptat
            fprintf(out, "%s\n", shm_data->words[i].result);
        }
    }
    fclose(out);
    return 1;
}

int main(int argc, char *argv[]) {
    if (argc != 4) {
        fprintf(stderr, "Utilizare: %s <encrypt|decrypt> <input_file> <output_file>\n", argv[0]);
        return 1;
    }

    const char *mode_str    = argv[1];
    const char *input_file  = argv[2];
    const char *output_file = argv[3];

    // determinam daca este encrypt (1) sau decrypt (0)
    int encrypt_mode = (strcmp(mode_str, "encrypt") == 0) ? 1 : 0;

    srand(time(NULL)); // initializare seed random

    // alocam memorie partajata (MAP_SHARED) pentru a putea fi folosita de procese
    SharedData *shm_data = mmap(NULL,
                                sizeof(SharedData),
                                PROT_READ | PROT_WRITE,
                                MAP_SHARED | MAP_ANONYMOUS,
                                -1, 0);
    if (shm_data == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    // citim fisierul de intrare in structura partajata
    if (!read_input_file(input_file, shm_data, encrypt_mode)) {
        munmap(shm_data, sizeof(SharedData)); // eliberam memoria partajata
        return 1;
    }
    
    int total_words = shm_data->word_count;
    if (total_words == 0) {
        fprintf(stderr, "Nu exista cuvinte de procesat.\n");
        munmap(shm_data, sizeof(SharedData));
        return 1;
    }

    // impartim cuvintele in chunk-uri și facem fork
    int words_per_process = (total_words + NUM_PROCESSES - 1) / NUM_PROCESSES;

	// cream procesele fiu si repartizam chunk-uri de cuvinte
    for (int i = 0; i < NUM_PROCESSES; i++) {
        int start = i * words_per_process;
        
        // daca start e mai mare decat nr de cuvinte, nu mai avem ce procesa
        if (start >= total_words) break;
        int end   = start + words_per_process;
        if (end > total_words) end = total_words;

        pid_t pid = fork();
        if (pid < 0) {
            perror("fork");
            // in caz de eroare, asteptam child-urile deja create
            for (int k = 0; k < i; k++) wait(NULL);
            munmap(shm_data, sizeof(SharedData));
            return 1;
        }
        else if (pid == 0) {
            // copil - procesează subsetul [start, end)
            process_chunk(shm_data, start, end, encrypt_mode);
            munmap(shm_data, sizeof(SharedData)); // eliberm memoria
            exit(0);
        }
    }

    // asteptam procesele fiu sa se termine
    for (int i = 0; i < NUM_PROCESSES; i++) {
        wait(NULL);
    }

    // scriem rezultatele in fisierul de iesire
    if (!write_output_file(output_file, shm_data, encrypt_mode)) {
        munmap(shm_data, sizeof(SharedData));
        return 1;
    }

    // eliberam memoria partajata
    munmap(shm_data, sizeof(SharedData));

    printf("Operatie '%s' finalizata cu succes. Rezultat in: %s\n", 
           encrypt_mode ? "encrypt" : "decrypt",
           output_file);

    return 0;
}

```
