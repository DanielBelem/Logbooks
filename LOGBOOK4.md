# LOGBOOK4

## Questão 1

O lab de Variáveis de ambiente e SET-UID começa introduzindo comandos e funções, nomeadamente:

| Comandos e Funções | Uso |
| ------------- |:-------------:|
| printenv / env | Imprimir variáveis de ambiente |
| export / unset | Definir e apagar variáveis de ambiente, respetivamente |
| fork() | Cria um novo processo duplicando o processo atual |
| execve(path, argv[], envp[]) | Chamada de sistema utilizada para executar um programa referido por path, substituindo o programa executado no processo atual |
| system(comando) | Executa um comando através de uma shell, equivalente a invocar `/bin/sh -c comando` |
| chown | Alterar dono e/ou grupo associado a um arquivo ou diretório |
| chmod | Alterar permissões de acesso (escrita, leitura, execução) e bits especiais, como o SET-UID, associados a um arquivo ou diretório |

#### Task 1-2

Na Task 1 usamos os comandos printenv, export e unset para observar, definir e apagar variáveis de ambiente.

Na Task 2 observamos a diferença entre as variáveis de ambiente de um processo pai e um processo filho através da utilização do fork(). Ao comparar o output de printenv() dos dois, guardado em dois ficheiros e comparado com diff, percebemos que são herdadas todas as variáveis de ambiente de pai pra filho. Não havendo diferenças entre os ficheiros, como mostra a imagem, o filho recebe uma cópia das variáveis naquele momento, quaisquer alterações futuras não serão partilhadas.

![Task 1-2](/task1.png)

#### Task 3-4

Na Task 3 observamos a diferença entre executar um programa através de execve() com o argumento de variáveis de ambiente a NULL e com environ. Ao passar NULL, o programa é executado com o ambiente vazio; ao passar environ, recebe as variáveis de ambiente do processo que o chamou. O execve() não cria um novo processo, só substitui o programa executado no processo atual.

![Task 3](/task3.png)

Na Task 4 usamos system(), que executa o comando através de uma shell. As variáveis de ambiente do processo chamador são passadas à shell automaticamente, sem ser preciso indicá-las como argumento de system().

![Task 4.](/task4.png)

#### Task 5

Nesta task observa-se as variáveis de ambiente herdadas por um programa SET-UID. Ao definir root como dono do executável e ativar o SET-UID, percebemos que nem todas as variáveis de ambiente são herdadas. Por exemplo, a variável *LD_LIBRARY_PATH* é removida pelo processo, enquanto PATH e a variável definida por nós são mantidas.

O programa SET-UID impede a alteração da variável *LD_LIBRARY_PATH* para não influenciar as bibliotecas carregadas pelo programa com privilégios root.

![Task 5](/task5.png)

#### Task 6

Nesta task é pedido que se mude a variável de ambiente PATH de forma a que, aquando da chamada da função system("ls") num programa SET-UID com dono root, ao invés de chamar o comando regular *ls*, corra código arbitrário.

Para isso, colocou-se um executável num diretório e adicionou-se ao início do PATH. Como o programa chama ls sem indicar o caminho absoluto, a shell procura o comando pelas pastas do PATH e encontra o nosso primeiro.

Na configuração do lab, /bin/sh aponta para /bin/dash, que apresenta uma contramedida. Para realizar o ataque com privilégios root, é alterada a ligação /bin/sh para apontar para /bin/zsh, como pedido no guião. A execução de whoami através do código chamado permite verificar os privilégios obtidos.

No fim do exercício é necessário repor a ligação original com `sudo ln -sf /bin/dash /bin/sh`.

![Task 6](/task6.png)

## Questão 2

#### Task 8

Nesta task o utilizador seed consegue apagar um ficheiro que não lhe é permitido apagar diretamente através do uso da função system() no código de catall.c

O programa junta o argumento dado pelo utilizador ao comando /bin/cat e passa essa string a system(). Ao incluir um separador como `;` seguido de outro comando dentro do argumento, a shell executa também esse comando. O argumento deve ser passado entre aspas para que o separador seja interpretado pela shell chamada por catall, e não pela shell inicial do utilizador

![Task 8.1](/questao2-1.png)

Após a alteração de uso de system() para execve(), já não é possível realizar a injeção de comandos

![Task 8.2](/questao2-2.png)

## Questão 3

#### Task 9

Nesta task é explorada a vulnerabilidade de capability leaking no programa cap_leak.c. O objetivo é escrever no ficheiro /etc/zzz como utilizador normal, mesmo depois de o programa abdicar dos privilégios root.

**Preparação do ambiente**

Primeiro compilamos o programa, alteramos o dono do executável para root e ativamos o SET-UID:

```bash
gcc cap_leak.c -o cap_leak
sudo chown root:root cap_leak
sudo chmod 4755 cap_leak
```

Antes de executar o programa, criamos o ficheiro alvo /etc/zzz, com dono root e permissões 0644:

```bash
sudo touch /etc/zzz
sudo chown root:root /etc/zzz
sudo chmod 0644 /etc/zzz
```

Estas permissões permitem que root leia e escreva no ficheiro, enquanto os restantes utilizadores apenas o podem ler.

**Evidência da execução**

Num terminal do utilizador seed, verificamos o utilizador atual e as permissões dos dois ficheiros:

```bash
whoami
ls -l ./cap_leak /etc/zzz
```
Depois, corre-se o programa

```bash
./cap_leak
```

O programa apresenta o número do descritor aberto, por exemplo:

```text
fd is 3
```

Na shell aberta pelo programa, verificamos que estamos como seed e escrevemos através desse descritor:

```bash
whoami
echo "Escrita no ficheiro zzz" >&3
cat /etc/zzz
```


![Task 3](/questao3.png)

**Explicação técnica**

O programa abre /etc/zzz enquanto ainda tem privilégios root:

```c
fd = open("/etc/zzz", O_RDWR | O_APPEND);
```

Como nesse momento tem permissão de escrita, o open() devolve um descritor que permite ler e escrever no ficheiro. O O_APPEND faz com que as escritas sejam adicionadas ao fim do ficheiro.

Depois, o programa abdica dos privilégios root:

```c
setuid(getuid());
```

No entanto, esta chamada não fecha o descritor que já estava aberto. Ao executar /bin/sh através de execve(), o descritor continua disponível, pois não foi fechado nem marcado para fechar durante o execve().

Assim, mantém o acesso ao ficheiro através do descritor aberto anteriormente. A escrita por esse descritor não volta a verificar as permissões normais do ficheiro como aconteceria numa nova abertura.

O comando `>&3` usa esse descritor como destino da escrita, em vez de tentar abrir /etc/zzz novamente. É por isso que a escrita direta falha, mas a escrita através do descritor funciona.

O processo perde os privilégios root, mas mantém um recurso obtido enquanto estava privilegiado.

**Medidas de mitigação**

Para evitar esta vulnerabilidade, o programa deve fechar o descritor privilegiado com `close(fd)` antes de entregar o controlo ao utilizador.

Também pode abrir o ficheiro com O_CLOEXEC ou definir FD_CLOEXEC, para que o descritor seja fechado automaticamente quando execve() executar a shell. Esta medida protege a passagem para outro programa, mas não fecha o descritor apenas por ser chamado setuid().

Quando possível, os ficheiros devem ser abertos depois de abdicar dos privilégios, para que a abertura seja feita com as permissões do utilizador normal. Também se deve verificar o resultado de open(), setuid() e execve(), evitar SET-UID quando não é necessário e manter apenas os privilégios indispensáveis.

**Evidência final e conclusões**

Depois da escrita, saímos da shell criada pelo programa e verificamos o conteúdo e as permissões do ficheiro:

```bash
exit
cat /etc/zzz
ls -l /etc/zzz
```

Asimm mostramos que abdicar dos privilégios através de setuid() não é suficiente se o programa continuar a disponibilizar recursos abertos enquanto tinha privilégios root.
