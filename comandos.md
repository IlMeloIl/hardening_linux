# 1. Preparação inicial da VM

## `sudo apt update`

**O que faz:** atualiza a lista local de pacotes disponíveis nos repositórios configurados no Ubuntu. Esse comando não atualiza os programas instalados; ele apenas sincroniza o índice que o `apt` usa para saber quais versões existem e de onde elas podem ser baixadas.

**Por que foi usado:** o primeiro passo é garantir que o sistema esteja trabalhando com informações atualizadas dos repositórios. Se a lista local estiver desatualizada, a VM pode instalar versões antigas de ferramentas de segurança ou deixar de encontrar pacotes necessários. Isso afeta diretamente a confiabilidade da prática, porque hardening não é só configuração: ele também depende de gestão de atualizações. Um servidor com pacotes desatualizados pode continuar vulnerável mesmo que firewall, SSH e auditoria estejam configurados corretamente.

---

## `sudo apt full-upgrade -y`

**O que faz:** atualiza os pacotes instalados para as versões mais recentes disponíveis. O `full-upgrade` pode instalar novas dependências ou remover pacotes quando isso é necessário para concluir a atualização corretamente. A opção `-y` confirma automaticamente as perguntas do gerenciador de pacotes.

**Por que foi usado:** antes de endurecer a configuração do sistema, é necessário reduzir a exposição a vulnerabilidades já conhecidas e corrigidas pelos mantenedores da distribuição. Esse comando representa a etapa de correção de software, ou patch management. Ele é diferente do hardening de configuração, mas complementa o processo: atualizar pacotes reduz falhas conhecidas no código; hardening reduz riscos de configuração, exposição e abuso de recursos. Na apresentação, esse comando deve ser explicado como a preparação do sistema para que a auditoria e os controles posteriores sejam aplicados sobre uma base atualizada.

---

## `sudo apt install -y lynis ufw auditd audispd-plugins libpam-pwquality openssh-server curl wget net-tools`

**O que faz:** instala as ferramentas básicas usadas na PoC. O `lynis` realiza auditoria local de segurança. O `ufw` permite configurar firewall de forma simplificada. O `auditd` registra eventos de auditoria do sistema. O `audispd-plugins` complementa o processamento de eventos de auditoria. O `libpam-pwquality` permite aplicar critérios de qualidade de senha no PAM. O `openssh-server` garante o serviço SSH usado para administração remota da VM. `curl`, `wget`, `net-tools` são utilitários auxiliares para operação no terminal.

**Por que foi usado:** essa etapa instala a base técnica da PoC. O Lynis é necessário para medir a postura de segurança antes e depois. O UFW é necessário para restringir conexões de entrada. O auditd é necessário para rastrear alterações sensíveis no sistema. O PAM/pwquality é necessário para controlar qualidade de senhas. O OpenSSH é necessário porque a administração da VM é feita a partir do host via SSH. Sem essas ferramentas, a PoC ficaria incompleta: não haveria uma auditoria inicial, não haveria aplicação de controles básicos e não haveria evidência técnica suficiente para demonstrar melhoria.

---

## `sudo reboot`

**O que faz:** reinicia a VM.

**Por que foi usado:** após atualizações de sistema, especialmente quando envolvem kernel, bibliotecas ou serviços essenciais, reiniciar garante que a VM passe a executar o estado atualizado. Sem reboot, o sistema pode estar com pacotes atualizados em disco, mas ainda executando processos antigos em memória. Na PoC, isso evita comparar um sistema parcialmente atualizado com um sistema endurecido depois, tornando as medições mais consistentes.

---

# 2. Identificação do ambiente da PoC

## `date | tee 00_ambiente/data_execucao.txt`

**O que faz:** exibe a data e a hora do momento da coleta e salva a mesma saída em arquivo por meio do `tee`.

**Por que foi usado:** registra quando a PoC foi executada. Em segurança, data e hora são importantes porque resultados de auditoria dependem do contexto temporal: versão de pacotes, estado dos repositórios, versão do Lynis, kernel em uso e configurações presentes naquele momento. Esse registro ajuda a mostrar que as evidências pertencem a uma execução específica e não são saídas soltas sem contexto.

---

## `lsb_release -a | tee 00_ambiente/versao_ubuntu.txt`

**O que faz:** mostra informações da distribuição Linux, como nome, versão e codinome.

**Por que foi usado:** comprova qual sistema operacional foi usado na PoC. Isso importa porque comandos, caminhos de configuração, nomes de serviços e recomendações de hardening podem variar entre distribuições e versões. Uma configuração aplicada no Ubuntu Server 24.04.4 LTS pode não ter exatamente o mesmo comportamento em Debian, Fedora ou versões antigas do Ubuntu. Na apresentação, esse comando contextualiza o laboratório.

---

## `uname -a | tee 00_ambiente/kernel.txt`

**O que faz:** exibe informações do kernel em execução, arquitetura, hostname e versão de build.

**Por que foi usado:** muitos controles de hardening atuam diretamente no kernel ou dependem dele, principalmente os parâmetros de rede configurados com `sysctl`, proteções de links simbólicos, ASLR e módulos bloqueados. Registrar o kernel permite explicar que os resultados pertencem ao kernel carregado naquela VM. Isso também ajuda em auditoria e reprodução do experimento.

---

## `whoami | tee 00_ambiente/usuario.txt`

**O que faz:** mostra o usuário atualmente autenticado na sessão.

**Por que foi usado:** documenta que a sessão foi executada com o usuário comum `aluno`, elevando privilégios apenas quando necessário com `sudo`. Isso é relevante para segurança porque administração direta como root é uma prática mais arriscada. Usar usuário comum com sudo cria uma separação entre conta de uso e operações privilegiadas, além de facilitar rastreabilidade.

---

## `hostnamectl | tee 00_ambiente/hostnamectl.txt`

**O que faz:** mostra informações consolidadas do host, como hostname, sistema operacional, kernel, arquitetura e ambiente de virtualização quando identificado.

**Por que foi usado:** reforça a identificação do ambiente. Na PoC, isso ajuda a provar que a prática foi executada em uma VM específica, com hostname próprio, evitando confusão com o host físico. Esse ponto é importante porque alguns comandos, como `aa-status`, podem gerar saídas diferentes se forem executados por engano no host em vez da VM.

---

## `ip a | tee 00_ambiente/interfaces_rede.txt`

**O que faz:** lista interfaces de rede, endereços IP, estado das interfaces e informações de camada de rede.

**Por que foi usado:** registra como a VM estava conectada. Como o ambiente usa NAT no VirtualBox, é esperado que a VM tenha um IP privado interno, como `10.0.2.15`, e que o acesso SSH ocorra por redirecionamento de porta no host. Esse comando ajuda a explicar por que o SSH precisa continuar habilitado e por que a política de firewall deve permitir OpenSSH. Ele também diferencia a rede interna da VM da rede do computador físico.

---

# 3. Coleta do estado inicial antes do hardening

## `systemctl list-units --type=service --state=running | tee 01_antes/servicos_ativos_antes.txt`

**O que faz:** lista unidades do tipo serviço que estão carregadas e em execução no systemd. O filtro `--type=service` limita a saída a serviços, e `--state=running` mostra apenas os que estão efetivamente rodando.

**Por que foi usado:** identifica a superfície de serviços antes do hardening. Superfície de serviços é o conjunto de processos de sistema que estão ativos e oferecendo alguma função, como SSH, resolução de nomes, logs, rede, agendamento de tarefas ou gerenciamento de dispositivos. Cada serviço ativo aumenta a responsabilidade de administração: ele pode ter falhas, configuração insegura, permissões excessivas ou funcionalidades desnecessárias para o papel do servidor. Em hardening, não basta perguntar “o sistema está funcionando?”; é preciso perguntar “quais serviços estão rodando e realmente precisam existir?”. Esse comando cria a linha de base para comparar se havia serviços desnecessários e se a VM já era uma instalação mínima.

---

## `ss -tulpen | tee 01_antes/portas_antes.txt`

**O que faz:** lista sockets de rede e portas em escuta. A opção `-t` mostra TCP, `-u` mostra UDP, `-l` limita a portas em escuta, `-p` mostra o processo associado, `-e` exibe informações estendidas e `-n` evita resolver nomes, mostrando IPs e portas numericamente.

**Por que foi usado:** mede a superfície de rede antes do hardening. Superfície de rede é o conjunto de portas, protocolos e endereços pelos quais a máquina pode receber ou processar tráfego. Uma porta em escuta significa que algum processo está aguardando conexões ou pacotes. Isso não é automaticamente vulnerável, mas representa um ponto de contato entre a VM e a rede. Quanto mais portas abertas, maior a quantidade de serviços que precisam estar corretos, atualizados e bem configurados. No caso da PoC, as portas não mudaram muito porque a VM já estava enxuta; o ponto importante foi mostrar que o SSH continuou necessário e que o firewall passou a controlar o acesso a ele.

---

## `sudo ufw status verbose | tee 01_antes/firewall_antes.txt`

**O que faz:** mostra o status detalhado do UFW, incluindo se o firewall está ativo, política padrão e regras existentes.

**Por que foi usado:** verifica se havia controle de tráfego antes do hardening. Firewall é um mecanismo que filtra conexões de rede conforme regras definidas. Ele funciona como uma barreira lógica: mesmo que um serviço esteja escutando em uma porta, o firewall pode permitir, negar ou limitar o tráfego que chega até ele. Na PoC, esse comando mostrou o estado inicial do UFW. Se o firewall está inativo, qualquer serviço exposto depende apenas da própria configuração do serviço para se proteger. Ativar o firewall depois demonstra aplicação do princípio de menor exposição: negar por padrão e liberar apenas o que é necessário.

---

## Bloco `sudo sysctl ... | tee 01_antes/sysctl_antes.txt`

```bash
sudo sysctl net.ipv4.ip_forward \
net.ipv4.tcp_syncookies \
net.ipv4.conf.all.accept_redirects \
net.ipv4.conf.all.rp_filter \
| tee 01_antes/sysctl_antes.txt
```

**O que faz:** consulta parâmetros de kernel e rede em tempo real. `net.ipv4.ip_forward` indica se a máquina pode encaminhar pacotes IPv4 como roteador. `net.ipv4.tcp_syncookies` indica se há mitigação para SYN flood. `accept_redirects` indica se o host aceita mensagens ICMP Redirect. `rp_filter` indica a política de reverse path filtering.

**Por que foi usado:** registra a configuração inicial de rede no nível do kernel. Esses parâmetros controlam comportamentos importantes. Roteamento IPv4, ou `ip_forward`, é necessário em roteadores, gateways e firewalls, mas não em uma VM servidor simples; se habilitado sem necessidade, a máquina poderia encaminhar tráfego entre redes, aumentando o risco de uso indevido. SYN flood é um ataque de negação de serviço em que o atacante inicia muitas conexões TCP e não as completa, tentando consumir recursos da pilha TCP; SYN cookies ajudam o kernel a lidar melhor com esse cenário. ICMP Redirect é uma mensagem usada para informar a um host que existe uma rota melhor; em servidores comuns, aceitar isso pode permitir manipulação de rotas em cenários de rede maliciosa ou mal configurada. Reverse path filtering verifica se o caminho de retorno de um pacote faz sentido pela interface por onde ele chegou, ajudando a reduzir pacotes com endereço de origem falsificado. A coleta inicial mostra o que já estava seguro por padrão e o que precisava ser ajustado.

---

# 4. Auditoria inicial com Lynis

## `sudo lynis audit system --quick --no-colors | tee 01_antes/lynis_tela_antes.txt`

**O que faz:** executa uma auditoria local do sistema com Lynis. O modo `audit system` avalia o host local. A opção `--quick` executa de forma mais direta, e `--no-colors` remove códigos de cor para facilitar leitura e salvamento.

**Por que foi usado:** cria a linha de base da postura de segurança antes do hardening. Sem uma auditoria inicial, não haveria como demonstrar evolução de forma mensurável. O Lynis verifica áreas como kernel, autenticação, pacotes, serviços, permissões, logs, rede e configurações gerais. O objetivo não é tratar o índice como uma nota absoluta de segurança, mas como indicador comparativo. Na apresentação, esse comando representa a fase “medir antes de alterar”.

---

## Bloco de resumo do Lynis inicial

```bash
grep -E '^hardening_index=|^warning\[\]=|^suggestion\[\]=' \
01_antes/lynis-report-antes.dat \
| tee 01_antes/resumo_lynis_antes.txt

echo "Hardening index antes:"
grep '^hardening_index=' 01_antes/lynis-report-antes.dat

echo "Warnings antes:"
grep -c '^warning\[\]=' 01_antes/lynis-report-antes.dat

echo "Suggestions antes:"
grep -c '^suggestion\[\]=' 01_antes/lynis-report-antes.dat
```

**O que faz:** extrai do relatório do Lynis as métricas principais: hardening index, warnings e suggestions. O `grep -E` filtra linhas por expressão regular. O `grep -c` conta quantas ocorrências existem.

**Por que foi usado:** transforma o relatório extenso do Lynis em números fáceis de comparar. O hardening index mostra uma visão geral da postura avaliada pela ferramenta. Warnings representam alertas mais relevantes. Suggestions representam recomendações de melhoria. Esses números são importantes para a PoC porque permitem demonstrar evolução objetiva: antes do hardening, a VM tinha hardening index 63, 1 warning e 45 suggestions. A explicação para o vídeo é que esse bloco cria a métrica inicial que será comparada depois das rodadas de hardening.

---

# 5. Primeira rodada de hardening: firewall UFW

## `sudo ufw default deny incoming`

**O que faz:** define a política padrão do firewall para negar conexões de entrada que não tenham regra explícita de permissão.

**Por que foi usado:** aplica o princípio de menor exposição. Em vez de permitir tudo e bloquear apenas o que parece perigoso, o sistema passa a negar por padrão e liberar somente o necessário. Isso é relevante porque serviços podem ser instalados futuramente ou começar a escutar em portas sem que o administrador perceba. Com `deny incoming`, mesmo que um novo serviço abra uma porta, ele não fica automaticamente acessível pela rede. Na PoC, esse é um dos controles mais claros para reduzir risco de exposição indevida.

---

## `sudo ufw default allow outgoing`

**O que faz:** define a política padrão para permitir conexões de saída.

**Por que foi usado:** mantém a VM funcional para operações normais, como consultar DNS, acessar repositórios, baixar atualizações e enviar conexões iniciadas pelo próprio sistema. Em servidores reais, tráfego de saída também pode ser restringido, mas isso exige conhecimento detalhado das aplicações. Para a PoC, permitir saída evita quebrar a operação básica enquanto o foco é reduzir a exposição de entrada.

---

## `sudo ufw allow OpenSSH`

**O que faz:** cria uma regra permitindo conexões para o perfil OpenSSH.

**Por que foi usado:** evita que o firewall bloqueie o próprio acesso administrativo. Como a VM é acessada do host por SSH, a porta 22 precisa continuar permitida. Essa ordem é importante: primeiro libera-se o OpenSSH, depois ativa-se o firewall. Se o firewall fosse ativado antes de permitir SSH, haveria risco de perder acesso remoto à VM. Na apresentação, esse comando mostra que hardening não é “bloquear tudo”; é bloquear o que não precisa e manter o necessário de forma controlada.

---

## `sudo ufw --force enable`

**O que faz:** ativa o UFW sem pedir confirmação interativa.

**Por que foi usado:** torna efetivas as políticas e regras configuradas. Antes desse comando, as regras podem existir, mas o firewall ainda pode estar inativo. O `--force` foi usado para automação, evitando a pergunta de confirmação. O ponto técnico é que, a partir daqui, a VM passa a ter filtragem ativa de tráfego de entrada.

---

## `sudo ufw status verbose | tee 02_hardening/firewall_configurado.txt`

**O que faz:** mostra o estado detalhado do firewall após a configuração.

**Por que foi usado:** valida o controle aplicado. A saída esperada deve mostrar UFW ativo, política `deny incoming`, política `allow outgoing` e regra permitindo OpenSSH. Essa evidência é importante porque não basta executar comandos de configuração; é preciso demonstrar o estado final. Na PoC, esse print mostra a diferença entre o firewall inicialmente inativo e o firewall ativo com regra mínima necessária.

---

# 6. Primeira rodada de hardening: SSH

## Bloco de configuração SSH

```bash
cat << 'EOF' | sudo tee /etc/ssh/sshd_config.d/99-av4-hardening.conf
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
EOF
```

**O que faz:** cria ou atualiza um arquivo de configuração adicional para o OpenSSH Server com parâmetros de hardening. `PermitRootLogin no` impede login direto como root. `MaxAuthTries 3` limita tentativas de autenticação por conexão. `LoginGraceTime 30` reduz o tempo permitido para concluir o login. `X11Forwarding no` desativa encaminhamento gráfico. `AllowTcpForwarding no` desativa túneis TCP via SSH. `ClientAliveInterval` e `ClientAliveCountMax` ajudam a encerrar sessões inativas ou sem resposta.

**Por que foi usado:** o SSH é o principal serviço exposto da VM e, por isso, precisa ser endurecido. Acesso remoto é um dos pontos mais sensíveis de qualquer servidor, porque permite administração do sistema pela rede. Impedir root direto reduz o impacto de ataques contra a conta mais privilegiada. Limitar tentativas e tempo de login reduz a janela para tentativas repetidas. Desativar X11 e tunelamento TCP remove funções que não são necessárias na PoC e que poderiam transformar o SSH em canal para acessar outros recursos. Manter controles de sessão reduz sessões abandonadas. Esse bloco não remove o SSH, porque ele é necessário para administrar a VM, mas reduz os recursos disponíveis e torna o acesso mais controlado.

---

## `sudo /usr/sbin/sshd -t`

**O que faz:** valida a sintaxe e a consistência da configuração do SSH sem reiniciar o serviço.

**Por que foi usado:** evita aplicar uma configuração inválida que poderia derrubar o acesso remoto. Esse é um cuidado crítico em hardening: antes de reiniciar um serviço essencial, é necessário verificar se a configuração é válida. Na apresentação, esse comando mostra preocupação com disponibilidade e rollback, não apenas com segurança. Se o comando não retorna erro, a configuração passou na validação sintática.

---

## `sudo systemctl restart ssh`

**O que faz:** reinicia o serviço SSH para carregar a nova configuração.

**Por que foi usado:** alterações nos arquivos do SSH não necessariamente entram em vigor sozinhas. O restart força o serviço a reler a configuração. Esse comando é necessário para que os controles aplicados passem de “arquivo escrito” para “serviço operando com a nova política”.

---

## `sudo systemctl status ssh --no-pager | tee 02_hardening/ssh_status.txt`

**O que faz:** mostra o estado do serviço SSH após o restart. `--no-pager` faz a saída aparecer diretamente no terminal.

**Por que foi usado:** comprova que o SSH continuou funcionando após o hardening. Isso é importante porque segurança não pode ser avaliada isolada de disponibilidade. Um controle que quebra o acesso administrativo poderia tornar o sistema inutilizável. A evidência esperada é o serviço ativo, indicando que a configuração foi aceita e que o acesso remoto permaneceu operacional.

---

## `cat /etc/ssh/sshd_config.d/99-av4-hardening.conf | tee 02_hardening/ssh_hardening_aplicado.txt`

**O que faz:** exibe o arquivo de configuração SSH aplicado e salva a saída.

**Por que foi usado:** registra exatamente quais parâmetros foram configurados. Esse comando ajuda na explicação porque mostra o controle em formato legível. Porém, é importante explicar que esse arquivo mostra o que foi escrito; a validação final com `sshd -T` mostra a configuração efetiva depois que o OpenSSH interpreta todos os arquivos e padrões.

---

# 7. Primeira rodada de hardening: parâmetros de kernel e rede

## Bloco de configuração sysctl

```bash
cat << 'EOF' | sudo tee /etc/sysctl.d/99-av4-network-hardening.conf
net.ipv4.ip_forward = 0
net.ipv4.tcp_syncookies = 1

net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1

kernel.randomize_va_space = 2

fs.protected_hardlinks = 1
fs.protected_symlinks = 1
EOF
```

**O que faz:** cria um arquivo persistente de parâmetros de kernel e rede. Esses parâmetros controlam comportamentos como roteamento, resposta a mensagens ICMP, proteção contra spoofing, registro de pacotes suspeitos, mitigação de SYN flood, ASLR e proteção contra abusos com links no sistema de arquivos.

**Por que foi usado:** reforça a pilha de rede e o comportamento do kernel para o papel real da VM. A VM é um servidor simples em NAT, não um roteador; por isso `ip_forward = 0` mantém o sistema sem encaminhar pacotes entre redes. `tcp_syncookies = 1` ajuda contra SYN flood, ataque em que muitas conexões TCP incompletas tentam consumir recursos. `accept_redirects = 0`, `secure_redirects = 0` e `send_redirects = 0` reduzem comportamentos de roteamento dinâmico que não são necessários para esse servidor e que poderiam ser abusados para manipular caminhos de rede. `accept_source_route = 0` impede que pacotes tentem determinar seu próprio caminho, recurso antigo e desnecessário em servidores comuns. `rp_filter = 1` ajuda contra pacotes com origem falsificada ao verificar coerência do caminho de retorno. `log_martians = 1` registra pacotes com endereços suspeitos. `icmp_echo_ignore_broadcasts = 1` evita participação em ataques de amplificação via broadcast. `kernel.randomize_va_space = 2` ativa ASLR, dificultando exploração baseada em endereços previsíveis de memória. `fs.protected_hardlinks` e `fs.protected_symlinks` reduzem abusos envolvendo links em diretórios compartilhados. Esse bloco mostra que hardening não é só firewall: também envolve reduzir comportamentos inseguros no kernel.

---

## `sudo sysctl --system | tee 02_hardening/sysctl_aplicacao.txt`

**O que faz:** carrega os parâmetros definidos nos arquivos de configuração sysctl, incluindo arquivos em `/etc/sysctl.d/`.

**Por que foi usado:** aplica imediatamente os parâmetros sem depender de reboot. Escrever o arquivo persistente garante que a configuração exista para próximas inicializações, mas `sysctl --system` faz o kernel carregar os valores naquele momento. Na PoC, isso permite auditar e validar logo em seguida.

---

## Bloco de verificação sysctl

```bash
sudo sysctl net.ipv4.ip_forward \
net.ipv4.tcp_syncookies \
net.ipv4.conf.all.accept_redirects \
net.ipv4.conf.all.rp_filter \
kernel.randomize_va_space \
fs.protected_hardlinks \
fs.protected_symlinks \
| tee 02_hardening/sysctl_verificacao.txt
```

**O que faz:** consulta os principais parâmetros aplicados.

**Por que foi usado:** valida se a configuração realmente entrou em vigor. Em hardening, existe diferença entre “editar arquivo” e “controle efetivo”. Esse comando mostra os valores carregados no kernel naquele momento. Ele também permite comparar com o estado inicial: por exemplo, `accept_redirects` foi ajustado de permitido para bloqueado, e `rp_filter` foi padronizado conforme a configuração definida.

---

# 8. Primeira rodada de hardening: política de senha

## Bloco de configuração `pwquality.conf`

```bash
cat << 'EOF' | sudo tee /etc/security/pwquality.conf
minlen = 12
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
retry = 3
EOF
```

**O que faz:** define requisitos de qualidade de senha para o módulo `pam_pwquality`. `minlen = 12` define tamanho mínimo. `dcredit = -1` exige dígito. `ucredit = -1` exige letra maiúscula. `ocredit = -1` exige caractere especial. `lcredit = -1` exige letra minúscula. `retry = 3` permite três tentativas ao definir uma senha.

**Por que foi usado:** senhas fracas são um vetor comum de comprometimento. Mesmo com firewall e SSH configurados, uma senha simples pode permitir acesso indevido se autenticação por senha estiver habilitada. Como a PoC manteve `PasswordAuthentication yes` por simplicidade operacional do acesso host-VM, a política de qualidade de senha ganha importância adicional. Esse controle aumenta a dificuldade de senhas triviais, reduzindo risco de ataques por tentativa e erro ou reutilização de senhas fracas.

---

## Bloco de expiração de senha em `/etc/login.defs`

```bash
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs
sudo sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/' /etc/login.defs
sudo sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE   7/' /etc/login.defs
```

**O que faz:** altera parâmetros de política de senha em `/etc/login.defs`. `PASS_MAX_DAYS` define validade máxima. `PASS_MIN_DAYS` define tempo mínimo antes de nova troca. `PASS_WARN_AGE` define quantos dias antes da expiração o usuário será avisado.

**Por que foi usado:** complementa a qualidade da senha com ciclo de vida de senha. `PASS_MAX_DAYS 90` reduz o tempo máximo de uso da mesma senha. `PASS_MIN_DAYS 1` evita trocas imediatas repetidas que poderiam burlar políticas de histórico em ambientes que usam esse controle. `PASS_WARN_AGE 7` melhora operação, avisando antes da expiração. Na PoC, isso demonstra que hardening de autenticação não é apenas “senha forte”, mas também política de validade e uso.

---

## `cat /etc/security/pwquality.conf | tee 02_hardening/pwquality_aplicado.txt`

**O que faz:** exibe a configuração de qualidade de senha aplicada.

**Por que foi usado:** gera evidência legível dos requisitos definidos. Esse comando é útil no print porque permite explicar exatamente quais critérios de senha foram configurados.

---

## `grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE" /etc/login.defs | tee 02_hardening/login_defs_aplicado.txt`

**O que faz:** filtra do arquivo `/etc/login.defs` apenas as linhas relacionadas à validade da senha.

**Por que foi usado:** valida de forma direta se os valores de expiração foram alterados. Em vez de mostrar o arquivo inteiro, o comando destaca apenas os parâmetros relevantes para a PoC.

---

# 9. Primeira rodada de hardening: auditoria com auditd

## `sudo systemctl enable --now auditd`

**O que faz:** habilita o `auditd` para iniciar automaticamente no boot e inicia o serviço imediatamente.

**Por que foi usado:** hardening também envolve rastreabilidade. Prevenir é importante, mas também é necessário registrar eventos relevantes para investigação. O `auditd` permite monitorar alterações em arquivos críticos, como usuários, grupos, senhas e privilégios. Ao habilitar o serviço, a PoC passa a demonstrar uma camada de detecção e auditoria, não apenas bloqueios preventivos.

---

## `sudo systemctl status auditd --no-pager | tee 02_hardening/auditd_status.txt`

**O que faz:** mostra o status do serviço auditd.

**Por que foi usado:** comprova que o mecanismo de auditoria está ativo. Se o serviço estivesse parado, as regras configuradas depois não teriam utilidade prática. Esse print mostra que a rastreabilidade está operacional.

---

## Bloco de regras auditd

```bash
cat << 'EOF' | sudo tee /etc/audit/rules.d/99-av4-hardening.rules
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k privilege
-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /etc/ssh/sshd_config.d/ -p wa -k ssh_config
EOF
```

**O que faz:** cria regras persistentes de auditoria. A opção `-w` define o arquivo ou diretório monitorado. `-p wa` monitora escrita e alteração de atributos. `-k` define uma chave para facilitar busca posterior nos logs.

**Por que foi usado:** os arquivos monitorados são críticos para segurança. `/etc/passwd` e `/etc/group` definem usuários e grupos. `/etc/shadow` guarda hashes de senha. `/etc/sudoers` controla privilégios administrativos. Arquivos de SSH controlam acesso remoto. Alterações nesses pontos podem ser legítimas, mas também podem indicar criação de usuário indevido, mudança de privilégios, tentativa de persistência ou enfraquecimento do acesso remoto. A PoC usa auditd para mostrar que hardening inclui detectar mudanças em ativos sensíveis.

---

## `sudo augenrules --load`

**O que faz:** carrega as regras persistentes de auditoria armazenadas em `/etc/audit/rules.d/`.

**Por que foi usado:** escrever o arquivo de regras não basta; é necessário carregar as regras para que o kernel passe a aplicá-las. Esse comando transforma a configuração salva em regra efetiva de auditoria.

---

## `sudo auditctl -l | tee 02_hardening/audit_rules_carregadas.txt`

**O que faz:** lista as regras de auditoria carregadas no momento.

**Por que foi usado:** valida se as regras realmente ficaram ativas. Na PoC, esse comando é uma evidência importante porque mostra os arquivos sensíveis monitorados. Ele confirma que a camada de rastreabilidade não ficou apenas escrita em arquivo, mas foi carregada no subsistema de auditoria.

---

# 10. Revisão de serviços e portas após primeira rodada

## `systemctl list-units --type=service --state=running | tee 02_hardening/servicos_ativos_pos_hardening.txt`

**O que faz:** lista serviços ativos depois da primeira rodada.

**Por que foi usado:** permite comparar a superfície de serviços antes e depois. A superfície de serviços pode não mudar muito se a VM já era mínima, e isso não é erro. O objetivo é demonstrar que a equipe verificou quais processos continuaram em execução e se havia algo desnecessário. Em ambiente real, esse tipo de comparação poderia revelar serviços como impressão, descoberta de rede, banco de dados ou web server rodando sem necessidade.

---

## `ss -tulpen | tee 02_hardening/portas_pos_hardening.txt`

**O que faz:** lista portas e sockets em escuta depois da primeira rodada.

**Por que foi usado:** permite comparar a superfície de rede após o hardening. Na PoC, as portas antes e depois permaneceram praticamente iguais porque o Ubuntu Server já estava com poucos serviços expostos. Isso está correto: o SSH precisa continuar aberto para administração, e serviços locais como `systemd-resolved` e DHCP são esperados. A melhoria principal não foi reduzir a quantidade de portas, mas ativar firewall, restringir SSH e validar a exposição de rede.

---

## Loop de revisão de serviços desnecessários

```bash
for svc in avahi-daemon cups bluetooth; do
  if systemctl list-unit-files | grep -q "^${svc}.service"; then
    echo "Desativando $svc"
    sudo systemctl disable --now "$svc" || true
  else
    echo "$svc não está instalado"
  fi
done | tee 02_hardening/revisao_servicos.txt
```

**O que faz:** verifica se `avahi-daemon`, `cups` e `bluetooth` existem no sistema. Se existirem, desativa imediatamente e impede inicialização no boot. Se não existirem, registra que não estão instalados.

**Por que foi usado:** esses serviços são comuns em estações de trabalho, mas geralmente não são necessários em um servidor Ubuntu usado para PoC de hardening. `avahi-daemon` faz descoberta de serviços na rede local, `cups` é voltado a impressão e `bluetooth` gerencia dispositivos Bluetooth. Em um servidor, manter serviços desnecessários aumenta superfície de ataque e ruído operacional. Mesmo que eles não estivessem instalados, a checagem mostra raciocínio técnico: validar e remover o que não pertence ao papel do servidor.

---

# 11. Auditoria após primeira rodada

## `sudo lynis audit system --quick --no-colors | tee 03_depois/lynis_tela_depois.txt`

**O que faz:** executa novamente a auditoria Lynis depois da primeira rodada de controles.

**Por que foi usado:** mede o impacto das primeiras alterações. A PoC segue um ciclo de segurança: medir, alterar, medir novamente. Esse comando mostra se UFW, SSH, sysctl, senha, auditd e revisão de serviços produziram melhora mensurável. Na execução da PoC, o hardening index subiu de 63 para 70, os warnings caíram de 1 para 0 e as suggestions caíram de 45 para 39.

---

## Bloco de resumo do Lynis após primeira rodada

```bash
grep -E '^hardening_index=|^warning\[\]=|^suggestion\[\]=' \
03_depois/lynis-report-depois.dat \
| tee 03_depois/resumo_lynis_depois.txt

echo "Hardening index depois da primeira rodada:"
grep '^hardening_index=' 03_depois/lynis-report-depois.dat

echo "Warnings depois da primeira rodada:"
grep -c '^warning\[\]=' 03_depois/lynis-report-depois.dat

echo "Suggestions depois da primeira rodada:"
grep -c '^suggestion\[\]=' 03_depois/lynis-report-depois.dat
```

**O que faz:** extrai as métricas principais da auditoria intermediária.

**Por que foi usado:** permite demonstrar numericamente o ganho da primeira rodada. Em apresentação, esse bloco é importante porque transforma a auditoria em uma comparação objetiva. A fala recomendada é que o hardening inicial corrigiu um warning e reduziu recomendações pendentes, mas ainda havia margem para uma segunda rodada.

---

# 12. Segunda rodada: ferramentas extras de segurança

## `sudo add-apt-repository universe -y`

**O que faz:** habilita o repositório Universe do Ubuntu.

**Por que foi usado:** algumas ferramentas complementares de segurança e auditoria podem estar nesse repositório. Habilitá-lo amplia a disponibilidade de pacotes para a segunda rodada. Isso mostra que a segunda etapa foi guiada por recomendações remanescentes e por defesa em profundidade, adicionando ferramentas além do conjunto básico.

---

## `sudo apt update`

**O que faz:** atualiza a lista de pacotes após habilitar o novo repositório.

**Por que foi usado:** depois de alterar repositórios, o `apt` precisa sincronizar novamente o índice de pacotes. Sem isso, a VM poderia não localizar os pacotes extras.

---

## Bloco de instalação de ferramentas extras

```bash
sudo apt install -y \
  fail2ban \
  debsums \
  apt-listchanges \
  libpam-tmpdir \
  apt-show-versions \
  acct \
  sysstat \
  aide \
  rkhunter \
  chkrootkit
```

**O que faz:** instala ferramentas complementares. `fail2ban` protege serviços contra tentativas repetidas de autenticação. `debsums` verifica integridade de arquivos instalados por pacotes. `apt-listchanges` mostra mudanças relevantes em atualizações. `libpam-tmpdir` melhora isolamento de diretórios temporários por usuário. `apt-show-versions` ajuda a verificar versões de pacotes. `acct` registra contabilidade de processos. `sysstat` coleta estatísticas do sistema. `aide` é voltado a integridade de arquivos. `rkhunter` e `chkrootkit` buscam indícios de rootkits.

**Por que foi usado:** amplia a PoC para além de configurações básicas. Hardening não é apenas bloquear porta; também envolve detecção, integridade, rastreabilidade e capacidade de verificar mudanças. O Fail2ban reforça SSH, o debsums ajuda a verificar alterações em arquivos de pacotes, o acct aumenta rastreabilidade de processos, o sysstat apoia observabilidade, e ferramentas como AIDE/rkhunter/chkrootkit mostram preocupação com integridade e comprometimento. Essas ferramentas explicam por que a segunda rodada aumentou o hardening index de forma mais expressiva.

---

## Loop de verificação de pacotes instalados

```bash
for pkg in fail2ban debsums apt-listchanges libpam-tmpdir apt-show-versions acct sysstat aide rkhunter chkrootkit; do
  if dpkg -s "$pkg" >/dev/null 2>&1; then
    echo "$pkg: instalado"
  else
    echo "$pkg: não instalado"
  fi
done | tee 05_hardening_extra/pacotes_extra_instalados.txt
```

**O que faz:** verifica se cada pacote da segunda rodada está instalado. `dpkg -s` consulta o status do pacote, e a estrutura `if` imprime se ele foi encontrado ou não.

**Por que foi usado:** gera evidência objetiva de instalação das ferramentas. Em vez de apenas presumir que o `apt install` funcionou, o comando confirma pacote por pacote. Isso é útil na apresentação porque permite mostrar que a segunda rodada foi efetivamente aplicada.

---

# 13. Segunda rodada: Fail2ban para SSH

## Bloco de configuração `/etc/fail2ban/jail.local`

```bash
cat << 'EOF' | sudo tee /etc/fail2ban/jail.local
[DEFAULT]
bantime = 10m
findtime = 10m
maxretry = 3

[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
backend = systemd
EOF
```

**O que faz:** cria a configuração local do Fail2ban. `bantime = 10m` define banimento por 10 minutos. `findtime = 10m` define a janela de observação. `maxretry = 3` define quantas falhas são permitidas antes do banimento. A seção `[sshd]` ativa a proteção do SSH e usa logs do systemd.

**Por que foi usado:** complementa o hardening do SSH. Mesmo com `MaxAuthTries` e limites de sessão, um atacante pode tentar múltiplas conexões ao longo do tempo. O Fail2ban monitora falhas de autenticação e bloqueia temporariamente IPs que excedem a política definida. Isso reduz risco de força bruta e ruído de tentativas repetidas. Como a PoC manteve autenticação por senha habilitada para simplificar o acesso host-VM, o Fail2ban se torna uma proteção adicional importante.

---

## `sudo systemctl enable --now fail2ban`

**O que faz:** habilita o Fail2ban para iniciar no boot e inicia o serviço imediatamente.

**Por que foi usado:** garante que a proteção não dependa de inicialização manual. Em hardening, um controle só é útil se permanecer ativo após reinicialização.

---

## `sudo systemctl restart fail2ban`

**O que faz:** reinicia o serviço para carregar a configuração recém-criada.

**Por que foi usado:** assegura que o `jail.local` foi lido e aplicado. Sem restart, o serviço poderia continuar usando configuração anterior.

---

## `sudo systemctl status fail2ban --no-pager | tee 05_hardening_extra/fail2ban_service_status.txt`

**O que faz:** mostra o status do serviço Fail2ban.

**Por que foi usado:** comprova que a ferramenta está ativa. Na apresentação, esse print mostra que o controle está rodando como serviço do sistema.

---

## `sudo fail2ban-client status | tee 05_hardening_extra/fail2ban_status.txt`

**O que faz:** mostra o status geral do Fail2ban e as jails ativas.

**Por que foi usado:** valida se há jails carregadas. O serviço pode estar ativo, mas sem jail protegendo o SSH. Esse comando confirma que o Fail2ban está operacional do ponto de vista da aplicação.

---

## `sudo fail2ban-client status sshd | tee 05_hardening_extra/fail2ban_sshd_status.txt`

**O que faz:** mostra detalhes da jail `sshd`, como número de falhas, IPs atualmente banidos e mecanismo de log usado.

**Por que foi usado:** comprova que o SSH está sendo monitorado especificamente. Mesmo que não existam IPs banidos no momento, a jail ativa já mostra que o controle está preparado para reagir a falhas de autenticação.

---

# 14. Segunda rodada: reforço extra no SSH

## Bloco final de configuração SSH

```bash
cat << 'EOF' | sudo tee /etc/ssh/sshd_config.d/99-av4-hardening.conf
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
LogLevel VERBOSE
MaxSessions 2
TCPKeepAlive no
EOF
```

**O que faz:** atualiza o arquivo de hardening do SSH com controles adicionais. Além dos controles anteriores, adiciona `AllowAgentForwarding no`, `LogLevel VERBOSE`, `MaxSessions 2` e `TCPKeepAlive no`.

**Por que foi usado:** endurece ainda mais o principal serviço exposto da VM. `AllowAgentForwarding no` impede encaminhamento do agente SSH, reduzindo risco de uso indevido de credenciais encaminhadas. `LogLevel VERBOSE` aumenta detalhes nos logs de autenticação, melhorando rastreabilidade. `MaxSessions 2` limita sessões por conexão, reduzindo abuso de multiplexação. `TCPKeepAlive no` evita dependência de keepalive TCP tradicional, que pode ser menos confiável em alguns cenários; a sessão continua sendo controlada pelos parâmetros `ClientAlive`. Esse reforço mostra que a segunda rodada não apenas instalou ferramentas, mas também revisou configurações críticas.

---

## `sudo /usr/sbin/sshd -t`

**O que faz:** valida a configuração final do SSH.

**Por que foi usado:** repete o cuidado operacional antes de reiniciar um serviço crítico. Como o SSH é o meio de acesso à VM, validar antes de aplicar evita perda de conexão por erro de sintaxe.

---

## `sudo systemctl restart ssh`

**O que faz:** reinicia o SSH para aplicar a configuração final.

**Por que foi usado:** faz o serviço carregar os controles extras configurados.

---

## `sudo systemctl status ssh --no-pager | tee 05_hardening_extra/ssh_status_extra.txt`

**O que faz:** mostra o status do SSH após o reforço extra.

**Por que foi usado:** valida que o SSH continuou ativo depois da segunda rodada. Isso demonstra equilíbrio entre segurança e disponibilidade.

---

## `cat /etc/ssh/sshd_config.d/99-av4-hardening.conf | tee 05_hardening_extra/ssh_hardening_extra.txt`

**O que faz:** exibe o arquivo final de configuração SSH.

**Por que foi usado:** gera evidência dos parâmetros finais aplicados. Esse print é útil para mostrar visualmente quais diretivas foram usadas, antes da validação efetiva com `sshd -T`.

---

# 15. Segunda rodada: umask restritivo

## Bloco de configuração de `UMASK 027`

```bash
if grep -q '^UMASK' /etc/login.defs; then
  sudo sed -i 's/^UMASK.*/UMASK           027/' /etc/login.defs
else
  echo 'UMASK           027' | sudo tee -a /etc/login.defs
fi

grep '^UMASK' /etc/login.defs | tee 05_hardening_extra/umask_login_defs.txt
```

**O que faz:** verifica se já existe configuração `UMASK` em `/etc/login.defs`. Se existir, substitui por `027`; se não existir, adiciona. Depois, exibe o valor final.

**Por que foi usado:** controla permissões padrão de novos arquivos e diretórios criados por usuários. Umask define quais permissões serão removidas no momento da criação. Com `027`, a tendência é que novos arquivos não fiquem acessíveis para “outros” e tenham permissões mais restritas para grupo. Isso reduz risco de exposição acidental de arquivos sensíveis. Em servidores multiusuário ou ambientes com logs, scripts e arquivos administrativos, permissões padrão muito abertas podem vazar informações.

---

## Bloco `/etc/profile.d/99-av4-umask.sh`

```bash
cat << 'EOF' | sudo tee /etc/profile.d/99-av4-umask.sh
umask 027
EOF

sudo chmod 644 /etc/profile.d/99-av4-umask.sh

cat /etc/profile.d/99-av4-umask.sh | tee 05_hardening_extra/umask_profiled.txt
```

**O que faz:** cria um script de profile que aplica `umask 027` em novas sessões de shell. Depois ajusta a permissão do script e exibe o conteúdo.

**Por que foi usado:** reforça a aplicação do umask em sessões interativas. A configuração em `login.defs` cobre parte dos cenários, mas scripts em `/etc/profile.d/` ajudam a aplicar a política em shells de login. O `chmod 644` garante que o arquivo seja legível, mas não editável por usuários comuns. Esse controle mostra preocupação com permissões padrão, que muitas vezes são esquecidas em hardening.

---

# 16. Segunda rodada: restrição de core dumps

## Bloco de limites de core dump

```bash
cat << 'EOF' | sudo tee /etc/security/limits.d/99-av4-disable-coredumps.conf
* hard core 0
* soft core 0
EOF
```

**O que faz:** define limites de core dump para todos os usuários. `soft core 0` define o limite inicial, e `hard core 0` impede que o limite seja aumentado acima de zero.

**Por que foi usado:** core dumps são arquivos gerados quando processos falham, contendo imagem da memória do processo. Essa memória pode incluir senhas, tokens, chaves, variáveis de ambiente, dados de sessão ou informações internas da aplicação. Em ambientes de produção, permitir core dumps sem controle pode gerar vazamento local de informações sensíveis. Na PoC, esse comando reduz o risco de exposição por arquivos de dump.

---

## Bloco `fs.suid_dumpable = 0`

```bash
cat << 'EOF' | sudo tee /etc/sysctl.d/99-av4-coredump-hardening.conf
fs.suid_dumpable = 0
EOF

sudo sysctl --system | tee 05_hardening_extra/sysctl_coredump_aplicacao.txt

sysctl fs.suid_dumpable | tee 05_hardening_extra/coredump_sysctl.txt
```

**O que faz:** configura e aplica o parâmetro `fs.suid_dumpable = 0`, depois verifica o valor efetivo.

**Por que foi usado:** processos SUID podem executar com privilégios maiores que os do usuário que os iniciou. Se um programa desse tipo gerar core dump, o arquivo pode conter informações sensíveis de um processo privilegiado. Desabilitar dumps de programas SUID reduz risco de vazamento de memória privilegiada. A verificação com `sysctl fs.suid_dumpable` é necessária para confirmar se o parâmetro foi carregado.

---

# 17. Segunda rodada: bloqueio de protocolos de rede raros

## Bloco `/etc/modprobe.d/99-av4-disable-rare-network-protocols.conf`

```bash
cat << 'EOF' | sudo tee /etc/modprobe.d/99-av4-disable-rare-network-protocols.conf
install dccp /bin/false
install sctp /bin/false
install rds /bin/false
install tipc /bin/false

blacklist dccp
blacklist sctp
blacklist rds
blacklist tipc
EOF

cat /etc/modprobe.d/99-av4-disable-rare-network-protocols.conf | tee 05_hardening_extra/protocolos_bloqueados.txt
```

**O que faz:** impede o carregamento de módulos de protocolos de rede pouco usados. A diretiva `install módulo /bin/false` faz a tentativa de carregar o módulo falhar. A diretiva `blacklist` impede carregamento automático.

**Por que foi usado:** reduz superfície de ataque no kernel. Protocolos como DCCP, SCTP, RDS e TIPC têm usos específicos e normalmente não são necessários em uma VM Ubuntu Server simples. Se um protocolo não é usado, mantê-lo disponível aumenta complexidade e potencial exposição sem benefício. Bloquear módulos não necessários segue o princípio de reduzir funcionalidades disponíveis ao mínimo necessário para o papel do servidor.

---

# 18. Segunda rodada: bloqueio de armazenamento USB

## Bloco `/etc/modprobe.d/99-av4-disable-usb-storage.conf`

```bash
cat << 'EOF' | sudo tee /etc/modprobe.d/99-av4-disable-usb-storage.conf
install usb-storage /bin/false
blacklist usb-storage
EOF

cat /etc/modprobe.d/99-av4-disable-usb-storage.conf | tee 05_hardening_extra/usb_storage_bloqueado.txt
```

**O que faz:** impede o carregamento do módulo `usb-storage`, usado para dispositivos de armazenamento USB.

**Por que foi usado:** reduz risco de uso não autorizado de mídia removível. Em servidores, dispositivos USB de armazenamento podem ser fonte de vazamento de dados, introdução de arquivos maliciosos ou cópia não controlada de informações. Mesmo em VM, o controle é demonstrativo e alinhado a hardening de servidores: se o recurso não é necessário, ele deve ser bloqueado ou restringido.

---

# 19. Segunda rodada: banners legais

## Blocos `/etc/issue` e `/etc/issue.net`

```bash
cat << 'EOF' | sudo tee /etc/issue
Acesso restrito. Uso autorizado apenas para fins administrativos e acadêmicos. Atividades podem ser registradas.
EOF

cat << 'EOF' | sudo tee /etc/issue.net
Acesso restrito. Uso autorizado apenas para fins administrativos e acadêmicos. Atividades podem ser registradas.
EOF

cat /etc/issue | tee 05_hardening_extra/banner_issue.txt
cat /etc/issue.net | tee 05_hardening_extra/banner_issue_net.txt
```

**O que faz:** define mensagens de aviso para login local e, dependendo da configuração do serviço, login remoto.

**Por que foi usado:** banners legais não bloqueiam tecnicamente um ataque, mas deixam explícito que o acesso é restrito e monitorado. Em ambientes corporativos, esse tipo de aviso apoia políticas internas, auditoria e conformidade. Na PoC, ele mostra que hardening também envolve comunicação de uso autorizado e registro de atividades, não apenas parâmetros técnicos.

---

# 20. Segunda rodada: contabilidade e estatísticas do sistema

## Bloco `acct`

```bash
sudo systemctl enable --now acct
sudo systemctl status acct --no-pager | tee 05_hardening_extra/acct_status.txt
```

**O que faz:** habilita e inicia contabilidade de processos, depois mostra o status do serviço.

**Por que foi usado:** aumenta rastreabilidade de execução de processos. Em segurança, é útil saber quais comandos e processos foram executados, especialmente em investigação posterior. O `acct` complementa o auditd: enquanto auditd monitora eventos definidos por regras, contabilidade de processos ajuda a registrar atividade geral de execução.

---

## Bloco `sysstat`

```bash
sudo sed -i 's/^ENABLED=.*/ENABLED="true"/' /etc/default/sysstat
sudo systemctl enable --now sysstat
sudo systemctl restart sysstat

cat /etc/default/sysstat | tee 05_hardening_extra/sysstat_default.txt
sudo systemctl status sysstat --no-pager | tee 05_hardening_extra/sysstat_status.txt
```

**O que faz:** habilita coleta do sysstat, inicia o serviço, reinicia para aplicar configuração e mostra configuração/status.

**Por que foi usado:** adiciona observabilidade ao sistema. Hardening pode impactar comportamento, carga e serviços; coletar estatísticas ajuda a acompanhar CPU, disco, memória e I/O ao longo do tempo. Em segurança operacional, observabilidade facilita perceber comportamento anormal e avaliar impacto dos controles aplicados.

---

# 21. Segunda rodada: AIDE como limitação documentada

## `echo "A inicialização do AIDE foi considerada..." | tee 05_hardening_extra/aide_observacao.txt`

**O que faz:** registra em arquivo que a inicialização completa do AIDE foi considerada, mas omitida da execução final da PoC por tempo elevado de varredura em ambiente virtualizado.

**Por que foi usado:** documenta uma limitação operacional em vez de simplesmente ignorá-la. AIDE cria uma base de integridade de arquivos para comparação futura, mas sua inicialização pode demorar. Em uma apresentação acadêmica, reconhecer essa limitação mostra maturidade técnica: o controle é relevante, mas a PoC delimitou escopo para caber no tempo e no ambiente.

---

# 22. Segunda rodada: rounds de hash de senha

## Bloco de `SHA_CRYPT_MIN_ROUNDS` e `SHA_CRYPT_MAX_ROUNDS`

```bash
if grep -q '^SHA_CRYPT_MIN_ROUNDS' /etc/login.defs; then
  sudo sed -i 's/^SHA_CRYPT_MIN_ROUNDS.*/SHA_CRYPT_MIN_ROUNDS 5000/' /etc/login.defs
else
  echo 'SHA_CRYPT_MIN_ROUNDS 5000' | sudo tee -a /etc/login.defs
fi

if grep -q '^SHA_CRYPT_MAX_ROUNDS' /etc/login.defs; then
  sudo sed -i 's/^SHA_CRYPT_MAX_ROUNDS.*/SHA_CRYPT_MAX_ROUNDS 10000/' /etc/login.defs
else
  echo 'SHA_CRYPT_MAX_ROUNDS 10000' | sudo tee -a /etc/login.defs
fi

grep -E '^SHA_CRYPT_MIN_ROUNDS|^SHA_CRYPT_MAX_ROUNDS' /etc/login.defs | tee 05_hardening_extra/password_hash_rounds.txt
```

**O que faz:** configura número mínimo e máximo de rounds para hashes SHA crypt usados em senhas, depois exibe os valores aplicados.

**Por que foi usado:** aumenta o custo computacional de ataques offline contra hashes de senha. Se um atacante obtiver hashes, por exemplo a partir de `/etc/shadow`, ele pode tentar quebrá-los fora do sistema. Aumentar rounds faz cada tentativa custar mais processamento, dificultando ataques em massa. O limite máximo evita custo excessivo que poderia afetar desempenho. Esse controle complementa a política de senha: além de exigir senha mais forte, torna o hash mais caro de atacar.

---

# 23. Auditoria final após segunda rodada

## `sudo lynis audit system --quick --no-colors | tee 06_depois_extra/lynis_tela_depois_extra.txt`

**O que faz:** executa a auditoria final do Lynis depois da segunda rodada de hardening.

**Por que foi usado:** mede o resultado final da PoC. Essa auditoria mostra o impacto acumulado dos controles básicos e extras. Na execução realizada, o hardening index chegou a 81, warnings ficaram em 0 e suggestions caíram para 24. Esse é o principal resultado quantitativo do trabalho.

---

## Bloco de resumo do Lynis final

```bash
grep -E '^hardening_index=|^warning\[\]=|^suggestion\[\]=' \
06_depois_extra/lynis-report-depois-extra.dat \
| tee 06_depois_extra/resumo_lynis_depois_extra.txt

echo "Hardening index depois da segunda rodada:"
grep '^hardening_index=' 06_depois_extra/lynis-report-depois-extra.dat

echo "Warnings depois da segunda rodada:"
grep -c '^warning\[\]=' 06_depois_extra/lynis-report-depois-extra.dat

echo "Suggestions depois da segunda rodada:"
grep -c '^suggestion\[\]=' 06_depois_extra/lynis-report-depois-extra.dat
```

**O que faz:** extrai as métricas principais da auditoria final.

**Por que foi usado:** cria evidência objetiva do resultado final. O comando permite mostrar a evolução do sistema em números, sem depender apenas de interpretação subjetiva. A fala para apresentação é que a PoC reduziu warnings de 1 para 0, reduziu suggestions de 45 para 24 e aumentou o hardening index de 63 para 81.

---

# 24. Comparação final das auditorias

## Bloco de variáveis e tabela comparativa

```bash
ANTES_INDEX=$(grep '^hardening_index=' 01_antes/lynis-report-antes.dat | cut -d= -f2)
DEPOIS_INDEX=$(grep '^hardening_index=' 03_depois/lynis-report-depois.dat | cut -d= -f2)
EXTRA_INDEX=$(grep '^hardening_index=' 06_depois_extra/lynis-report-depois-extra.dat | cut -d= -f2)

ANTES_WARN=$(grep -c '^warning\[\]=' 01_antes/lynis-report-antes.dat)
DEPOIS_WARN=$(grep -c '^warning\[\]=' 03_depois/lynis-report-depois.dat)
EXTRA_WARN=$(grep -c '^warning\[\]=' 06_depois_extra/lynis-report-depois-extra.dat)

ANTES_SUG=$(grep -c '^suggestion\[\]=' 01_antes/lynis-report-antes.dat)
DEPOIS_SUG=$(grep -c '^suggestion\[\]=' 03_depois/lynis-report-depois.dat)
EXTRA_SUG=$(grep -c '^suggestion\[\]=' 06_depois_extra/lynis-report-depois-extra.dat)

{
  echo "Métrica,Antes,Depois 1,Depois 2"
  echo "Hardening index,$ANTES_INDEX,$DEPOIS_INDEX,$EXTRA_INDEX"
  echo "Warnings,$ANTES_WARN,$DEPOIS_WARN,$EXTRA_WARN"
  echo "Suggestions,$ANTES_SUG,$DEPOIS_SUG,$EXTRA_SUG"
} | tee 06_depois_extra/comparativo_lynis_3_etapas.csv

column -s, -t 06_depois_extra/comparativo_lynis_3_etapas.csv | tee 06_depois_extra/comparativo_lynis_3_etapas_tabela.txt

cat 06_depois_extra/comparativo_lynis_3_etapas_tabela.txt
```

**O que faz:** extrai automaticamente as métricas dos três relatórios do Lynis, monta um CSV e exibe uma tabela alinhada no terminal.

**Por que foi usado:** consolida a evidência principal da PoC. Em vez de mostrar três relatórios separados e esperar que o público compare manualmente, o script apresenta a evolução em uma tabela simples: antes, depois da primeira rodada e depois da segunda rodada. Isso reforça a reprodutibilidade do trabalho, porque a tabela é gerada a partir dos próprios relatórios salvos. O resultado final demonstra melhoria mensurável: hardening index 63 -> 70 -> 81, warnings 1 -> 0 -> 0 e suggestions 45 -> 39 -> 24.

---

# 25. Validação final dos controles aplicados

## `sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|maxauthtries|logingracetime|x11forwarding|allowtcpforwarding|allowagentforwarding|clientaliveinterval|clientalivecountmax|loglevel|maxsessions|tcpkeepalive' | tee 07_validacao_final/ssh_config_efetiva.txt`

**O que faz:** exibe a configuração efetiva do SSH após o OpenSSH processar arquivos principais, arquivos incluídos e valores padrão. O `grep` filtra apenas as diretivas relevantes para a PoC.

**Por que foi usado:** valida o estado real do SSH, não apenas o conteúdo do arquivo criado. Isso é importante porque o OpenSSH pode combinar configurações de múltiplos locais e aplicar padrões quando uma opção não está definida explicitamente. A saída efetiva permite comprovar que `PermitRootLogin no`, `MaxAuthTries 3`, `LoginGraceTime 30`, `X11Forwarding no`, `AllowTcpForwarding no`, `AllowAgentForwarding no`, `LogLevel VERBOSE`, `MaxSessions 2` e outros controles foram realmente interpretados pelo serviço. A saída também mostra `PasswordAuthentication yes`, que foi mantido por decisão de escopo: a PoC usa SSH do host para a VM via NAT, e configurar chaves SSH seria uma etapa adicional. Em ambiente real, usar chaves e desabilitar senha seria um reforço recomendado.

---

## `sudo ufw status verbose | tee 07_validacao_final/ufw_status_final.txt`

**O que faz:** mostra o status final detalhado do UFW.

**Por que foi usado:** confirma a política final de firewall. A validação deve mostrar o UFW ativo, entrada negada por padrão e saída permitida por padrão, com OpenSSH liberado. Esse comando fecha o ciclo do firewall: primeiro foi coletado o estado inicial, depois o UFW foi configurado, e agora a configuração final é confirmada. Ele também ajuda a explicar por que as portas podem continuar iguais no `ss`, mas o risco muda: o firewall controla o tráfego permitido.

---

## `sudo ufw status numbered | tee 07_validacao_final/ufw_regras_numeradas.txt`

**O que faz:** mostra as regras do UFW com numeração.

**Por que foi usado:** facilita auditoria e manutenção das regras. Em ambientes reais, regras numeradas ajudam a remover ou alterar entradas específicas com menor risco de erro. Na PoC, isso demonstra que a regra principal é explícita e verificável.

---

## `ss -tulpen | tee 07_validacao_final/portas_finais.txt`

**O que faz:** lista portas finais em escuta.

**Por que foi usado:** valida a superfície de rede ao final da PoC. O esperado é que o SSH continue aparecendo, porque ele é necessário para administração. Serviços locais de DNS/resolução e DHCP também podem aparecer em uma VM com systemd e NAT. Se o resultado for semelhante ao inicial, isso não invalida o hardening: significa que a VM já tinha pouca exposição de rede e que o ganho principal foi controlar o acesso com firewall e endurecer o serviço SSH.

---

## Bloco de validação sysctl final

```bash
sudo sysctl net.ipv4.ip_forward \
net.ipv4.tcp_syncookies \
net.ipv4.conf.all.accept_redirects \
net.ipv4.conf.default.accept_redirects \
net.ipv4.conf.all.accept_source_route \
net.ipv4.conf.default.accept_source_route \
net.ipv4.conf.all.rp_filter \
net.ipv4.conf.default.rp_filter \
kernel.randomize_va_space \
fs.protected_hardlinks \
fs.protected_symlinks \
fs.suid_dumpable \
| tee 07_validacao_final/sysctl_final.txt
```

**O que faz:** consulta os parâmetros finais de kernel e rede mais relevantes para a PoC.

**Por que foi usado:** confirma se os controles sysctl continuam efetivos no final. Essa etapa é importante porque configurações podem ser sobrescritas por outros arquivos ou serviços. O comando valida roteamento IPv4, SYN cookies, redirects, source routing, reverse path filtering, ASLR, proteção contra hardlinks/symlinks e core dumps SUID. Para apresentação, esse print deve ser usado para mostrar que os controles foram verificados depois de todas as rodadas, não apenas no momento em que foram criados.

---

## `sudo systemctl status auditd --no-pager | tee 07_validacao_final/auditd_status_final.txt`

**O que faz:** mostra se o serviço auditd continua ativo na validação final.

**Por que foi usado:** confirma que a camada de auditoria permaneceu operacional após as mudanças. Um controle de auditoria só tem valor se o serviço estiver rodando.

---

## `sudo auditctl -s | tee 07_validacao_final/auditd_estado_final.txt`

**O que faz:** mostra o estado do subsistema de auditoria do kernel, incluindo se está habilitado e se houve eventos perdidos.

**Por que foi usado:** vai além do status do serviço. O auditd pode estar rodando, mas é importante verificar também o estado do subsistema de auditoria. Eventos perdidos podem indicar sobrecarga ou falha de registro. Esse comando reforça a confiabilidade da evidência.

---

## `sudo auditctl -l | tee 07_validacao_final/auditd_regras_finais.txt`

**O que faz:** lista as regras de auditoria ativas no final.

**Por que foi usado:** comprova que as regras para arquivos sensíveis continuam carregadas. Essa validação mostra que alterações em identidade, grupos, senhas, sudoers e SSH permanecem monitoradas.

---

## `sudo systemctl status fail2ban --no-pager | tee 07_validacao_final/fail2ban_status_servico_final.txt`

**O que faz:** mostra o status final do serviço Fail2ban.

**Por que foi usado:** confirma que o serviço de proteção contra tentativas repetidas continua ativo.

---

## `sudo fail2ban-client status | tee 07_validacao_final/fail2ban_status_final.txt`

**O que faz:** mostra o status geral do Fail2ban e jails carregadas.

**Por que foi usado:** verifica se o serviço está operacional no nível da aplicação, não apenas como processo systemd.

---

## `sudo fail2ban-client status sshd | tee 07_validacao_final/fail2ban_sshd_final.txt`

**O que faz:** mostra o estado final da jail SSH.

**Por que foi usado:** comprova que o SSH está sendo monitorado. Na PoC, mesmo sem IPs banidos no momento, a jail ativa demonstra que falhas de autenticação serão acompanhadas e poderão gerar bloqueios conforme a política configurada.

---

## `grep -n 'pam_pwquality' /etc/pam.d/common-password | tee 07_validacao_final/pam_pwquality_verificacao.txt || true`

**O que faz:** procura no arquivo PAM de senhas a linha que carrega `pam_pwquality`. A opção `-n` mostra o número da linha. O `|| true` evita que a execução seja considerada falha caso a string não seja encontrada.

**Por que foi usado:** valida se a política de qualidade de senha está conectada ao fluxo real de alteração de senha. Editar `/etc/security/pwquality.conf` define critérios, mas o PAM precisa chamar o módulo para que eles sejam aplicados. Esse comando fecha essa lacuna: ele mostra se o sistema está usando `pam_pwquality` no arquivo `common-password`. Em hardening, isso é fundamental porque configuração isolada sem integração pode gerar falsa sensação de controle.

---

## `sudo debsums -s | tee 07_validacao_final/debsums_resultado.txt`

**O que faz:** verifica checksums de arquivos instalados por pacotes Debian/Ubuntu. A opção `-s` usa modo silencioso, exibindo apenas problemas encontrados.

**Por que foi usado:** valida integridade básica dos arquivos de pacotes instalados. Se não houver saída, isso normalmente indica que o `debsums` não encontrou divergências nos arquivos verificados. Esse controle não substitui AIDE nem investigação forense, mas fornece uma checagem rápida contra alterações inesperadas em arquivos gerenciados por pacotes.

---

# Encerramento da PoC

Resultado principal demonstrado:

```text
Hardening index: 63 -> 70 -> 81
Warnings:         1 -> 0  -> 0
Suggestions:     45 -> 39 -> 24
```

A conclusão para apresentação é que a PoC demonstrou um ciclo técnico completo: medição inicial, aplicação de controles, auditoria intermediária, segunda rodada de reforço, auditoria final e validação efetiva dos principais controles. A superfície de rede da VM já era pequena, por isso as portas não mudaram muito. O ganho principal ocorreu na política de firewall, endurecimento do SSH, parâmetros de kernel e rede, auditoria de arquivos sensíveis, proteção contra tentativas repetidas de login e verificação final da configuração efetiva.
