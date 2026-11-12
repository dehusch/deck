# Roteiro para um sistema Unix próprio

## 0.1 — terminal inicializável (implementado)

Boot BIOS, kernel x86-64 próprio, console e teclado, comandos internos, diretórios Unix, identidade root, instalador IDE e persistência experimental. Esta entrega não fecha o objetivo de sistema base completo.

## 0.2 — fundação do kernel

- Mapa de RAM E820/UEFI, alocador de páginas e heap, proteção de páginas e pilhas.
- Tratamento de exceções com contexto e diagnóstico, interrupções, timer e escalonador.
- Transição ring 0/ring 3, espaço de endereços por processo, ELF64 estático.
- Chamadas de sistema mínimas: leitura/escrita, open/close, exit, wait e criação/execução de processos.
- Teste de aceitação: programa C em userland escreve no terminal e termina; acesso inválido mata o processo sem derrubar o kernel.

## 0.3 — sistema base Unix

- `init` e shell como programas reais, libc mínima, descritores, pipes, redirecionamento geral e sinais.
- `fork`, `execve`, `waitpid`, `mmap`, `ioctl`, terminais e conjunto de syscalls necessário.
- VFS, filesystem persistente robusto e montagem de pseudoarquivos em `/dev` e `/proc` se escolhido.
- Contas e grupos, credenciais efetivas, permissões, login e armazenamento adequado de hashes de senha.
- Executáveis separados para utilitários, páginas de manual e testes de comportamento Unix.
- Teste de aceitação: boot chama `/sbin/init`, abre login, executa `/bin/sh`; usuário comum não modifica arquivos protegidos.

## 0.4 — dispositivos e rede

UEFI, ACPI, PCI, AHCI ou virtio-blk, driver de rede adequado à VM, Ethernet, ARP, IPv4, ICMP, UDP/TCP, DHCP e DNS; USB e NVMe depois, conforme o hardware alvo. TLS exige biblioteca de criptografia, entropia confiável, relógio e validação de certificados.

Teste de aceitação: obter IP por DHCP e baixar um arquivo por HTTPS com validação de certificado.

## 0.5 — compatibilidade binária externa

A compatibilidade dos pacotes não vem da árvore de diretórios. Implementar e testar:

- ABI ELF amd64 do sistema de referência, notas ELF, pilha inicial, auxv, TLS, linker dinâmico e convenções de chamada.
- Chamadas de sistema e comportamento de retorno/errno suficientes, sinais, threads, sincronização, sockets, polling/kqueue e operações de filesystem.
- Bibliotecas compatíveis e runtime esperado; verificar licenças e preservação de avisos ao incorporar código de terceiros.
- Identificação de ABI para `pkg`, catálogo, dependências, assinatura dos repositórios, extração, scripts pós-instalação e banco de pacotes.
- Compatibilidade de versão: pacotes podem depender de funcionalidades de kernels externos específicos, inclusive módulos, que não passam a funcionar automaticamente.

Começar com um binário pequeno e estático. Depois ELF dinâmico, libc, shell e aplicações simples. Somente então validar um `pkg` real e o Vim.

**Critério final solicitado pelo usuário:** em instalação limpa, com rede, `pkg install vim` acessa o repositório externo compatível, valida e instala o pacote e suas dependências; `vim` inicia e edita um arquivo que persiste após reboot. Esse critério ainda não foi atingido.

## Evolução para uso real

Testes em múltiplas máquinas virtuais e hardware físico, recuperação de falhas, atualização do kernel e base, instalador com particionamento, documentação de segurança, compatibilidade e desempenho. Nenhuma estimativa curta para “Unix completo” é assumida: isso é um projeto de engenharia continuado.

## Estado em 0.2.9

Concluídos no protótipo: ELF protegido, leitura/escrita de arquivos, exec,
spawn_wait, shell ELF com diretórios e TAB, dup/dup2, redirecionamentos e
pipelines de até três etapas com fork/wait e escalonamento. A próxima etapa
proposta é Ctrl-C para o grupo em primeiro plano. O objetivo final de pacotes
e Vim permanece nas etapas de rede, runtime e compatibilidade acima.

## Estado em 0.2.10

Teclado alemão/UTF-8, histórico e edição do terminal e interrupção do grupo
em primeiro plano foram entregues em conjunto. Agrupar tarefas relacionadas
em marcos utilizáveis, sem exigir confirmação entre melhorias do mesmo
terminal. A proposta seguinte reúne a sessão padrão em userland e sua base
de utilitários/ambiente; detalhes em PROXIMO-MARCO.md.

## Estado em 0.3.0

Boot por init/login/sh, contas com senhas, UID/GID por processo, permissões,
utilitários externos, ambiente/PATH e scripts simples estão implementados.
O critério de sessão e proteção dos arquivos do sistema foi validado na VM.
Isso não encerra todos os itens amplos da etapa 0.3: VFS, mmap/ioctl, sinais
POSIX, libc e terminais completos ainda estão pendentes. Rede/pkg/Vim também.
