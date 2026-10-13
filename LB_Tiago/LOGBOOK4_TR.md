# LOGBOOK4 — Environment Variables and Set-UID Programs

## Comandos e Conceitos Introduzidos

| Comando / Função | Uso |
|---|---|
| `printenv` / `env` | Imprimir variáveis de ambiente |
| `export` / `unset` | Definir e apagar variáveis de ambiente (comandos internos da Bash) |
| `fork()` | Cria um novo processo duplicando o processo atual |
| `execve(path, argv[], envp[])` | Substitui o programa em execução no processo atual pelo referido em `path` |
| `system(comando)` | Executa um comando através de uma shell — equivalente a `/bin/sh -c comando` |
| `chown` | Altera o dono (e/ou grupo) de um ficheiro ou diretório |
| `chmod` | Altera permissões de acesso e bits especiais (como o Set-UID) de um ficheiro |
| `setuid(uid)` | Define o UID efetivo do processo; se o processo tiver UID efetivo 0, também atualiza o UID real e o saved-UID |

---

## Questão 1 — Tasks 1 a 6

### Task 1 — Manipulação de Variáveis de Ambiente

Usámos `printenv`/`env` para listar variáveis de ambiente, e os comandos internos da Bash `export`/`unset` para as definir e remover, respetivamente.

### Task 2 — Propagação de Variáveis entre Processos (fork)

Compilámos e corremos `myprintenv.c`, que usa `fork()` para criar um processo filho. Numa primeira execução, o `printenv()` foi chamado apenas no filho; numa segunda, apenas no pai. Guardámos os dois outputs em ficheiros e comparámo-los com `diff`.

![Task 1-2](../task1-2.png)

**Conclusão:** o `diff` não mostrou diferenças entre os dois ficheiros — o processo filho herda uma **cópia completa** das variáveis de ambiente do pai no momento do `fork()`. Qualquer alteração posterior num dos processos não é propagada ao outro, já que cada um passa a ter o seu próprio espaço de memória independente.

### Task 3 — `execve()` e Variáveis de Ambiente

Compilámos e corremos `myenv.c`, que chama `execve("/usr/bin/env", argv, envp)` com dois valores diferentes no terceiro argumento:

- **Com `NULL`:** o `/usr/bin/env` não imprimiu nada — não recebeu nenhuma variável de ambiente.
- **Com `environ`:** o `/usr/bin/env` imprimiu todas as variáveis de ambiente do processo chamador.

![Task 3](../task3.png)

**Conclusão:** ao contrário do `fork()`, a chamada de sistema `execve()` **não herda automaticamente** as variáveis de ambiente do processo que a invoca — elas têm de ser passadas **explicitamente** através do terceiro parâmetro (`envp`).

### Task 4 — `system()` e Variáveis de Ambiente

Corremos um programa que invoca `system("/usr/bin/env")`.

![Task 4](../task4.png)

**Conclusão:** `system()` executa internamente `/bin/sh -c comando`, e essa invocação da shell usa `execve()` passando-lhe o `environ` do processo chamador. Por isso, ao contrário do `execve()` "nu" (Task 3), o `system()` propaga as variáveis de ambiente **automaticamente**, sem ser necessário indicá-las explicitamente — a shell lançada herda o ambiente do processo pai.

### Task 5 — Variáveis de Ambiente em Programas Set-UID

Escrevemos um programa que imprime todas as variáveis de `environ`, tornámo-lo Set-UID root (`sudo chown root foo && sudo chmod 4755 foo`) e definimos três variáveis na shell antes de o correr: `PATH`, `LD_LIBRARY_PATH` e uma variável à nossa escolha.

![Task 5](../task5.png)

**Conclusão:** as variáveis de ambiente do utilizador são, em geral, herdadas pelo processo Set-UID (incluindo `PATH` e a variável definida por nós). No entanto, o sistema **remove seletivamente** variáveis consideradas perigosas — como `LD_LIBRARY_PATH` — precisamente para impedir que um utilizador normal influencie as bibliotecas carregadas por um programa que vai correr com privilégios elevados.

### Task 6 — Exploração da Variável `PATH` num Programa Set-UID

**Objetivo:** demonstrar que um programa Set-UID que invoca `system("ls")` com um caminho relativo pode ser enganado para executar código arbitrário com privilégios de root, manipulando apenas a variável `PATH`.

**Passo 1 — Preparar o shell "atacante" (remover a contramedida do dash):**

Por omissão, `/bin/sh` aponta para `/bin/dash`, que deteta quando está a correr num processo Set-UID e descarta os privilégios de imediato, neutralizando o ataque. Para o demonstrar sem essa proteção, seguimos a indicação do guião:

```bash
sudo ln -sf /bin/zsh /bin/sh
```

**Passo 2 — Compilar o programa vulnerável e torná-lo Set-UID root:**

```c
/* vulnerable.c */
#include <stdio.h>
#include <stdlib.h>
int main()
{
    system("ls");
    return 0;
}
```

```bash
gcc vulnerable.c -o vulnerable
sudo chown root vulnerable
sudo chmod 4755 vulnerable
```

**Passo 3 — Criar um `ls` malicioso num diretório próprio:**

```bash
nano /home/seed/bin/ls
```
```bash
#!/bin/sh
whoami
```
```bash
chmod +x /home/seed/bin/ls
```

**Passo 4 — Colocar esse diretório no início do `PATH`:**

```bash
export PATH=/home/seed/bin/:$PATH
```

**Passo 5 — Correr o programa Set-UID:**

```bash
./vulnerable
```

![Task 6](../task6.png)

**Resultado:** em vez de executar `/bin/ls`, o `vulnerable` executou o nosso `ls` malicioso, e o `whoami` dentro dele imprimiu **`root`** — o código malicioso correu com privilégios de administrador.

**Porque é que estes passos são suficientes:**
1. O programa é Set-UID root, logo o seu UID efetivo é 0 durante toda a execução.
2. `system("ls")` invoca `/bin/sh -c "ls"`. Como alterámos `/bin/sh` para `/bin/zsh` (um shell sem a contramedida de dropar privilégios em Set-UID), essa shell continua a correr com UID efetivo 0.
3. O comando `ls`, por ser passado com **caminho relativo**, é procurado pela shell nos diretórios listados em `PATH`, pela ordem em que aparecem. Como colocámos `/home/seed/bin` no **início** do `PATH`, o nosso `ls` é encontrado e executado antes do `/bin/ls` real.
4. O nosso script corre então com o UID efetivo herdado do processo (root), daí o `whoami` devolver `root`.

**Nota importante:** no fim do exercício, repusemos a ligação original para não comprometer o estado do sistema:

```bash
sudo ln -sf /bin/dash /bin/sh
```

---

## Questão 2 — Task 8 (Step 1): Exploração do `catall.c` via `system()`

**Objetivo:** mostrar que, sem alterar o código C, é possível abusar do executável `catall` (Set-UID root) para apagar um ficheiro que o utilizador `seed` não tem permissão para ler nem apagar.

> **Pré-requisito:** tal como na Task 6, `/bin/sh` tem de continuar a apontar para `/bin/zsh` (`sudo ln -sf /bin/zsh /bin/sh`). Caso contrário, o `dash` deteta a execução Set-UID e larga os privilégios, fazendo com que o comando injetado corra como `seed` (sem privilégios de root) em vez de root.

**Preparação — criar o ficheiro-alvo:**

```bash
sudo touch /home/seed/Desktop/text
sudo chown root /home/seed/Desktop/text
sudo chmod 700 /home/seed/Desktop/text
```

Confirmar que `seed` não consegue acedê-lo diretamente:

```bash
cat /home/seed/Desktop/text   # Permission denied
whoami                         # seed
```

**Compilar e preparar o `catall` como Set-UID root:**

```c
/* catall.c (excerto relevante) */
v[0] = "/bin/cat"; v[1] = argv[1]; v[2] = NULL;
command = malloc(strlen(v[0]) + strlen(v[1]) + 2);
sprintf(command, "%s %s", v[0], v[1]);
system(command);   // única linha ativa nesta fase
```

```bash
gcc catall.c -o catall
sudo chown root catall
sudo chmod 4755 catall
```

**Ataque — injetar um comando através do argumento:**

```bash
./catall "/home/seed/Desktop/text; rm /home/seed/Desktop/text"
```

![Task 8.1](../questao2-1.png)

**Verificação:**

```bash
ls /home/seed/Desktop/text   # No such file or directory
```

O ficheiro, pertencente a root e sem permissões para `seed`, foi apagado por um utilizador sem privilégios de administrador.

**Mecanismo de ataque e justificação técnica:**

O `catall.c` constrói a string `"/bin/cat <argumento>"` e passa-a a `system()`. Internamente, `system()` executa `/bin/sh -c "<string>"`, e a shell **interpreta metacaracteres** como `;`, `|` e `&&`. Ao fornecer um argumento como `"ficheiro; rm ficheiro"`, estamos na prática a construir dois comandos separados por `;`, ambos executados sequencialmente pela mesma shell — o `cat` original e o `rm` injetado por nós.

Como `catall` é Set-UID root, a shell invocada por `system()` corre com UID efetivo 0, e por isso **ambos** os comandos — incluindo o `rm` injetado — são executados com privilégios de root, contornando as permissões do ficheiro que, de outra forma, impediriam `seed` de o apagar.

O problema de fundo não está nos dados em si, mas no facto de `system()` delegar a interpretação do comando a uma shell completa, em vez de executar diretamente o binário pretendido. Isto é o que torna `system()` perigoso em programas Set-UID sempre que o comando incorpora dados fornecidos pelo utilizador sem sanitização.

**Mitigação — substituir `system()` por `execve()`:**

Alterando o programa para invocar diretamente o binário com `execve()` em vez de `system()`, a injeção deixa de funcionar:

```c
char *v[3] = { "/bin/cat", argv[1], NULL };
execve(v[0], v, NULL);   // em vez de system(command)
```

Agora o argumento `"/home/seed/Desktop/text; rm /home/seed/Desktop/text"` é entregue ao `/bin/cat` como **um único nome de ficheiro** (incluindo o `;` e o `rm`), e não como uma linha de comando interpretada por uma shell. Não havendo shell a interpretar metacaracteres, o `rm` nunca chega a ser executado — o `cat` limita-se a dar erro por não existir um ficheiro com esse nome.

![Task 8.2](../questao2-2.png)

**Conclusão:** a diferença essencial é que `execve()` executa o binário indicado passando os argumentos como dados puros, enquanto `system()` passa pela shell, que trata os dados como código. Evitar `system()` (ou sanitizar rigorosamente a entrada) é a contramedida central contra injeção de comandos em programas Set-UID.

---

## Questão 3 — Task 9: Capability Leaking

### 1. Preparação do ambiente

Criação do ficheiro-alvo, pertencente a root, com permissões que **não** permitem escrita a `seed`:

```bash
sudo touch /etc/zzz
sudo chown root:root /etc/zzz
sudo chmod 0644 /etc/zzz
```

Compilação do `cap_leak.c` e configuração como Set-UID root:

```bash
gcc cap_leak.c -o cap_leak
sudo chown root:root cap_leak
sudo chmod 4755 cap_leak
```

### 2. Evidência da execução

```bash
./cap_leak
```

O programa imprime o descritor obtido (ex.: `fd is 3`) e lança uma shell interativa — já sem privilégios de root (o programa já chamou `setuid(getuid())` antes de executar a shell). A partir dessa shell, sem ser root:

```bash
whoami                               # seed
echo "Escrita no ficheiro zzz" >&3
exit
```

Verificação final, fora do programa:

```bash
cat /etc/zzz
```

![Task 9](../questao3.png)

**Resultado:** o texto escrito dentro da shell não privilegiada aparece em `/etc/zzz`, apesar de `seed` não ter permissão de escrita nesse ficheiro (`0644`, dono `root`).

### 3. Explicação técnica

**O que é capability leaking:** é a situação em que um processo privilegiado adquire recursos (como file descriptors, sockets, ou capacidades do kernel) enquanto ainda tem privilégios elevados, e depois reduz o seu nível de privilégio (por exemplo via `setuid()`) sem libertar ou revogar esses recursos. O processo deixa de ser "root" no sentido do UID, mas continua de posse de acessos que só deveriam existir enquanto era root.

**Mecanismo concreto no `cap_leak.c`:**

```c
fd = open("/etc/zzz", O_RDWR | O_APPEND);   // (1) ainda com euid = 0
setuid(getuid());                            // (2) descarta privilégios
v[0] = "/bin/sh"; v[1] = 0;
execve(v[0], v, 0);                          // (3) lança shell sem privilégios
```

1. O `open()` é chamado **antes** do `setuid()`, ou seja, enquanto o UID efetivo do processo ainda é 0 (root). O kernel verifica as permissões do ficheiro nesse momento e, como o processo é root, concede um descritor com acesso de leitura e escrita (`O_RDWR`), independentemente das permissões `0644` que, em condições normais, impediriam `seed` de escrever.
2. Esse descritor (`fd`) é uma referência interna do kernel a uma *open file description* — as permissões associadas a essa referência **ficam fixadas no momento da abertura** e não são reavaliadas em cada leitura/escrita subsequente.
3. `setuid(getuid())` altera apenas o UID real/efetivo/saved do processo; **não fecha nenhum descritor aberto**.
4. `execve()` substitui a imagem do processo por `/bin/sh`, mas os file descriptors abertos **são preservados através do `execve()`** por omissão (a menos que tenham a flag `FD_CLOEXEC` explicitamente definida, o que não acontece aqui). Por isso, a shell resultante herda o `fd` 3, aberto com permissão de escrita sobre `/etc/zzz`.

**Recurso concretamente explorado:** o file descriptor (fd 3) para `/etc/zzz`, aberto com `O_RDWR | O_APPEND` enquanto o processo ainda tinha UID efetivo 0 — é esse descritor, e não qualquer permissão do UID atual (`seed`), que permite a escrita bem-sucedida via `echo ... >&3`.

**Porque os passos são suficientes:** a vulnerabilidade só existe porque (a) o `open()` ocorre antes da queda de privilégio, (b) o descritor nunca é fechado, e (c) o `execve()` não marca esse descritor para fechar automaticamente. Bastaria qualquer uma destas três condições falhar (fechar o fd antes do `setuid`, ou marcá-lo com `FD_CLOEXEC`, ou abrir o ficheiro só depois do `setuid`) para que o ataque deixasse de funcionar.

### 4. Medidas de mitigação

- **Abrir ficheiros privilegiados depois do `setuid()`**, sempre que a lógica do programa o permita, para que a abertura já seja feita com os privilégios finais e reduzidos.
- **Fechar explicitamente** (`close(fd)`) qualquer descritor privilegiado antes de descartar os privilégios ou antes de invocar `execve()`/lançar uma shell.
- Marcar os descritores com a flag **`FD_CLOEXEC`** (via `fcntl(fd, F_SETFD, FD_CLOEXEC)`), para que sejam automaticamente fechados em qualquer `exec*()` subsequente.
- Em sistemas com suporte a **POSIX capabilities**, limpar explicitamente todas as capabilities com `cap_set_proc()` / `libcap`, em vez de confiar apenas na troca de UID.
- Usar `prctl(PR_SET_KEEPCAPS, 0)` com cuidado, para garantir que capabilities não sobrevivem indevidamente a uma mudança de UID.
- **Evitar Set-UID sempre que possível**, preferindo alternativas como `sudo` com regras específicas, capabilities do Linux, `polkit`, ou um daemon dedicado que execute a operação privilegiada de forma controlada.
- Aplicar o **princípio do menor privilégio**: descartar os privilégios root assim que a operação privilegiada específica estiver concluída, e não apenas "algures" antes de entregar o controlo ao utilizador.
- Auditar sistematicamente todos os recursos (ficheiros, sockets, memória partilhada) adquiridos antes de qualquer queda de privilégio num programa Set-UID.

### 5. Evidência final e conclusões

O `cat /etc/zzz` executado fora do programa, já sem qualquer processo privilegiado em curso, confirma que o conteúdo escrito por `seed` (confirmado pelo `whoami` dentro da shell gerada pelo próprio exploit) persiste no ficheiro pertencente a root. Isto prova, de forma independente da sessão que correu o ataque, que um utilizador sem privilégios de escrita em `/etc/zzz` conseguiu, através do `cap_leak`, escrever nesse ficheiro.

**Conclusão geral:** reduzir o UID de um processo com `setuid()` não é, por si só, suficiente para garantir que o processo deixou de ser privilegiado. Recursos adquiridos antes da queda de privilégio — neste caso, um file descriptor aberto para escrita — continuam acessíveis depois da queda, constituindo um vetor de escalada de privilégios indireta. Um programa Set-UID bem construído tem de tratar explicitamente o ciclo de vida de **todos** os recursos privilegiados que adquire, e não apenas o seu próprio UID.

---

## Nota complementar — Task 7 (não solicitada diretamente pelas questões do relatório)

Realizámos também a Task 7, sobre o comportamento da variável `LD_PRELOAD`:

- **Programa normal, utilizador normal:** `LD_PRELOAD` funciona — a função `sleep()` definida em `libmylib.so.1.0.1` substitui a da libc, imprimindo `"I am not sleeping!"`.
- **Programa Set-UID root, utilizador normal:** `LD_PRELOAD` é **ignorado** pelo dynamic linker — o programa dorme normalmente.
- **Programa Set-UID root, corrido pelo próprio root:** `LD_PRELOAD` volta a funcionar, porque real UID = effective UID = owner, não havendo escalada de privilégio.
- **Programa Set-UID de outro utilizador, corrido por `seed`:** `LD_PRELOAD` é novamente ignorado.

**Conclusão:** o dynamic linker só respeita `LD_PRELOAD` (e variáveis `LD_*` semelhantes) quando o UID real coincide com o UID efetivo do processo. Sempre que há uma diferença entre estes dois UIDs — sinal de que o processo está a correr com privilégios herdados de outro utilizador — essas variáveis são ignoradas, precisamente para impedir a injeção de bibliotecas maliciosas em programas privilegiados.
