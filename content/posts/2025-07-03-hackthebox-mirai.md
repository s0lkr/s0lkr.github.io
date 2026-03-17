---
title: "Mirai – Technical Report"
date: 2017-10-03
description: "Analysis of the Mirai machine from HackTheBox."
categories: ["HackTheBox"]
tags: ["linux", "pihole", "credencialdefault", "raspberry", "IoT", "mirai"]
featureImage: "https://labs.hackthebox.com/storage/avatars/4ef1ea77e69185063d4200d7d0142baa.png"
draft: false
---

# 🗞 [Mirai] – Relatório Técnico

## 1. Identificação
- **Nome da máquina**: Mirai
- **Sistema operacional**: Raspbian (Linux para Raspberry Pi)
- **IP**: 10.10.10.48
- **Data da análise**: 3 de Outubro de 2017
- **Dificuldade**: Fácil
- **Classificação**: Oficial

---
## 2. Objetivo

> Demonstrar o vetor de ataque contra dispositivos IoT mal configurados, explorado pelo malware Mirai, obtendo acesso completo ao sistema e recuperando evidências forenses.

---
## 3. Metodologia

- **Coleta de informações**: Scan completo de portas com Nmap
    
- **Enumeração**:
    - Fuzzing de diretórios web com Dirbuster
    - Identificação de serviços vulneráveis
- **Exploração**:
    - Acesso via SSH com credenciais padrão
    - Análise de partições e recuperação de arquivos
- **Pós-exploração**:
    - Recuperação forense de arquivos deletados
    - Análise de artefatos do sistema

---

## 4. Coleta de Informações

**Scan Nmap** 

```bash
nmap -sCV -Pn -p- -T4 10.10.10.48 -oA initial-nmap
````

**Resultados**:
![NON](file-20250703224759.png)

| Porta | Protocolo | Serviço | Versão                        |
| ----- | --------- | ------- | ----------------------------- |
| 22    | TCP       | SSH     | OpenSSH 6.7p1 Debian 5+deb8u3 |
| 53    | TCP       | DNS     | dnsmasq 2.76                  |
| 80    | TCP       | HTTP    | lighttpd 1.4.35               |
| 1361  | TCP       | UPnP    | Platinum UPnP 1.0.5.13        |
| 32400 | TCP       | HTTP    | Plex Media Server             |
| 32469 | TCP       | UPnP    | Platinum UPnP 1.0.5.13        |

**Observação**: Página web inicialmente apresenta tela em branco.

---

## 5. Enumeração (Ativa)

**Fuzzing de diretórios**:
- Ferramenta: Dirbuster (wordlist lowercase medium)
    
- Diretórios encontrados:
    - `/admin` → Painel Pi-hole (sistema de bloqueio de anúncios)
        ![[Pasted image 20250703224859.png]]
    - `/phpmyadmin` (inacessível)
        

**Conclusão**: Dispositivo é um Raspberry Pi rodando Raspbian com Pi-hole instalado.

---

## 6. Identificação de Vulnerabilidades

### Vulnerabilidade #1
- **Categoria**: A6:2017 - Security Misconfiguration (OWASP)
- **Local**: Serviço SSH com credenciais padrão (`pi:raspberry`)
- **Impacto**: Acesso completo ao sistema

### Vulnerabilidade #2
- **Categoria**: A5:2017 - Broken Access Control (OWASP)
- **Local**: Usuário `pi` com permissões sudo sem restrições

---
## 7. Exploração

### 7.1 Acesso inicial via SSH
```bash
ssh pi@10.10.10.48
```

# Senha: raspberry

**Resultado**:
- Acesso concedido com aviso: "SSH is enabled and the default password for the 'pi' user has not been changed"
- Verificação de privilégios:
```bash
sudo id
# uid=0(root) gid=0(root) groups=0(root)
```

![NON](file-20250703224927.png)

**Flags**:
- User flag: `/home/pi/Desktop/user.txt`
- Root flag: Mensagem indicando backup em pendrive (ver seção 8)

---
## 8. Escalada de Privilégios

**Análise de partições**:
```bash
df -h
```
![NON](file-20250703225136.png)

**Resultado**:

- Pendrive montado em `/media/usbstick`
- Conteúdo:
    - `dammit.txt`: Nota sobre arquivos deletados acidentalmente

**Recuperação da flag root**:
### Método 1: Strings no dispositivo
```bash
sudo strings /dev/sdb
```

**Saída**:
```bash
3d3e483143ff12ec505d026fa13e020b  # Flag root
```
![NON](file-20250703225413.png)

### Método 2: Análise forense (método intencional)

- Criar imagem do pendrive:
```bash
sudo dcfldd if=/dev/sdb of=/home/pi/usb.dd
```
- Transferir para máquina atacante:
```bash
scp pi@10.10.10.48:/home/pi/usb.dd .
```
- Análise com ferramentas forenses (testdisk, hex editor, strings)
![NON](file-20250703225427.png)

---
## 9. Pós-exploração
**Artefatos encontrados**:
- Configurações do Pi-hole em `/etc/pihole/`
- Serviços expostos desnecessariamente (Plex, UPnP)
- Pendrive com evidências de arquivos deletados

**Análise forense**:
- Arquivo `root.txt` deletado mas recuperável via análise de strings
- Dados residuais no dispositivo de armazenamento

---
## 10. Lições Aprendidas
> - **IoT é crítico**: Credenciais padrão em dispositivos IoT são vetores comuns de ataque
> - **Forensics**: Dados deletados podem permanecer acessíveis em dispositivos de armazenamento
> - **Privilégios**: Contas com sudo irrestrito são equivalentes a acesso root
> - **Mirai**: Demonstração real de como botnets comprometem dispositivos


---
## 11. Mitigações
1. **Credenciais**:
    - Alterar todas as senhas padrão imediatamente
    - Implementar autenticação por chaves SSH
2. **Serviços**:
    - Desativar serviços desnecessários (UPnP, Plex)
    - Atualizar todos os softwares (Pi-hole, lighttpd, dnsmasq)
3. **Controle de acesso**:
    - Restringir privilégios sudo
    - Implementar firewall para limitar portas expostas
4. **Monitoramento**:
    - Logar tentativas de acesso SSH
    - Monitorar partições não usuais

---

### Contexto Adicional (Mirai Botnet)
**Características observadas**:
- Vetor de ataque idêntico ao malware Mirai real
- Exploração de:
    
    - Credenciais padrão em IoT
    - Serviços expostos desnecessariamente
    - Permissões excessivas

**Arquitetura típica**:

1. Scan automático de dispositivos
2. Tentativa de login com lista de credenciais padrão
3. Infecção e incorporação à botnet
4. Uso em ataques DDoS

**Relevância**: Caso realístico de dispositivo IoT comprometido, demonstrando a importância de configurações seguras mesmo em ambientes domésticos.
