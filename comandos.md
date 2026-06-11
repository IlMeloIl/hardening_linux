# AV4 - PoC Hardening Linux com CIS Benchmark + Lynis — comandos explicados

## Ambiente utilizado

- **Host:** Ubuntu 24.04.4 LTS
- **VM:** Ubuntu Server 24.04.4 LTS
- **Ferramentas principais:** VirtualBox, Lynis, UFW, auditd e Fail2ban

# 1. HOST - Instalar ferramentas para virtualização

```bash
sudo apt update
sudo apt install -y software-properties-common
```

**Explicação:**

- `sudo apt update`: atualiza a lista local de pacotes disponíveis nos repositórios configurados no host. Não instala pacotes; apenas sincroniza os índices.
- `sudo apt install -y software-properties-common`: instala ferramentas auxiliares para gerenciamento de repositórios e propriedades de software. O parâmetro `-y` confirma automaticamente a instalação.

```bash
sudo apt install -y virtualbox virtualbox-dkms dkms build-essential linux-headers-$(uname -r)
```

**Explicação:**

- `virtualbox`: instala o VirtualBox, usado para criar e executar a VM da PoC.
- `virtualbox-dkms`: instala os módulos do VirtualBox integrados ao DKMS, permitindo recompilação automática dos módulos quando o kernel é atualizado.
- `dkms`: gerencia recompilação de módulos externos do kernel.
- `build-essential`: instala compilador e ferramentas básicas de compilação.
- `linux-headers-$(uname -r)`: instala os headers do kernel atualmente em execução. O trecho `$(uname -r)` retorna a versão do kernel do host.

```bash
virtualbox
```

**Explicação:**

- `virtualbox`: abre a interface gráfica do VirtualBox para criação e configuração da máquina virtual.

---

# 2. HOST - Baixar ISO do Ubuntu Server 24.04.4 LTS

```text
https://ubuntu.com/download/server/thank-you?version=24.04.4&architecture=amd64&lts=true
```

**Explicação:**

- Esse link foi usado para baixar a ISO oficial do Ubuntu Server 24.04.4 LTS, que serviu como sistema operacional da VM.
- A escolha do Ubuntu Server evita componentes gráficos desnecessários e aproxima a PoC de um cenário real de servidor.

---

# 3. VIRTUALBOX - Criar a VM

Pela interface gráfica do VirtualBox:

```text
New ->
Name: AV4-Hardening-Ubuntu ->
Type: Linux ->
Version: Ubuntu (64-bit) ->
ISO: ubuntu-24.04.4-live-server-amd64.iso ->
RAM: 4096 MB ->
CPU: 2 ->
Disk: 25 GB ->
Network: NAT ->
Marcar "Skip Unattended Installation"
```

**Explicação:**

- `Name`: define o nome da VM no VirtualBox.
- `Type` e `Version`: indicam que a VM executará Linux Ubuntu 64-bit.
- `ISO`: aponta para a imagem de instalação do Ubuntu Server.
- `RAM: 4096 MB`: reserva 4 GB de memória para a VM.
- `CPU: 2`: reserva 2 núcleos virtuais para a VM.
- `Disk: 25 GB`: cria um disco virtual suficiente para sistema, ferramentas e evidências.
- `Network: NAT`: permite que a VM acesse a internet usando a rede do host, sem expor diretamente a VM à rede local.
- `Skip Unattended Installation`: evita instalação automática, permitindo controlar manualmente usuário, hostname e instalação do OpenSSH Server.

---

# 4. VIRTUALBOX - Configurar redirecionamento SSH

Com a VM desligada, configurar pela interface gráfica:

```text
Settings ->
Network ->
Adapter 1: NAT ->
Advanced ->
Port Forwarding ->

Adicionar:
Name: SSH
Protocol: TCP
Host IP: 127.0.0.1
Host Port: 2222
Guest IP: deixar vazio
Guest Port: 22
```

**Explicação:**

- Essa regra permite acessar o SSH da VM pelo host.
- `Host IP: 127.0.0.1`: limita o acesso ao próprio host, sem abrir a porta para a rede externa.
- `Host Port: 2222`: define a porta usada no host.
- `Guest Port: 22`: aponta para a porta padrão do SSH dentro da VM.
- Com essa configuração, o acesso fica: `ssh -p 2222 aluno@127.0.0.1`.

---

# 5. VM - Instalar Ubuntu Server

Durante a instalação:

```text
Username: aluno
Hostname: av4-hardening
OpenSSH Server: instalar
Featured Server Snaps: não selecionar nada
Após finalizar: Reboot Now
```

**Explicação:**

- `Username: aluno`: cria o usuário administrativo da PoC.
- `Hostname: av4-hardening`: define o nome da máquina na rede/sistema.
- `OpenSSH Server: instalar`: instala o serviço SSH, necessário para administrar a VM a partir do host.
- `Featured Server Snaps: não selecionar nada`: evita instalar serviços extras que poderiam aumentar a superfície de ataque.
- `Reboot Now`: reinicia a VM após a instalação.

---

# 6. HOST - Acessar a VM por SSH

```bash
ssh -p 2222 aluno@127.0.0.1
```

**Explicação:**

- `ssh`: inicia uma conexão remota segura.
- `-p 2222`: usa a porta 2222 no host, configurada no redirecionamento NAT.
- `aluno@127.0.0.1`: conecta como usuário `aluno` ao endereço local do host, que encaminha o tráfego para a VM.
- A partir desse ponto, os comandos passam a ser executados dentro da VM, salvo quando indicado o contrário.

---

# A PARTIR DAQUI, TODOS OS COMANDOS SÃO DENTRO DA VM

---

# 7. VM - Atualizar sistema e instalar ferramentas básicas

```bash
sudo apt update
sudo apt full-upgrade -y

sudo apt install -y lynis ufw auditd audispd-plugins libpam-pwquality openssh-server curl wget net-tools vim

sudo reboot
```

**Explicação:**

- `sudo apt update`: atualiza os índices de pacotes da VM.
- `sudo apt full-upgrade -y`: aplica atualizações disponíveis, incluindo atualizações que podem instalar/remover dependências quando necessário.
- `lynis`: ferramenta de auditoria de segurança usada para medir o estado antes e depois do hardening.
- `ufw`: firewall simplificado usado para controlar conexões de entrada e saída.
- `auditd`: serviço de auditoria do Linux, usado para registrar eventos relevantes.
- `audispd-plugins`: plugins auxiliares para despacho/processamento de eventos de auditoria.
- `libpam-pwquality`: módulo PAM para política de qualidade de senha.
- `openssh-server`: servidor SSH para administração remota.
- `curl` e `wget`: ferramentas para download e requisições HTTP/HTTPS.
- `net-tools`: fornece ferramentas clássicas de rede.
- `vim`: editor de texto para ajustes manuais.
- `sudo reboot`: reinicia a VM para garantir que atualizações e serviços sejam carregados corretamente.

Depois do reboot, conectar novamente pelo host:

```bash
ssh -p 2222 aluno@127.0.0.1
```

**Explicação:**

- Reconecta à VM após a reinicialização.

---

# 8. VM - Criar estrutura de evidências

```bash
mkdir -p ~/av4-hardening/{00_ambiente,01_antes,02_hardening,03_depois,04_comparacao,05_hardening_extra,06_depois_extra}
cd ~/av4-hardening
```

**Explicação:**

- `mkdir -p`: cria diretórios e não gera erro se eles já existirem.
- `~/av4-hardening`: diretório principal da PoC.
- `00_ambiente`: informações do ambiente.
- `01_antes`: evidências antes do hardening.
- `02_hardening`: evidências da primeira rodada de hardening.
- `03_depois`: auditoria após a primeira rodada.
- `04_comparacao`: reservado para comparações.
- `05_hardening_extra`: evidências da segunda rodada.
- `06_depois_extra`: auditoria final após a segunda rodada.
- `cd ~/av4-hardening`: entra no diretório principal da PoC.

---

# 9. VM - Registrar informações do ambiente

```bash
date | tee 00_ambiente/data_execucao.txt
lsb_release -a | tee 00_ambiente/versao_ubuntu.txt
uname -a | tee 00_ambiente/kernel.txt
whoami | tee 00_ambiente/usuario.txt
hostnamectl | tee 00_ambiente/hostnamectl.txt
ip a | tee 00_ambiente/interfaces_rede.txt
```

**Explicação:**

- `date`: registra data e horário da execução.
- `lsb_release -a`: mostra versão e distribuição do Ubuntu.
- `uname -a`: mostra versão do kernel e arquitetura.
- `whoami`: registra o usuário usado na execução.
- `hostnamectl`: mostra hostname, sistema operacional, kernel e dados da máquina.
- `ip a`: lista interfaces de rede e endereços IP.
- `tee arquivo.txt`: mostra a saída na tela e ao mesmo tempo grava no arquivo indicado. Isso cria evidência auditável da execução.

---

# 10. VM - Coletar estado inicial antes do hardening

```bash
systemctl list-units --type=service --state=running | tee 01_antes/servicos_ativos_antes.txt

ss -tulpen | tee 01_antes/portas_antes.txt

sudo ufw status verbose | tee 01_antes/firewall_antes.txt

sudo sysctl net.ipv4.ip_forward \
net.ipv4.tcp_syncookies \
net.ipv4.conf.all.accept_redirects \
net.ipv4.conf.all.rp_filter \
| tee 01_antes/sysctl_antes.txt
```

**Explicação:**

- `systemctl list-units --type=service --state=running`: lista serviços systemd em execução.
- `ss -tulpen`: lista portas TCP/UDP em escuta e processos relacionados.
  - `-t`: TCP.
  - `-u`: UDP.
  - `-l`: listening.
  - `-p`: processo associado.
  - `-e`: informações estendidas.
  - `-n`: não resolve nomes, mostrando portas e IPs numericamente.
- `sudo ufw status verbose`: mostra o status do firewall UFW antes do hardening.
- `sudo sysctl ...`: consulta parâmetros de kernel/rede relevantes para segurança.
- `net.ipv4.ip_forward`: indica se a máquina encaminha pacotes como roteador.
- `net.ipv4.tcp_syncookies`: proteção contra certos cenários de SYN flood.
- `net.ipv4.conf.all.accept_redirects`: controla aceitação de ICMP redirects.
- `net.ipv4.conf.all.rp_filter`: controla reverse path filtering.
- O objetivo da etapa é gerar uma linha de base para comparação posterior.

---

# 11. VM - Auditoria inicial com Lynis

```bash
sudo lynis audit system --quick --no-colors | tee 01_antes/lynis_tela_antes.txt

sudo cp /var/log/lynis.log 01_antes/lynis_antes.log
sudo cp /var/log/lynis-report.dat 01_antes/lynis-report-antes.dat
sudo chown -R "$USER:$USER" 01_antes

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

**Explicação:**

- `sudo lynis audit system`: executa auditoria local de segurança do sistema.
- `--quick`: executa a auditoria em modo mais direto.
- `--no-colors`: remove códigos de cor da saída, melhorando a legibilidade dos arquivos.
- `tee 01_antes/lynis_tela_antes.txt`: salva a saída exibida no terminal.
- `sudo cp /var/log/lynis.log ...`: copia o log detalhado do Lynis para a pasta de evidências.
- `sudo cp /var/log/lynis-report.dat ...`: copia o relatório estruturado do Lynis.
- `sudo chown -R "$USER:$USER" 01_antes`: altera o dono dos arquivos para o usuário atual, facilitando leitura e versionamento.
- `grep -E`: filtra linhas usando expressão regular estendida.
- `^hardening_index=`: captura o índice de hardening.
- `^warning\[\]=`: captura advertências.
- `^suggestion\[\]=`: captura sugestões.
- `grep -c`: conta quantas linhas correspondem ao padrão.
- `echo`: imprime títulos explicativos no terminal.
- Resultado obtido na PoC:
  - Hardening index: 63
  - Warnings: 1
  - Suggestions: 45

---

# 12. VIRTUALBOX - Criar snapshot antes do hardening

Pela interface do VirtualBox:

```text
Snapshots ->
Take Snapshot ->
Nome: Antes do Hardening
```

**Explicação:**

- O snapshot cria um ponto de restauração antes das alterações.
- Isso reduz risco operacional, pois permite voltar ao estado anterior caso firewall, SSH, PAM ou sysctl sejam configurados incorretamente.

---

# 13. VM - Backup dos arquivos que serão alterados

```bash
sudo cp /etc/ssh/sshd_config 02_hardening/sshd_config.bak 2>/dev/null || true
sudo cp /etc/sysctl.conf 02_hardening/sysctl.conf.bak
sudo cp /etc/login.defs 02_hardening/login.defs.bak
sudo cp /etc/security/pwquality.conf 02_hardening/pwquality.conf.bak 2>/dev/null || true
```

**Explicação:**

- `sudo cp origem destino`: copia arquivos de configuração antes de modificá-los.
- `/etc/ssh/sshd_config`: configuração principal do SSH.
- `/etc/sysctl.conf`: configurações persistentes de parâmetros de kernel.
- `/etc/login.defs`: política de expiração de senha e parâmetros de login.
- `/etc/security/pwquality.conf`: política de qualidade de senha.
- `.bak`: extensão usada para indicar backup.
- `2>/dev/null`: descarta mensagens de erro.
- `|| true`: evita que o script pare caso o arquivo não exista. Isso foi usado em arquivos que podem não estar presentes dependendo da instalação.

---

# 14. VM - Primeira rodada: configurar firewall UFW

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw --force enable
sudo ufw status verbose | tee 02_hardening/firewall_configurado.txt
```

**Explicação:**

- `sudo ufw default deny incoming`: define política padrão para negar conexões de entrada.
- `sudo ufw default allow outgoing`: permite conexões de saída, mantendo o uso normal de internet e atualizações.
- `sudo ufw allow OpenSSH`: libera o perfil do OpenSSH antes de ativar o firewall, evitando bloquear o próprio acesso remoto.
- `sudo ufw --force enable`: ativa o UFW sem pedir confirmação interativa.
- `sudo ufw status verbose`: exibe status, políticas padrão e regras ativas.
- Essa etapa reduz exposição de rede, permitindo apenas serviços explicitamente liberados.

---

# 15. VM - Primeira rodada: hardening básico do SSH

```bash
sudo mkdir -p /etc/ssh/sshd_config.d

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

**Explicação:**

- `sudo mkdir -p /etc/ssh/sshd_config.d`: garante que o diretório de configurações adicionais do SSH exista.
- `cat << 'EOF' ... EOF`: cria um bloco de texto literal, também chamado de here-document.
- `sudo tee /etc/ssh/sshd_config.d/99-av4-hardening.conf`: grava o bloco no arquivo de configuração do SSH com privilégio administrativo.
- `PermitRootLogin no`: bloqueia login direto do usuário root via SSH.
- `MaxAuthTries 3`: limita tentativas de autenticação por conexão.
- `LoginGraceTime 30`: limita o tempo para completar o login.
- `X11Forwarding no`: desabilita encaminhamento de interface gráfica X11.
- `AllowTcpForwarding no`: desabilita uso do SSH para encaminhamento de portas TCP.
- `ClientAliveInterval 300`: envia verificação de atividade após 300 segundos.
- `ClientAliveCountMax 2`: encerra conexão após duas verificações sem resposta.

```bash
sudo /usr/sbin/sshd -t

sudo systemctl restart ssh
sudo systemctl status ssh --no-pager | tee 02_hardening/ssh_status.txt

cat /etc/ssh/sshd_config.d/99-av4-hardening.conf \
| tee 02_hardening/ssh_hardening_aplicado.txt
```

**Explicação:**

- `sudo /usr/sbin/sshd -t`: valida a sintaxe da configuração SSH. Se não houver saída, a configuração está válida.
- `sudo systemctl restart ssh`: reinicia o serviço SSH para aplicar a configuração.
- `sudo systemctl status ssh --no-pager`: mostra o status do serviço sem usar paginação.
- `cat arquivo`: exibe o conteúdo do arquivo.
- `| tee 02_hardening/ssh_hardening_aplicado.txt`: salva a configuração aplicada como evidência.
- Observação: a barra invertida `\` antes do pipe garante que o comando continue na linha seguinte.

---

# 16. VM - Primeira rodada: parâmetros de kernel e rede

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

**Explicação:**

- O comando cria um arquivo persistente de parâmetros `sysctl` em `/etc/sysctl.d/`.
- `net.ipv4.ip_forward = 0`: impede que a VM atue como roteador IPv4.
- `net.ipv4.tcp_syncookies = 1`: ativa proteção contra certos cenários de SYN flood.
- `accept_redirects = 0`: impede aceitar redirects ICMP que poderiam alterar rotas.
- `accept_source_route = 0`: bloqueia pacotes com source routing.
- `secure_redirects = 0`: desabilita redirects considerados seguros; para servidor comum, redirects não são necessários.
- `send_redirects = 0`: impede a máquina de enviar redirects ICMP.
- `rp_filter = 1`: ativa reverse path filtering para mitigar tráfego com origem suspeita.
- `log_martians = 1`: registra pacotes com endereços inválidos ou suspeitos.
- `icmp_echo_ignore_broadcasts = 1`: ignora ping para broadcast, reduzindo risco de amplificação.
- `icmp_ignore_bogus_error_responses = 1`: ignora respostas ICMP inválidas.
- `kernel.randomize_va_space = 2`: ativa ASLR, dificultando exploração baseada em endereços previsíveis.
- `fs.protected_hardlinks = 1` e `fs.protected_symlinks = 1`: protegem contra abusos envolvendo hardlinks e symlinks.

```bash
sudo sysctl --system | tee 02_hardening/sysctl_aplicacao.txt

sudo sysctl net.ipv4.ip_forward \
net.ipv4.tcp_syncookies \
net.ipv4.conf.all.accept_redirects \
net.ipv4.conf.all.rp_filter \
kernel.randomize_va_space \
fs.protected_hardlinks \
fs.protected_symlinks \
| tee 02_hardening/sysctl_verificacao.txt
```

**Explicação:**

- `sudo sysctl --system`: recarrega parâmetros de todos os arquivos sysctl do sistema.
- `tee 02_hardening/sysctl_aplicacao.txt`: salva a saída da aplicação dos parâmetros.
- `sudo sysctl parâmetro`: consulta o valor efetivo do parâmetro no kernel.
- A segunda parte valida se os principais parâmetros foram aplicados.

---

# 17. VM - Primeira rodada: política de senha

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

**Explicação:**

- Cria ou sobrescreve a política de qualidade de senha usada pelo `libpam-pwquality`.
- `minlen = 12`: exige tamanho mínimo de 12 caracteres.
- `dcredit = -1`: exige pelo menos um dígito.
- `ucredit = -1`: exige pelo menos uma letra maiúscula.
- `ocredit = -1`: exige pelo menos um caractere especial.
- `lcredit = -1`: exige pelo menos uma letra minúscula.
- `retry = 3`: permite até três tentativas ao definir uma nova senha.

```bash
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs
sudo sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/' /etc/login.defs
sudo sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE   7/' /etc/login.defs

cat /etc/security/pwquality.conf | tee 02_hardening/pwquality_aplicado.txt

grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE" /etc/login.defs \
| tee 02_hardening/login_defs_aplicado.txt
```

**Explicação:**

- `sed -i`: edita o arquivo diretamente.
- `PASS_MAX_DAYS 90`: define validade máxima de senha em 90 dias.
- `PASS_MIN_DAYS 1`: impede troca de senha várias vezes no mesmo dia para burlar histórico/política.
- `PASS_WARN_AGE 7`: avisa o usuário 7 dias antes da expiração.
- `cat /etc/security/pwquality.conf`: exibe a política aplicada.
- `grep -E`: filtra as linhas de expiração de senha em `/etc/login.defs`.
- Os comandos com `tee` salvam evidências em `02_hardening`.

---

# 18. VM - Primeira rodada: ativar auditd

```bash
sudo systemctl enable --now auditd
sudo systemctl status auditd --no-pager | tee 02_hardening/auditd_status.txt
```

**Explicação:**

- `sudo systemctl enable --now auditd`: habilita o auditd para iniciar com o sistema e já inicia o serviço imediatamente.
- `sudo systemctl status auditd --no-pager`: verifica se o serviço está ativo.
- `tee`: salva o status como evidência.

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

**Explicação:**

- Cria regras persistentes de auditoria em `/etc/audit/rules.d/`.
- `-w caminho`: define um arquivo ou diretório a ser monitorado.
- `-p wa`: monitora escrita (`w`) e alteração de atributos (`a`).
- `-k chave`: adiciona uma chave de busca para facilitar investigação nos logs.
- `/etc/passwd`, `/etc/group` e `/etc/shadow`: arquivos relacionados a usuários, grupos e senhas.
- `/etc/sudoers`: arquivo de privilégios administrativos.
- Arquivos de SSH: monitorados para detectar alterações na configuração de acesso remoto.

```bash
sudo augenrules --load
sudo auditctl -l | tee 02_hardening/audit_rules_carregadas.txt
```

**Explicação:**

- `sudo augenrules --load`: compila/carrega as regras persistentes de auditoria.
- `sudo auditctl -l`: lista as regras de auditoria carregadas no kernel.
- `tee`: salva a evidência das regras ativas.

---

# 19. VM - Primeira rodada: revisar serviços e portas

```bash
systemctl list-units --type=service --state=running \
| tee 02_hardening/servicos_ativos_pos_hardening.txt

ss -tulpen | tee 02_hardening/portas_pos_hardening.txt

for svc in avahi-daemon cups bluetooth; do
  if systemctl list-unit-files | grep -q "^${svc}.service"; then
    echo "Desativando $svc"
    sudo systemctl disable --now "$svc" || true
  else
    echo "$svc não está instalado"
  fi
done | tee 02_hardening/revisao_servicos.txt
```

**Explicação:**

- O primeiro `systemctl` registra os serviços ativos após a primeira rodada.
- `ss -tulpen` registra portas em escuta após a primeira rodada.
- `for svc in avahi-daemon cups bluetooth; do ... done`: percorre uma lista de serviços comuns que podem ser desnecessários em servidor.
- `systemctl list-unit-files | grep -q "^${svc}.service"`: verifica se o serviço existe na máquina.
- `sudo systemctl disable --now "$svc"`: desabilita o serviço no boot e para o serviço imediatamente.
- `|| true`: evita que algum erro interrompa a execução.
- Se o serviço não estiver instalado, registra a mensagem correspondente.
- Na PoC, as portas não mudaram porque a VM já tinha superfície de rede pequena: SSH, DNS local e DHCP.

---

# 20. VM - Auditoria após primeira rodada

```bash
sudo lynis audit system --quick --no-colors | tee 03_depois/lynis_tela_depois.txt

sudo cp /var/log/lynis.log 03_depois/lynis_depois.log
sudo cp /var/log/lynis-report.dat 03_depois/lynis-report-depois.dat
sudo chown -R "$USER:$USER" 03_depois

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

**Explicação:**

- Repete a auditoria Lynis após a primeira rodada de hardening.
- Copia logs e relatório estruturado para `03_depois`.
- Filtra e conta os indicadores principais.
- Permite comparar o estado inicial com o primeiro estado endurecido.
- Resultado obtido na PoC:
  - Hardening index: 70
  - Warnings: 0
  - Suggestions: 39

---

# 21. VM - Segunda rodada: instalar ferramentas extras

```bash
sudo add-apt-repository universe -y
sudo apt update

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

for pkg in fail2ban debsums apt-listchanges libpam-tmpdir apt-show-versions acct sysstat aide rkhunter chkrootkit; do
  if dpkg -s "$pkg" >/dev/null 2>&1; then
    echo "$pkg: instalado"
  else
    echo "$pkg: não instalado"
  fi
done | tee 05_hardening_extra/pacotes_extra_instalados.txt
```

**Explicação:**

- `sudo add-apt-repository universe -y`: habilita o repositório Universe, onde alguns pacotes extras podem estar disponíveis.
- `sudo apt update`: atualiza os índices após habilitar o repositório.
- `fail2ban`: monitora tentativas falhas de autenticação e aplica bloqueios temporários.
- `debsums`: verifica integridade de arquivos instalados por pacotes Debian/Ubuntu.
- `apt-listchanges`: mostra mudanças relevantes de pacotes durante atualizações.
- `libpam-tmpdir`: melhora isolamento de diretórios temporários por usuário em sessões PAM.
- `apt-show-versions`: mostra versões instaladas e disponíveis de pacotes.
- `acct`: habilita contabilidade de processos.
- `sysstat`: coleta estatísticas de desempenho do sistema.
- `aide`: ferramenta de verificação de integridade baseada em base de referência.
- `rkhunter` e `chkrootkit`: ferramentas de busca por indícios de rootkits.
- `dpkg -s "$pkg"`: verifica se o pacote está instalado.
- `>/dev/null 2>&1`: descarta saída normal e erros.
- O loop registra quais pacotes foram instalados com sucesso.

---

# 22. VM - Segunda rodada: configurar Fail2ban para SSH

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

**Explicação:**

- Cria o arquivo local de configuração do Fail2ban.
- `[DEFAULT]`: define parâmetros padrão.
- `bantime = 10m`: bloqueia o IP por 10 minutos.
- `findtime = 10m`: considera tentativas dentro de uma janela de 10 minutos.
- `maxretry = 3`: bloqueia após 3 falhas.
- `[sshd]`: configura a jail específica para SSH.
- `enabled = true`: ativa a jail.
- `port = ssh`: usa a porta definida para o serviço SSH.
- `logpath = %(sshd_log)s`: usa o caminho padrão de logs do SSH no Fail2ban.
- `backend = systemd`: usa o journal do systemd como fonte de logs.

```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban

sudo systemctl status fail2ban --no-pager \
| tee 05_hardening_extra/fail2ban_service_status.txt

sudo fail2ban-client status \
| tee 05_hardening_extra/fail2ban_status.txt

sudo fail2ban-client status sshd \
| tee 05_hardening_extra/fail2ban_sshd_status.txt
```

**Explicação:**

- `sudo systemctl enable --now fail2ban`: habilita e inicia o Fail2ban.
- `sudo systemctl restart fail2ban`: reinicia o serviço para aplicar a configuração.
- `sudo systemctl status fail2ban --no-pager`: verifica se o serviço está rodando.
- `sudo fail2ban-client status`: mostra jails ativas.
- `sudo fail2ban-client status sshd`: mostra detalhes da jail SSH.
- Os arquivos gerados servem como evidência da proteção contra tentativas repetidas de login.

---

# 23. VM - Segunda rodada: reforço extra no SSH

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

**Explicação:**

- Sobrescreve o arquivo de hardening do SSH com controles adicionais.
- `AllowAgentForwarding no`: impede encaminhamento de agente SSH.
- `LogLevel VERBOSE`: aumenta detalhamento dos logs de autenticação.
- `MaxSessions 2`: limita sessões por conexão.
- `TCPKeepAlive no`: desabilita keepalive TCP no SSH.
- As demais opções mantêm o bloqueio de root, limite de tentativas, tempo de login e bloqueio de encaminhamentos.

```bash
sudo /usr/sbin/sshd -t
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager | tee 05_hardening_extra/ssh_status_extra.txt

cat /etc/ssh/sshd_config.d/99-av4-hardening.conf \
| tee 05_hardening_extra/ssh_hardening_extra.txt
```

**Explicação:**

- `sshd -t`: valida a sintaxe antes de reiniciar.
- `systemctl restart ssh`: aplica as alterações.
- `systemctl status ssh`: confirma que o serviço permaneceu ativo.
- `cat ... | tee`: salva a configuração final do SSH como evidência.

---

# 24. VM - Segunda rodada: configurar umask 027

```bash
sudo cp /etc/login.defs 05_hardening_extra/login.defs.bak.extra

if grep -q '^UMASK' /etc/login.defs; then
  sudo sed -i 's/^UMASK.*/UMASK           027/' /etc/login.defs
else
  echo 'UMASK           027' | sudo tee -a /etc/login.defs
fi

grep '^UMASK' /etc/login.defs | tee 05_hardening_extra/umask_login_defs.txt
```

**Explicação:**

- `sudo cp`: cria backup adicional de `/etc/login.defs`.
- `grep -q '^UMASK'`: verifica se já existe linha de UMASK.
- `sed -i`: altera a linha existente para `UMASK 027`.
- `echo ... | sudo tee -a`: adiciona a linha ao fim do arquivo caso ela não exista.
- `UMASK 027`: faz novos arquivos/diretórios nascerem com permissões mais restritivas.
- `grep '^UMASK'`: confirma a configuração aplicada.

```bash
cat << 'EOF' | sudo tee /etc/profile.d/99-av4-umask.sh
umask 027
EOF

sudo chmod 644 /etc/profile.d/99-av4-umask.sh

cat /etc/profile.d/99-av4-umask.sh \
| tee 05_hardening_extra/umask_profiled.txt
```

**Explicação:**

- Cria um script em `/etc/profile.d/` para aplicar `umask 027` em novas sessões de shell.
- `chmod 644`: define permissões de leitura para todos e escrita apenas para root.
- `cat ... | tee`: registra o conteúdo aplicado.
- Esse controle reduz permissões padrão de novos arquivos criados por usuários.

---

# 25. VM - Segunda rodada: desabilitar core dumps

```bash
cat << 'EOF' | sudo tee /etc/security/limits.d/99-av4-disable-coredumps.conf
* hard core 0
* soft core 0
EOF

cat << 'EOF' | sudo tee /etc/sysctl.d/99-av4-coredump-hardening.conf
fs.suid_dumpable = 0
EOF
```

**Explicação:**

- Cria arquivo de limites para impedir geração de core dumps.
- `* hard core 0`: define limite rígido de core dump como zero para todos os usuários.
- `* soft core 0`: define limite flexível de core dump como zero para todos os usuários.
- `fs.suid_dumpable = 0`: impede dump de processos com privilégios elevados ou SUID.
- O objetivo é reduzir risco de exposição de dados sensíveis em dumps de memória.

```bash
sudo sysctl --system | tee 05_hardening_extra/sysctl_coredump_aplicacao.txt

cat /etc/security/limits.d/99-av4-disable-coredumps.conf \
| tee 05_hardening_extra/coredump_limits.txt

sysctl fs.suid_dumpable \
| tee 05_hardening_extra/coredump_sysctl.txt
```

**Explicação:**

- `sudo sysctl --system`: aplica parâmetros sysctl persistentes.
- `cat ... | tee`: salva os limites configurados.
- `sysctl fs.suid_dumpable`: verifica o valor efetivo do parâmetro no momento da coleta.
- Observação: em validações posteriores, se o valor aparecer diferente, deve-se investigar precedência de arquivos sysctl ou interferência de mecanismos do sistema.

---

# 26. VM - Segunda rodada: bloquear protocolos de rede raros

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
```

**Explicação:**

- Cria arquivo de configuração para impedir carregamento de módulos de protocolos raramente usados.
- `install dccp /bin/false`: faz tentativas de carregar o módulo `dccp` falharem.
- `install sctp /bin/false`: bloqueia SCTP.
- `install rds /bin/false`: bloqueia RDS.
- `install tipc /bin/false`: bloqueia TIPC.
- `blacklist`: adiciona os módulos à lista de bloqueio automático.
- A finalidade é reduzir superfície de ataque de protocolos que não são necessários à função da VM.

```bash
cat /etc/modprobe.d/99-av4-disable-rare-network-protocols.conf \
| tee 05_hardening_extra/protocolos_bloqueados.txt
```

**Explicação:**

- Exibe e salva a configuração aplicada como evidência.

---

# 27. VM - Segunda rodada: bloquear USB storage

```bash
cat << 'EOF' | sudo tee /etc/modprobe.d/99-av4-disable-usb-storage.conf
install usb-storage /bin/false
blacklist usb-storage
EOF
```

**Explicação:**

- Impede o carregamento do módulo `usb-storage`.
- `install usb-storage /bin/false`: faz a tentativa de carregamento falhar.
- `blacklist usb-storage`: evita carregamento automático do módulo.
- O objetivo é reduzir risco de uso de dispositivos USB de armazenamento não autorizados.

```bash
cat /etc/modprobe.d/99-av4-disable-usb-storage.conf \
| tee 05_hardening_extra/usb_storage_bloqueado.txt
```

**Explicação:**

- Registra o bloqueio aplicado como evidência.

---

# 28. VM - Segunda rodada: banners legais

```bash
cat << 'EOF' | sudo tee /etc/issue
Acesso restrito. Uso autorizado apenas para fins administrativos e acadêmicos. Atividades podem ser registradas.
EOF

cat << 'EOF' | sudo tee /etc/issue.net
Acesso restrito. Uso autorizado apenas para fins administrativos e acadêmicos. Atividades podem ser registradas.
EOF
```

**Explicação:**

- `/etc/issue`: banner exibido em logins locais.
- `/etc/issue.net`: banner usado por serviços remotos, como SSH, quando configurado.
- A mensagem informa que o acesso é restrito e que atividades podem ser registradas.
- Esse controle tem função de aviso e apoio a políticas administrativas.

```bash
cat /etc/issue | tee 05_hardening_extra/banner_issue.txt
cat /etc/issue.net | tee 05_hardening_extra/banner_issue_net.txt
```

**Explicação:**

- Exibe e salva os banners configurados como evidência.

---

# 29. VM - Segunda rodada: ativar acct e sysstat

```bash
sudo systemctl enable --now acct
sudo systemctl status acct --no-pager | tee 05_hardening_extra/acct_status.txt

sudo sed -i 's/^ENABLED=.*/ENABLED="true"/' /etc/default/sysstat
sudo systemctl enable --now sysstat
sudo systemctl restart sysstat

cat /etc/default/sysstat | tee 05_hardening_extra/sysstat_default.txt
sudo systemctl status sysstat --no-pager | tee 05_hardening_extra/sysstat_status.txt
```

**Explicação:**

- `acct`: habilita contabilidade de processos, permitindo registrar comandos/processos executados.
- `systemctl enable --now acct`: habilita no boot e inicia imediatamente.
- `sed -i 's/^ENABLED=.*/ENABLED="true"/' /etc/default/sysstat`: ativa coleta de estatísticas do sysstat.
- `systemctl enable --now sysstat`: habilita e inicia o serviço.
- `systemctl restart sysstat`: reinicia para aplicar a configuração.
- `cat /etc/default/sysstat`: mostra a configuração aplicada.
- `systemctl status`: confirma estado dos serviços.
- O objetivo é ampliar capacidade de monitoramento e rastreabilidade do sistema.

---

# 30. VM - Segunda rodada: AIDE

```bash
echo "A inicialização do AIDE foi considerada, mas omitida da execução final da PoC devido ao tempo elevado de varredura em ambiente virtualizado." \
| tee 05_hardening_extra/aide_observacao.txt
```

**Explicação:**

- `echo`: imprime a justificativa operacional.
- `tee`: salva a observação em arquivo.
- O AIDE foi instalado, mas a criação da base inicial foi omitida na execução final por tempo elevado.
- Essa decisão foi documentada como limitação operacional, em vez de deixar o controle sem explicação.

---

# 31. VM - Segunda rodada: configurar rounds de hash de senha

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

grep -E '^SHA_CRYPT_MIN_ROUNDS|^SHA_CRYPT_MAX_ROUNDS' /etc/login.defs \
| tee 05_hardening_extra/password_hash_rounds.txt
```

**Explicação:**

- `grep -q`: verifica silenciosamente se a configuração já existe.
- Se existir, `sed -i` altera o valor.
- Se não existir, `echo ... | sudo tee -a` adiciona a linha ao final do arquivo.
- `SHA_CRYPT_MIN_ROUNDS 5000`: define mínimo de rounds para hashing de senha.
- `SHA_CRYPT_MAX_ROUNDS 10000`: define máximo de rounds.
- Mais rounds aumentam o custo computacional para geração/verificação de hashes de senha.
- O último `grep` registra os valores configurados.

---

# 32. VM - Auditoria final após segunda rodada

```bash
sudo lynis audit system --quick --no-colors \
| tee 06_depois_extra/lynis_tela_depois_extra.txt

sudo cp /var/log/lynis.log 06_depois_extra/lynis_depois_extra.log
sudo cp /var/log/lynis-report.dat 06_depois_extra/lynis-report-depois-extra.dat
sudo chown -R "$USER:$USER" 06_depois_extra

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

**Explicação:**

- Executa a auditoria final do Lynis após a segunda rodada de hardening.
- Copia logs e relatório estruturado.
- Filtra e conta índice, warnings e suggestions.
- Resultado obtido na PoC:
  - Hardening index: 81
  - Warnings: 0
  - Suggestions: 24

---

# 33. VM - Gerar tabela comparativa final

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

column -s, -t 06_depois_extra/comparativo_lynis_3_etapas.csv \
| tee 06_depois_extra/comparativo_lynis_3_etapas_tabela.txt

cat 06_depois_extra/comparativo_lynis_3_etapas_tabela.txt
```

**Explicação:**

- `VAR=$(comando)`: executa o comando e armazena a saída em uma variável.
- `grep '^hardening_index='`: extrai a linha do índice de hardening.
- `cut -d= -f2`: separa a linha usando `=` e pega o segundo campo, ou seja, o valor.
- `grep -c`: conta warnings e suggestions.
- O bloco entre `{ ... }` gera um CSV com três métricas.
- `tee comparativo_lynis_3_etapas.csv`: salva a comparação em CSV.
- `column -s, -t`: transforma o CSV em tabela alinhada no terminal.
- `cat`: exibe a tabela final.
- Essa etapa cria a evidência quantitativa principal da PoC: 63 → 70 → 81.

---

# 34. VIRTUALBOX - Criar snapshot final

Pela interface gráfica:

```text
Snapshots ->
Take Snapshot ->
Nome: Depois do Hardening Final
```

**Explicação:**

- Cria um ponto de restauração após a aplicação do hardening.
- Permite preservar o estado final da PoC para demonstração, revisão ou recuperação.

---

# 35. VM - Validação final dos controles aplicados

```bash
mkdir -p ~/av4-hardening/07_validacao_final
cd ~/av4-hardening
```

**Explicação:**

- Cria o diretório de validação final.
- Entra no diretório principal da PoC para salvar os resultados com caminhos relativos.

---

## 35.1 Ver configuração efetiva do SSH

```bash
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|maxauthtries|logingracetime|x11forwarding|allowtcpforwarding|allowagentforwarding|clientaliveinterval|clientalivecountmax|loglevel|maxsessions|tcpkeepalive' \
| tee 07_validacao_final/ssh_config_efetiva.txt
```

**Explicação:**

- `sudo sshd -T`: mostra a configuração efetiva do SSH depois de ler todos os arquivos de configuração.
- Isso é mais confiável do que olhar apenas um arquivo isolado.
- `grep -E`: filtra apenas os parâmetros relevantes para a PoC.
- `tee`: salva a configuração efetiva em arquivo.
- A autenticação por senha permanece habilitada porque a PoC usa acesso do host para a VM via SSH. Chaves SSH seriam uma melhoria adicional, mas ficaram fora do escopo prático.

---

## 35.2 Verificar firewall

```bash
sudo ufw status verbose | tee 07_validacao_final/ufw_status_final.txt
sudo ufw status numbered | tee 07_validacao_final/ufw_regras_numeradas.txt
```

**Explicação:**

- `sudo ufw status verbose`: mostra status, logs, políticas padrão e regras do firewall.
- `sudo ufw status numbered`: mostra regras numeradas, útil para auditoria e eventual remoção de regras.
- Essa validação confirma que o firewall ficou ativo e com regra explícita para OpenSSH.

---

## 35.3 Verificar portas em escuta

```bash
ss -tulpen | tee 07_validacao_final/portas_finais.txt
```

**Explicação:**

- Lista portas TCP/UDP em escuta no estado final.
- Na PoC, as portas finais permaneceram semelhantes às iniciais porque a VM já era mínima.
- O ganho principal foi a ativação do firewall e o endurecimento do SSH, não a redução da quantidade de portas.

---

## 35.4 Verificar parâmetros sysctl efetivos

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

**Explicação:**

- Consulta os valores efetivos dos parâmetros de kernel/rede após todas as alterações.
- Confirma controles contra roteamento indevido, redirects, source routing, spoofing e abuso de links.
- Inclui `fs.suid_dumpable` para verificar restrição de dumps de processos privilegiados.
- Caso algum valor final seja diferente do esperado, isso deve ser tratado como ponto de investigação ou limitação.

---

## 35.5 Verificar auditd

```bash
sudo systemctl status auditd --no-pager | tee 07_validacao_final/auditd_status_final.txt
sudo auditctl -s | tee 07_validacao_final/auditd_estado_final.txt
sudo auditctl -l | tee 07_validacao_final/auditd_regras_finais.txt
```

**Explicação:**

- `systemctl status auditd`: confirma que o serviço de auditoria está ativo.
- `auditctl -s`: mostra estado do subsistema de auditoria, incluindo backlog e eventos perdidos.
- `auditctl -l`: lista regras de auditoria carregadas.
- Esses comandos validam a rastreabilidade após o hardening.

---

## 35.6 Verificar Fail2ban

```bash
sudo systemctl status fail2ban --no-pager | tee 07_validacao_final/fail2ban_status_servico_final.txt
sudo fail2ban-client status | tee 07_validacao_final/fail2ban_status_final.txt
sudo fail2ban-client status sshd | tee 07_validacao_final/fail2ban_sshd_final.txt
```

**Explicação:**

- `systemctl status fail2ban`: confirma o estado do serviço.
- `fail2ban-client status`: lista jails ativas.
- `fail2ban-client status sshd`: mostra detalhes da jail SSH, como tentativas falhas e IPs banidos.
- Essa validação confirma a mitigação contra tentativas repetidas de login.

---

## 35.7 Verificar AppArmor, se estiver presente

```bash
sudo systemctl status apparmor --no-pager | tee 07_validacao_final/apparmor_status.txt || true
sudo aa-status | tee 07_validacao_final/apparmor_aa_status.txt || true
```

**Explicação:**

- `systemctl status apparmor`: verifica se o serviço AppArmor está presente e ativo.
- `aa-status`: mostra perfis AppArmor carregados.
- `|| true`: evita erro caso o serviço ou comando não esteja disponível.
- Observação: se aparecerem perfis de aplicações do host, como Discord, Brave ou VS Code, a coleta provavelmente foi feita no host e não deve ser usada como evidência da VM.

---

## 35.8 Verificar política PAM de senha

```bash
grep -n 'pam_pwquality' /etc/pam.d/common-password \
| tee 07_validacao_final/pam_pwquality_verificacao.txt || true
```

**Explicação:**

- `grep -n`: procura o texto e mostra o número da linha.
- `pam_pwquality`: módulo PAM responsável por aplicar regras de qualidade de senha.
- Esse comando verifica se a política definida em `pwquality.conf` está conectada ao fluxo real de alteração de senhas.
- `|| true`: evita erro caso a linha não seja encontrada.

---

## 35.9 Verificar integridade de pacotes com debsums

```bash
sudo debsums -s | tee 07_validacao_final/debsums_resultado.txt
```

**Explicação:**

- `debsums -s`: verifica checksums de arquivos instalados por pacotes e mostra apenas problemas.
- Se não houver saída, não foram reportadas divergências nos arquivos verificados.
- Esse controle ajuda na verificação de integridade do sistema.

---

## 35.10 Registrar árvore de evidências

```bash
sudo apt install -y tree
tree -a ~/av4-hardening | tee 07_validacao_final/arvore_evidencias.txt
```

**Explicação:**

- `sudo apt install -y tree`: instala o utilitário `tree`.
- `tree -a ~/av4-hardening`: lista a estrutura completa de diretórios e arquivos, incluindo ocultos.
- `tee`: salva a árvore de evidências.
- Essa etapa facilita demonstrar organização e rastreabilidade da PoC.

---

## 35.11 Gerar manifesto hash das evidências

```bash
find ~/av4-hardening -type f | sort | xargs sha256sum \
| tee 07_validacao_final/sha256_manifesto_evidencias.txt
```

**Explicação:**

- `find ~/av4-hardening -type f`: lista todos os arquivos de evidência.
- `sort`: ordena a lista para gerar saída estável.
- `xargs sha256sum`: calcula o hash SHA-256 de cada arquivo.
- `tee`: salva o manifesto.
- O manifesto permite verificar se os arquivos foram alterados depois da coleta.

---

# Resultado geral da PoC

A execução documentada mostrou evolução mensurável da postura de segurança:

```text
Hardening index: 63 -> 70 -> 81
Warnings:         1  -> 0  -> 0
Suggestions:      45 -> 39 -> 24
```

**Interpretação:**

- A VM já era relativamente enxuta, por isso algumas evidências, como portas abertas, não mudaram muito.
- O principal ganho ocorreu na configuração: firewall ativo, SSH endurecido, parâmetros de kernel/rede ajustados, auditoria com auditd, Fail2ban ativo, política de senha e validação final.
- O hardening não prova invulnerabilidade. Ele reduz superfície de ataque, melhora rastreabilidade e cria evidências técnicas para auditoria.
