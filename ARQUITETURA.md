# Arquitetura e decisões do Deck

## Objetivo

Criar o Deck como sistema operacional próprio, escrito em C/Assembly, com ambiente Unix, **kernel próprio** e objetivo futuro de executar aplicações externas. A identidade do produto é exclusivamente Deck.

A versão 0.2.8 acrescenta execução protegida e acesso a arquivos por programas. Uma estrutura de diretórios familiar e comandos com nomes iguais não garantem conformidade POSIX nem compatibilidade com binários externos.

## Decisões atuais

| Área | Decisão implementada | Razão / consequência |
| --- | --- | --- |
| CPU | AMD64 / x86-64, uma CPU | Alvo comum no VirtualBox. i386 exige outro alvo e imagem. |
| Linguagens | C11 freestanding e Assembly x86 | Sem runtime de outro SO; ferramentas hospedeiras não entram na imagem. |
| Kernel | Monolítico mínimo | Drivers e terminal inicial no kernel; shell ELF opcional; um processo ELF64 em ring 3 por vez. |
| Boot | Bootloader próprio BIOS, 512 bytes | Entrada em long mode sem GRUB, Limine ou kernel externo. UEFI é etapa separada. |
| ISO | El Torito, emulação de disquete de 1,44 MiB | BIOS lê o boot sector, que lê 256 setores reservados ao kernel. |
| Memória | Identidade do primeiro GiB com páginas de 2 MiB | Mapeamento inicial; não é alocação nem detecção de 1 GiB de RAM. |
| Console | VGA 80×25, PS/2 US, COM1 | Adequado à VM BIOS, sem suporte gráfico moderno. |
| Identidade | Root UID 0, wheel GID 0 | Sem autenticação ou usuários comuns nesta versão. |
| Armazenamento | DeckFS, RAM + snapshots em ATA PIO | Implementação pequena e independente. Não equivale a UFS/ZFS. |
| Instalação | Disco IDE primário inteiro, confirmação explícita | Não permite escolher outros discos ou partições. Disco virtual novo recomendado. |
| Pacotes | Não implementados | Falta a infraestrutura exigida pelos executáveis externos. |

## Memória e boot

- `0x0500`: unidade BIOS usada no boot (transmitida ao kernel).
- `0x1000..0x3fff`: tabelas PML4, PDPT e PD.
- `0x7c00`: bootloader BIOS.
- `0x5000`: contagem/assinatura E820; entradas de mapa a partir de `0x5010`.
- `0x10000`: consulta BIOS E820; entrada em long mode em `0x10200`. Texto, constantes e dados usam uma reserva de 131072 bytes.
- `0x200000`: início de BSS. Linker recusa BSS que alcance a área reservada à instalação.
- `0x3ff000`: cópia do bootloader para instalação.
- `0x400000`: cópia dos 256 setores carregados, preservada antes de alterar dados do kernel.
- `0x800000`: topo inicial da pilha.

O bootloader lê o kernel via BIOS CHS no CD com emulação de disquete ou EDD/LBA no disco. Habilita A20, GDT, PAE, EFER.LME e paginação, e entra no kernel em 64 bits. A entrada Assembly preserva a imagem original, zera BSS e chama `kmain()`.

O C configura GDT/TSS/IDT, console, PIT, filesystem e ATA. Interrupções ficam desabilitadas no kernel, exceto durante read_wait; o processo de usuário habilita o timer, que limita sua execução a aproximadamente três segundos. Teclado, serial e disco usam polling. Falhas de usuário encerram o processo com diagnóstico e devolvem o controle ao pai em espera ou ao shell; falhas do kernel param a CPU.

Um pool de 128 páginas é selecionado em RAM verificada pelo mapa E820 a partir de 16 MiB. O processo possui mapeamentos em 0x40000000, com código protegido contra escrita, pilha NX e memória do kernel restrita ao supervisor. A ABI própria oferece write, exit, read, open, close, exec, spawn_wait e read_wait; argc/argv ficam na pilha de usuário. Descritores aplicam modos de leitura/escrita e são liberados ao terminar.

Exec prepara a nova imagem em páginas separadas e só troca os mapeamentos após sucesso completo. Erros liberam a preparação e preservam o chamador. A troca reescreve o quadro de retorno da syscall, mantendo PID e tempo já consumido. Descritores são herdados, salvo os marcados com DECK_O_CLOEXEC. Spawn_wait inicia filhos com o pai bloqueado.

O kernel não usa serviços BIOS depois da transição para long mode. O driver de disco instalado é ATA PIO LBA28, primário mestre. Os dispositivos precisam corresponder à configuração documentada.

## Formato DeckFS v1

O disco não tem tabela de partições convencional. LBA 0 contém o bootloader; LBAs 1–256 contêm a reserva do kernel. As duas áreas de snapshots começam nos LBAs 2048 e 4096, cada uma com espaço de 1024 setores.

O primeiro setor de cada snapshot contém `DECKFS1`, versão, tamanho, sequência e CRC32 do payload. O payload é a estrutura de filesystem: hostname de 64 bytes e 96 entradas. Cada entrada armazena tipo, modo, UID, GID, tamanho, caminho de 160 bytes e conteúdo de 4096 bytes. Todos os campos inteiros usam o layout little-endian do alvo; a versão do formato deve mudar se o layout mudar.

Ao salvar, o kernel escreve o payload do slot alternado, solicita flush, grava o header por último e solicita outro flush. Ao iniciar, aceita somente os snapshots com metadados válidos e checksum correto, escolhendo a sequência mais nova. Uma gravação interrompida pode perder alterações recentes; o outro slot preserva a versão anterior se ainda estiver íntegro. O CRC do payload detecta corrupção acidental, sem autenticação criptográfica. O header não tem checksum próprio; isso limita a robustez a cenários simples de falha.

Arquivos são editados em RAM. `sync` é explícito, com tentativa automática antes de desligar/reiniciar. Não há journaling completo, permissões efetivas, timestamps, links, arquivos especiais ou consistência entre processos concorrentes.

## Para decidir nas próximas versões

1. **Processos e chamadas de sistema:** ampliar APIs de diretório e o shell em userland; manter isolamento e validação de memória.
2. **Compatibilidade binária externa:** recomendação: criar uma camada de ABI separada e testável; não falsificar `uname`/ABI para baixar pacotes antes de conseguir executá-los.
3. **Filesystem definitivo:** recomendação: VFS primeiro; depois decidir entre implementar UFS2 ou adotar uma implementação licenciada compatível. DeckFS serve apenas para o protótipo. ZFS adiciona complexidade e requisitos maiores.
4. **Boot moderno:** recomendação: adicionar um carregador UEFI próprio em C, mantendo o kernel independente. Firmware UEFI não é um kernel externo.
5. **Segurança:** processos isolados, credenciais, permissões e autenticação antes de habilitar serviços de rede ou guardar dados reais.
6. **Atualizações e pacotes:** pacotes nativos Deck antes da compatibilidade binária externa pode reduzir dependências; o objetivo `pkg install vim` direto do repositório externo desejado continua exigindo ABI e bibliotecas compatíveis.
7. **32 bits:** adiar i386 até estabilizar amd64. Suporte a binários i386 em CPU amd64 e boot em CPU somente 32 bits são trabalhos diferentes.

## Referências de engenharia

- [Intel SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html): modos de execução, paginação, descritores e arquitetura x86.
- `/usr/src/usr.sbin/makefs/cd9660/cd9660_eltorito.c` no hospedeiro: investigado para corrigir o número de setores inicialmente carregados no catálogo El Torito. A ferramenta própria `scripts/isofix.c` valida o catálogo e ajusta esse campo para 1.

Na 0.2.8, spawn_wait guarda o frame, as páginas, os descritores e o orçamento
do pai. O filho usa páginas físicas diferentes no mesmo intervalo virtual.
Uma única pilha de entrada do kernel atende ambos sem reentrada: somente o
frame de retorno é substituído, sem suspender uma chamada C no kernel.
Ao terminar, o filho libera suas páginas e restaura o pai. Há quatro contextos
no máximo, em cadeia, sem execução simultânea. O pool de 128 páginas é global.

Na 0.2.8, há quatro contextos em cadeia e 128 páginas no pool global. O shell
ELF `/bin/sh` é iniciado manualmente. read_wait espera um byte em ring 0 com
interrupções habilitadas e HLT; timer em ring 0 não consome orçamento de usuário.
Nenhum outro processo é escalonado durante a espera. O terminal inicial do
kernel permanece disponível para administração e autocompletar com TAB.

Na 0.2.8, o diretório ativo acompanha o contexto: spawn copia, exec preserva,
e a saída restaura o pai ou o terminal inicial. Chdir/getcwd/readdir validam
ponteiros e tamanhos na fronteira da ABI. O shell ELF oferece cd/pwd/ls e TAB;
complete reutiliza a busca de candidatos do kernel com a lista de comandos
do shell ELF. Read_wait renova o orçamento de usuário ao receber entrada real.

Na 0.2.8, descritores 0..10 apontam para descrições de arquivo. Dup/dup2
compartilham posição; spawn copia a tabela de descritores e compartilha as
descrições com o pai, incluindo posição e flags de acesso. CLOEXEC pertence ao descritor.
O shell salva os streams afetados com CLOEXEC, aplica redirecionamentos em
ordem e restaura os streams após a execução. Read_wait lê arquivos até EOF
quando stdin é redirecionado; apenas a entrada real de terminal renova CPU.

Na 0.2.9, a cadeia de pais foi substituída por quatro slots de processos, com
estados runnable, espera de filho, leitura/escrita de pipe, terminal e zombie.
O timer alterna os prontos. Chamadas bloqueadas guardam o frame e repetem o
int 0x80 quando a condição permite progresso; os ponteiros são revalidados.
Fork copia páginas físicas; exec e spawn continuam preparando a imagem antes
de commit. O diretório e a tabela de páginas ativos são trocados junto com o
frame. Pipes têm buffer circular de 1024 bytes e referências contadas nas
mesmas descrições globais usadas por arquivos. Veja ALTERACOES-0.2.9.md para
os limites, tratamento de EOF, término de descendentes e recuperação de deadlock.

Na 0.2.10, keyboard.h separa tradução de scancodes alemães, composição e fila
UTF-8. O timer e as syscalls observam Ctrl-C mesmo quando o usuário executa
um loop sem leitura. O grupo corresponde à árvore abaixo do shell interativo
mais próximo; o encerramento mantém zombies dos filhos diretos para wait e
libera os recursos imediatamente. A criação de novos filhos fica bloqueada
até o próximo prompt após cancelamento, evitando uma corrida durante fork.

History.h fornece dois históricos voláteis de 16 linhas. O editor compartilhado
trata UTF-8, TAB e setas; a syscall 18 verifica buffer, terminal e ausência de
outros processos ativos antes de atendê-lo. A lógica do shell permanece em
ring 3. Unicode.h decodifica a saída VGA e instala glifos latinos/símbolos
adicionais; a serial recebe UTF-8 sem conversão. O acesso à fonte VGA segue
os registradores documentados no driver local /usr/src/sys/dev/fb/vga.c;
o código e os glifos adicionados ao Deck são próprios.

## Sessão e contas na 0.3.0

Boot inicia init/login/sh em ring 3. UID/GID e 16 variáveis de ambiente fazem
parte de cada tarefa; exec preserva, fork/spawn herdam. Login autenticado
fecha descritores adicionais, redefine ambiente/cwd e limpa o histórico.
Syscalls 19..23 acrescentam ambiente, administração limitada, operações de
filesystem, contas e entrada sem eco. O console do kernel exige autorização
root na sessão interativa e é usado para recuperação.

DeckFS v2: 120 nós, 8192 bytes por arquivo, snapshots em LBA 2048/4096,
validação por CRC; não migra o layout anterior. O kernel tem reserva de
384 setores e BSS em 5 MiB, abaixo da pilha em 8 MiB. O escalonador dispõe
de 8 tarefas, 88 descrições, 8 pipes e pool de 256 páginas.

`permissions.h`, `fsops.h`, `accounts.h` e `password.h` concentram permissões,
operações, banco de contas e derivação de senhas. A base usa executáveis
multicall com despacho por basename; os nomes suportados são os instalados
em /bin. Nome desconhecido falha sem executar operações em arquivos.

`scripts/seed-disk.c` usa os mesmos seeds do kernel para criar um disco
instalado limpo, sem credenciais predefinidas; isso prepara a VM de entrega.
A inicialização desse disco foi verificada até a configuração inicial de root.
