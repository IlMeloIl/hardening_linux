# AV4 - PoC Hardening Linux com CIS Benchmark + Lynis
### Ambiente:
 - HOST: Ubuntu 24.04.4 LTS
 - VM: Ubuntu Server 24.04.4 LTS
 - Ferramentas principais: VirtualBox, Lynis, UFW, auditd, Fail2ban

# 1. HOST - Instalar ferramentas para gravação e virtualização

sudo apt update
sudo apt install -y software-properties-common

# Instalar VirtualBox
sudo apt install -y virtualbox virtualbox-dkms dkms build-essential linux-headers-$(uname -r)

# Abrir VirtualBox
virtualbox

# 2. HOST - Baixar ISO do Ubuntu Server 24.04.4 LTS

https://ubuntu.com/download/server/thank-you?version=24.04.4&architecture=amd64&lts=true

# 3. VIRTUALBOX - Criar a VM

### Fazer pela interface gráfica do VirtualBox:

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

# 4. VIRTUALBOX - Configurar redirecionamento SSH

### Com a VM desligada:

 Settings ->
 Network ->
 Adapter 1: NAT ->
 Advanced ->
 Port Forwarding ->
 ->
 Adicionar: ->
 Name: SSH ->
 Protocol: TCP ->
 Host IP: 127.0.0.1 ->
 Host Port: 2222 ->
 Guest IP: deixar vazio ->
 Guest Port: 22

# 5. VM - Instalar Ubuntu Server
### Iniciar a VM e instalar o Ubuntu Server.

#### Durante a instalação:

 Username: aluno ->
 Hostname: av4-hardening ->
 OpenSSH Server: instalar ->
 Featured Server Snaps: não selecionar nada ->

 Após finalizar:
 Reboot Now

# 6. HOST - Acessar a VM por SSH

ssh -p 2222 aluno@127.0.0.1

# A PARTIR DAQUI, TODOS OS COMANDOS SÃO DENTRO DA VM

# 7. VM - Atualizar sistema e instalar ferramentas básicas
```
sudo apt update
sudo apt full-upgrade -y

sudo apt install -y lynis ufw auditd audispd-plugins libpam-pwquality openssh-server curl wget net-tools vim

sudo reboot
```
 Depois do reboot, conectar de novo pelo HOST:
 ```ssh -p 2222 aluno@127.0.0.1```

# 8. VM - Criar estrutura de evidências
```
mkdir -p ~/av4-hardening/{00_ambiente,01_antes,02_hardening,03_depois,04_comparacao,05_hardening_extra,06_depois_extra}
cd ~/av4-hardening
```

# 9. VM - Registrar informações do ambiente
```
date | tee 00_ambiente/data_execucao.txt
lsb_release -a | tee 00_ambiente/versao_ubuntu.txt
uname -a | tee 00_ambiente/kernel.txt
whoami | tee 00_ambiente/usuario.txt
hostnamectl | tee 00_ambiente/hostnamectl.txt
ip a | tee 00_ambiente/interfaces_rede.txt
```
# 10. VM - Coletar estado inicial antes do hardening
```
systemctl list-units --type=service --state=running | tee 01_antes/servicos_ativos_antes.txt

ss -tulpen | tee 01_antes/portas_antes.txt

sudo ufw status verbose | tee 01_antes/firewall_antes.txt

sudo sysctl net.ipv4.ip_forward \
net.ipv4.tcp_syncookies \
net.ipv4.conf.all.accept_redirects \
net.ipv4.conf.all.rp_filter \
| tee 01_antes/sysctl_antes.txt
```
# 11. VM - Auditoria inicial com Lynis
```
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
# Resultado obtido na PoC:
  - Hardening index: 63
  - Warnings: 1
  - Suggestions: 45

# 12. VIRTUALBOX - Criar snapshot antes do hardening

### Fazer pela interface do VirtualBox:

 Snapshots ->
 Take Snapshot ->
 Nome: Antes do Hardening

# 13. VM - Backup dos arquivos que serão alterados
```
sudo cp /etc/ssh/sshd_config 02_hardening/sshd_config.bak 2>/dev/null || true
sudo cp /etc/sysctl.conf 02_hardening/sysctl.conf.bak
sudo cp /etc/login.defs 02_hardening/login.defs.bak
sudo cp /etc/security/pwquality.conf 02_hardening/pwquality.conf.bak 2>/dev/null || true
```
# 14. VM - Primeira rodada: configurar firewall UFW
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw --force enable
sudo ufw status verbose | tee 02_hardening/firewall_configurado.txt
```
# 15. VM - Primeira rodada: hardening básico do SSH
```
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
# Validar configuração.
### Se não aparecer nada, está correto.
```
sudo /usr/sbin/sshd -t

sudo systemctl restart ssh
sudo systemctl status ssh --no-pager | tee 02_hardening/ssh_status.txt

cat /etc/ssh/sshd_config.d/99-av4-hardening.conf
| tee 02_hardening/ssh_hardening_aplicado.txt
```
# 16. VM - Primeira rodada: parâmetros de kernel e rede
```
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
```
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
# 17. VM - Primeira rodada: política de senha
```
cat << 'EOF' | sudo tee /etc/security/pwquality.conf
minlen = 12
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
retry = 3
EOF
```
```
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs
sudo sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   1/' /etc/login.defs
sudo sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE   7/' /etc/login.defs

cat /etc/security/pwquality.conf | tee 02_hardening/pwquality_aplicado.txt

grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_WARN_AGE" /etc/login.defs \
| tee 02_hardening/login_defs_aplicado.txt
```

# 18. VM - Primeira rodada: ativar auditd
```
sudo systemctl enable --now auditd
sudo systemctl status auditd --no-pager | tee 02_hardening/auditd_status.txt
```
```
cat << 'EOF' | sudo tee /etc/audit/rules.d/99-av4-hardening.rules
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k privilege
-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /etc/ssh/sshd_config.d/ -p wa -k ssh_config
EOF
```
```
sudo augenrules --load
sudo auditctl -l | tee 02_hardening/audit_rules_carregadas.txt
```
# 19. VM - Primeira rodada: revisar serviços e portas
```
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
# 20. VM - Auditoria após primeira rodada
```
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
### Resultado obtido na PoC:
  - Hardening index: 70
  - Warnings: 0
  - Suggestions: 39

# 21. VM - Segunda rodada: instalar ferramentas extras
```
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
# 22. VM - Segunda rodada: configurar Fail2ban para SSH
```
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
```
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban

sudo systemctl status fail2ban --no-pager \
| tee 05_hardening_extra/fail2ban_service_status.txt

sudo fail2ban-client status \
| tee 05_hardening_extra/fail2ban_status.txt

sudo fail2ban-client status sshd \
| tee 05_hardening_extra/fail2ban_sshd_status.txt
```
# 23. VM - Segunda rodada: reforço extra no SSH
```
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
```
sudo /usr/sbin/sshd -t
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager | tee 05_hardening_extra/ssh_status_extra.txt

cat /etc/ssh/sshd_config.d/99-av4-hardening.conf \
| tee 05_hardening_extra/ssh_hardening_extra.txt
```
# 24. VM - Segunda rodada: configurar umask 027
```
sudo cp /etc/login.defs 05_hardening_extra/login.defs.bak.extra

if grep -q '^UMASK' /etc/login.defs; then
  sudo sed -i 's/^UMASK.*/UMASK           027/' /etc/login.defs
else
  echo 'UMASK           027' | sudo tee -a /etc/login.defs
fi

grep '^UMASK' /etc/login.defs | tee 05_hardening_extra/umask_login_defs.txt
```
```
cat << 'EOF' | sudo tee /etc/profile.d/99-av4-umask.sh
umask 027
EOF
```
```
sudo chmod 644 /etc/profile.d/99-av4-umask.sh

cat /etc/profile.d/99-av4-umask.sh \
| tee 05_hardening_extra/umask_profiled.txt
```
# 25. VM - Segunda rodada: desabilitar core dumps
```
cat << 'EOF' | sudo tee /etc/security/limits.d/99-av4-disable-coredumps.conf
* hard core 0
* soft core 0
EOF

cat << 'EOF' | sudo tee /etc/sysctl.d/99-av4-coredump-hardening.conf
fs.suid_dumpable = 0
EOF
```
```
sudo sysctl --system | tee 05_hardening_extra/sysctl_coredump_aplicacao.txt

cat /etc/security/limits.d/99-av4-disable-coredumps.conf \
| tee 05_hardening_extra/coredump_limits.txt

sysctl fs.suid_dumpable \
| tee 05_hardening_extra/coredump_sysctl.txt
```
# 26. VM - Segunda rodada: bloquear protocolos de rede raros
```
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
```
cat /etc/modprobe.d/99-av4-disable-rare-network-protocols.conf \
| tee 05_hardening_extra/protocolos_bloqueados.txt
```
# 27. VM - Segunda rodada: bloquear USB storage
```
cat << 'EOF' | sudo tee /etc/modprobe.d/99-av4-disable-usb-storage.conf
install usb-storage /bin/false
blacklist usb-storage
EOF
```
```
cat /etc/modprobe.d/99-av4-disable-usb-storage.conf \
| tee 05_hardening_extra/usb_storage_bloqueado.txt
```
# 28. VM - Segunda rodada: banners legais
```
cat << 'EOF' | sudo tee /etc/issue
Acesso restrito. Uso autorizado apenas para fins administrativos e acadêmicos. Atividades podem ser registradas.
EOF
```
```
cat << 'EOF' | sudo tee /etc/issue.net
Acesso restrito. Uso autorizado apenas para fins administrativos e acadêmicos. Atividades podem ser registradas.
EOF
```
```
cat /etc/issue | tee 05_hardening_extra/banner_issue.txt
cat /etc/issue.net | tee 05_hardening_extra/banner_issue_net.txt
```
# 29. VM - Segunda rodada: ativar acct e sysstat
```
sudo systemctl enable --now acct
sudo systemctl status acct --no-pager | tee 05_hardening_extra/acct_status.txt

sudo sed -i 's/^ENABLED=.*/ENABLED="true"/' /etc/default/sysstat
sudo systemctl enable --now sysstat
sudo systemctl restart sysstat

cat /etc/default/sysstat | tee 05_hardening_extra/sysstat_default.txt
sudo systemctl status sysstat --no-pager | tee 05_hardening_extra/sysstat_status.txt
```
# 30. VM - Segunda rodada: AIDE
```
echo "A inicialização do AIDE foi considerada, mas omitida da execução final da PoC devido ao tempo elevado de varredura em ambiente virtualizado." \
| tee 05_hardening_extra/aide_observacao.txt
```
# 31. VM - Segunda rodada: configurar rounds de hash de senha
```
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
# 32. VM - Auditoria final após segunda rodada
```
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
### Resultado obtido na PoC:
  - Hardening index: 81
  - Warnings: 0
  - Suggestions: 24

# 35. VM - Gerar tabela comparativa final
```
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
# 36. VIRTUALBOX - Criar snapshot final

### Fazer pela interface gráfica:
Snapshots ->
Take Snapshot ->
Nome: Depois do Hardening Final

# 37. VM - Validação final dos controles aplicados
```
mkdir -p ~/av4-hardening/07_validacao_final
cd ~/av4-hardening
```
# Ver configuração efetiva do SSH, não apenas o arquivo editado
```
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|maxauthtries|logingracetime|x11forwarding|allowtcpforwarding|allowagentforwarding|clientaliveinterval|clientalivecountmax|loglevel|maxsessions|tcpkeepalive' \
| tee 07_validacao_final/ssh_config_efetiva.txt
```
# Verificar firewall
```
sudo ufw status verbose | tee 07_validacao_final/ufw_status_final.txt
sudo ufw status numbered | tee 07_validacao_final/ufw_regras_numeradas.txt
```
# Verificar portas em escuta
```
ss -tulpen | tee 07_validacao_final/portas_finais.txt
```
# Verificar parâmetros sysctl efetivos
```
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

# Verificar auditd
```
sudo systemctl status auditd --no-pager | tee 07_validacao_final/auditd_status_final.txt
sudo auditctl -s | tee 07_validacao_final/auditd_estado_final.txt
sudo auditctl -l | tee 07_validacao_final/auditd_regras_finais.txt
```

# Verificar Fail2ban
```
sudo systemctl status fail2ban --no-pager | tee 07_validacao_final/fail2ban_status_servico_final.txt
sudo fail2ban-client status | tee 07_validacao_final/fail2ban_status_final.txt
sudo fail2ban-client status sshd | tee 07_validacao_final/fail2ban_sshd_final.txt
```

# Verificar AppArmor, se estiver presente
```
sudo systemctl status apparmor --no-pager | tee 07_validacao_final/apparmor_status.txt || true
sudo aa-status | tee 07_validacao_final/apparmor_aa_status.txt || true
```

# Verificar política PAM de senha
```
grep -n 'pam_pwquality' /etc/pam.d/common-password \
| tee 07_validacao_final/pam_pwquality_verificacao.txt || true
```

# Verificar integridade de pacotes com debsums
```
sudo debsums -s | tee 07_validacao_final/debsums_resultado.txt
```
# Registrar árvore de evidências
```
sudo apt install -y tree
tree -a ~/av4-hardening | tee 07_validacao_final/arvore_evidencias.txt
```
# Gerar manifesto hash das evidências
```
find ~/av4-hardening -type f | sort | xargs sha256sum \
| tee 07_validacao_final/sha256_manifesto_evidencias.txt
```
