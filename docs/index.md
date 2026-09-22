---
icon: lucide/rocket
---
# Tutorial de Configuração de Serviços e Redes no Debian

Este documento descreve o passo a passo cronológico para a configuração de uma Máquina Virtual Debian de laboratório.

**Identificação do Servidor:** `madrid.espanha.lab`

---

## 1. Configuração de Identidade (Hostname e Hosts)

Antes de iniciar os serviços, é necessário definir o nome de domínio totalmente qualificado (FQDN) do servidor no sistema.

### Passo A: Configurar o Hostname
O arquivo `/etc/hostname` define o nome curto do sistema. Altere-o diretamente com o comando:
```bash
sudo echo "madrid" | sudo tee /etc/hostname
```

### Passo B: Configurar o Arquivo /etc/hosts
Para mapear o nome da máquina local com o IP de loopback e o domínio completo, configure o arquivo `/etc/hosts`:
```bash
sudo cat << 'EOF' > /etc/hosts
127.0.0.1   localhost
127.0.1.1   madrid.espanha.lab madrid

# The following lines are desirable for IPv6 capable hosts
::1         localhost ip6-localhost ip6-loopback
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters
EOF
```

### Passo C: Aplicar e Verificar
Para aplicar as alterações sem reiniciar o servidor, use o comando:
```bash
sudo hostnamectl set-hostname madrid.espanha.lab

# Verifique se o nome foi aplicado corretamente
hostname -f
```

---

## 2. Instalação e Configuração Inicial do SSH (OpenSSH-Server)

O próximo passo é obter o acesso remoto ao terminal do servidor instalando o serviço SSH e ajustando suas configurações básicas.

```bash
# Atualizar a lista de repositórios e instalar o servidor OpenSSH
sudo apt update && sudo apt install openssh-server -y

# Iniciar o serviço e configurá-lo para iniciar junto com o sistema
sudo systemctl start ssh
sudo systemctl enable ssh
```
*(Nota: Caso tenha alterado a porta padrão 22 para a porta **2222** no arquivo `/etc/ssh/sshd_config`, reinicie o serviço com `sudo systemctl restart ssh`).*

---

## 3. Configuração de Rede (3 Interfaces)

Com o serviço SSH ativo, realiza-se o provisionamento e o levantamento das três placas de rede configuradas no arquivo `/etc/network/interfaces`.

```bash
# Reiniciar o serviço de rede para aplicar as novas configurações das 3 placas
sudo systemctl restart networking

# Comando alternativo para subir todas as interfaces configuradas de uma vez
sudo ifup -a

# Verificar se as 3 placas subiram e receberam os IPs corretamente (procure pelo status UP)
ip a
```

---

## 4. Configuração do Par de Chaves (Acesso Sem Senha)

Após as redes estarem ativas e comunicáveis, configura-se o par de chaves criptográficas para permitir o login seguro sem a necessidade de digitar senhas.

1. **Na sua máquina física (fora da VM)**, gere o par de chaves:
   ```bash
   ssh-keygen -t ed25519
   ```
   *(Pressione Enter em todas as etapas para não definir uma senha para a chave).*

2. **Envie a chave pública para a VM** especificando a porta customizada do servidor:
   * **Via PowerShell (Windows):**
     ```powershell
     cat ~/.ssh/id_ed25519.pub | ssh -p 2222 usuario@ip_do_servidor "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
     ```
   * **Via Terminal (Linux/macOS):**
     ```bash
     ssh-copy-id -p 2222 usuario@ip_do_servidor
     ```

3. **Validação do acesso direto:**
   ```bash
   ssh -p 2222 usuario@ip_do_servidor
   ```

---

## 5. Servidor Web HTTP (Caddy Server)

Substituindo servidores tradicionais, o Caddy é utilizado por sua leveza e facilidade de configuração.

### Passo A: Instalação das Dependências e Repositório Oficial
Como o Caddy não está nos repositórios padrão do Debian, adicionamos a chave GPG e o repositório oficial estável:
```bash
# Instalar dependências necessárias
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl

# Importar a chave GPG oficial do Caddy
curl -1sLf 'https://cloudsmith.io' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

# Adicionar o repositório oficial à lista de fontes
curl -1sLf 'https://cloudsmith.io' | sudo tee /etc/apt/sources.list.d/caddy-stable.list

# Atualizar o índice e instalar o Caddy
sudo apt update && sudo apt install caddy -y
```

### Passo B: Ativação do Serviço
```bash
# Iniciar o serviço do Caddy e habilitar na inicialização do sistema
sudo systemctl start caddy
sudo systemctl enable caddy
```

### Passo C: Configuração da Página e Mensagem Simples
1. Criar o diretório e o arquivo HTML com o formato limpo exigido:
   ```bash
   sudo mkdir -p /var/www/html
   sudo cat << 'EOF' > /var/www/html/index.html
   <!DOCTYPE html>
   <html>
   <head>
       <title>Minha VM</title>
   </head>
   <body>
       <h1>Página funcionando e ok!</h1>
       <h3>Servidor: madrid.espanha.lab</h3>
   </body>
   </html>
   EOF
   ```

2. Configurar o arquivo principal do Caddy (`/etc/caddy/Caddyfile`) para servir este diretório na porta HTTP comum (80):
   ```bash
   sudo cat << 'EOF' > /etc/caddy/Caddyfile
   :80 {
       root * /var/www/html
       file_server
   }
   EOF
   ```

3. Aplicar as alterações recarregando o serviço:
   ```bash
   sudo systemctl reload caddy
   ```

---

## 6. Servidor de Área de Trabalho Remota (RDP)

Por fim, instala-se o protocolo RDP para viabilizar conexões visuais e gráficas ao ambiente Debian.

1. **Instalação do servidor Xrdp:**
   ```bash
   sudo apt install xrdp -y
   ```

2. **Ativação do serviço:**
   ```bash
   sudo systemctl start xrdp
   sudo systemctl enable xrdp
   ```

3. **Ajuste de permissões de certificados:**
   ```bash
   sudo adduser xrdp ssl-cert
   ```

---

## Resumo da Tabela de Portas do Laboratório

| Serviço | Protocolo | Porta | Validação / Teste |
| :--- | :--- | :--- | :--- |
| **SSH** | OpenSSH | `2222` | Conexão via Terminal/PowerShell (`ssh -p 2222`) |
| **HTTP** | Caddy Server | `80` | Acesso via Navegador Web |
| **RDP** | Xrdp | `3389` | Conexão de Área de Trabalho Remota (`mstsc`) |
