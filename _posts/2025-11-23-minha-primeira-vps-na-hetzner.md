---
layout: post
title: "Minha primeira VPS na Hetzner — segurança, setup e diagnóstico"
date: 2025-11-23 22:36:00 -0300
categories: [DevOps, Infraestrutura]
tags: [vps, hetzner, linux, security]
---

*Foco: história real + passo a passo técnico + cultura de aprendizado.*

---

## 1. Por que resolvi criar uma VPS pessoal

Resolvi finalmente contratar uma VPS para aprender mais a fundo sobre deploy, infraestrutura, bancos e microprojetos. A ideia não é criar nada grande no momento, mas sim ter um ambiente real para experimentos, estudo e hospedagem de pequenos sistemas.

Depois de avaliar algumas opções (RackNerd, OVH e outras), escolhi a **[Hetzner](https://www.hetzner.com){:target="_blank"}**. O modelo permite crescer aos poucos, é transparente nos preços e tem boa reputação técnica.

---

## 2. Escolha do plano e configuração inicial

Optei pelo plano **CX23**, datacenter **NBG1 (Nuremberg)**.
Usei IPv6 com IPv4 habilitado, pois quero deixar a porta aberta para trabalhar com Cloudflare no futuro.

Durante a criação da VPS, fiz duas escolhas importantes:

1.  Gerei e usei minha chave SSH (sem senha por e-mail).
2.  Habilitei 2FA na conta da [Hetzner](https://www.hetzner.com){:target="_blank"}.

Dessa forma, a máquina já nasceu com um nível de segurança adequado para estudo e deploy.

---

## 3. Primeira conexão via SSH

Após a criação da VPS, conectei direto via SSH:

```bash
ssh root@IP_DA_VPS
```

Como a chave SSH já estava registrada, a conexão funcionou sem uso de senha. A partir daí, iniciei a fase de segurança básica.

---

## 4. Fase 1 — Estrutura mínima de segurança

**Objetivo:** eliminar login root via SSH e trabalhar com um usuário próprio.

### 4.1 Atualização do sistema

```bash
apt update && apt upgrade -y && apt autoremove -y
```

### 4.2 Instalação de ferramentas básicas

```bash
apt install htop ncdu glances -y
```

### 4.3 Criação do usuário principal

```bash
adduser me
usermod -aG sudo me
```

### 4.4 Transferência da chave SSH para o usuário me

```bash
rsync --archive --chown=me:me ~/.ssh /home/me
```

### 4.5 Firewall

```bash
apt install ufw -y
ufw allow OpenSSH
ufw enable
```

---

## 5. Teste de login e desativação do root

Em um novo terminal:

```bash
ssh me@IP_DA_VPS
```

Funcionou. A partir disso, desativei o acesso SSH do root e passei a entrar apenas com o usuário `me`.

Resultado ao tentar login como root:

```text
root@IP: Permission denied (publickey).
```

Estágio de segurança concluído.

---

## 6. Diagnóstico da VPS — sistema recém-criado

### Identidade e tempo ligado

```bash
hostname
uptime
```

**Resultado:**

```text
ubuntu-4gb-nbg1-3
18:14:16 up 6 min, 1 user, load average: 0.00, 0.00, 0.00
```

---

### Uso de recursos — htop / glances

Os prints mostram:

*   CPU em idle (~97%)
*   RAM baixa (~400 MB usados)
*   Sem swap (normal nesse momento)
*   Latência extremamente baixa para Google (~4ms)

---

### Disco

```bash
df -h
```

**Resultado:**

```text
/dev/sda1  38G total  / 2.2G usados
```

Boot limpo e 34 GB livres para projetos.

---

### Memória e swap

```bash
free -m
```

*   Memória total: ~4GB
*   Usado: 402MB
*   Swap: 0

---

### Latência

```bash
ping -c 5 google.com
```

Ping médio de 4ms. Rede muito boa.

---

## 7. Status final da Fase 1

A VPS está pronta para seguir evoluindo.

**Estado atual:**

*   Acesso root desativado ✔
*   SSH seguro por chave ✔
*   Firewall ativo ✔
*   Sistema atualizado ✔
*   Diagnóstico limpo ✔

Agora o próximo passo será instalar Docker, Coolify e Caddy. Mas essa parte ficará para o próximo capítulo.

---

## 8. Sobre aprendizado

A intenção não é apenas “subir um servidor”. Estou tentando aprender infraestrutura de verdade, com maturidade técnica e aos poucos. É diferente de usar scripts prontos. Esta etapa foi essencial: entender o que realmente está acontecendo por trás da linha de comando e ganhar confiança para avançar.

---

## 9. O que virá depois

*   **Fase 2** — Docker, Coolify e Caddy.
*   **Fase 3** — micro SaaS, automações e bancos.
*   **Fase 4** — monitoramento, backup e prática em produção.

Mas, por enquanto, a Fase 1 está oficialmente encerrada.
